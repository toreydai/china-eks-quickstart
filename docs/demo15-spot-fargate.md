# Demo15 — Spot 节点与 Fargate Profile

## 实验简介

本实验演练 EKS 的两种弹性计算形态：Spot 托管节点组（用折扣实例承载容错型负载）和 Fargate Profile（无服务器容器，无需管理节点）。通过把应用分别部署到 Spot 节点和 Fargate，并对比两者差异，理解不同场景下的成本与运维权衡。

**实验目标：**
- 掌握 Spot 托管节点组与 Fargate Profile 的创建
- 理解 nodeSelector/toleration 将负载定向到 Spot 的方式
- 能够对比 EC2 节点与 Fargate 在管理与计费上的差异

**实验流程：**
1. 设置环境变量
2. 添加 Spot 托管节点组
3. 部署应用到 Spot 节点
4. 创建 Fargate Profile
5. 部署应用到 Fargate
6. 对比 EC2 节点与 Fargate 节点的区别

**预计时长：** 20-30 分钟

## 前提条件

- **工具**：AWS CLI v2、eksctl、kubectl
- **权限**：EKS、EC2、IAM 权限
- **前提**：Demo01 已完成
- **预计耗时**：20-30 分钟

---

## 步骤

### 1. 设置环境变量

复用集群环境，定义 Spot 节点组、Fargate Profile 与 Fargate 命名空间名称。

```bash
source /tmp/demo-eks.env
export SPOT_NG=ng-spot
export FARGATE_PROFILE=fp-demo
export FARGATE_NS=fargate-demo
```

### 2. 添加 Spot 托管节点组

用 eksctl 创建多实例类型的 Spot 节点组（minSize=0），分散实例类型可降低中断风险并节省成本。

```bash
cat > /tmp/ng-spot.yaml <<EOF
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: ${CLUSTER_NAME}
  region: ${AWS_REGION}
managedNodeGroups:
  - name: ${SPOT_NG}
    spot: true
    instanceTypes:
      - t3.medium
      - t3.large
      - t3a.medium
      - t3a.large
    minSize: 0
    desiredCapacity: 2
    maxSize: 5
    labels:
      lifecycle: spot
      node-type: spot
    tags:
      Project: eks-china-quickstart
      NodeType: spot
EOF

eksctl create nodegroup -f /tmp/ng-spot.yaml

kubectl get nodes -l lifecycle=spot
```

**预期输出**：2 个 Spot 节点加入集群，状态为 `Ready`，标签 `lifecycle=spot`

### 3. 部署应用到 Spot 节点

用 nodeSelector + toleration 把应用定向调度到 Spot 节点，验证工作负载能在 Spot 容量上运行。

```bash
cat > /tmp/spot-app.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spot-app
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: spot-app
  template:
    metadata:
      labels:
        app: spot-app
    spec:
      nodeSelector:
        lifecycle: spot
      tolerations:
        - key: "eks.amazonaws.com/capacityType"
          value: "SPOT"
          effect: "NoSchedule"
      containers:
        - name: app
          image: 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/nginx:1.27-alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "128Mi"
EOF

kubectl apply -f /tmp/spot-app.yaml
kubectl rollout status deployment/spot-app
kubectl get pods -l app=spot-app -o wide
```


> ⚠️ **首次拉取 048912060910 镜像若出现 ImagePullBackOff（403）**：这是节点 ECR credential provider 冷启动的短暂问题，无需修改任何权限配置。等待约 30 秒后执行 `kubectl rollout restart deployment/<name> -n <ns>` 即可恢复。节点一旦缓存镜像后，同节点后续 Pod 直接走本地缓存不再触发。

**预期输出**：3 个 Pod 调度到 Spot 节点上运行

### 4. 创建 Fargate Profile

创建 Fargate 执行角色并基于命名空间选择器创建 Profile（需私有子网），命中该命名空间的 Pod 将自动运行在 Fargate。

```bash
kubectl create namespace ${FARGATE_NS} --dry-run=client -o yaml | kubectl apply -f -

# 创建 Fargate 执行角色（如果不存在）
cat > /tmp/fargate-trust.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "eks-fargate-pods.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name eks-fargate-execution-role \
  --assume-role-policy-document file:///tmp/fargate-trust.json || true

aws iam attach-role-policy \
  --role-name eks-fargate-execution-role \
  --policy-arn arn:${AWS_PARTITION}:iam::aws:policy/AmazonEKSFargatePodExecutionRolePolicy

FARGATE_ROLE_ARN=$(aws iam get-role \
  --role-name eks-fargate-execution-role \
  --query 'Role.Arn' --output text)

# 获取私有子网（Fargate 需要私有子网）
CLUSTER_VPC=$(aws eks describe-cluster \
  --name ${CLUSTER_NAME} \
  --query 'cluster.resourcesVpcConfig.vpcId' --output text)

PRIVATE_SUBNETS=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=${CLUSTER_VPC}" \
             "Name=map-public-ip-on-launch,Values=false" \
  --query 'Subnets[*].SubnetId' --output text | tr '\t' ',')

if [ -z "${PRIVATE_SUBNETS}" ]; then
  echo "未找到私有子网，使用公有子网（仅用于演示，生产环境建议私有子网）"
  PRIVATE_SUBNETS=$(aws ec2 describe-subnets \
    --filters "Name=vpc-id,Values=${CLUSTER_VPC}" \
    --query 'Subnets[0].SubnetId' --output text)
fi

set +e
aws eks create-fargate-profile \
  --cluster-name ${CLUSTER_NAME} \
  --fargate-profile-name ${FARGATE_PROFILE} \
  --pod-execution-role-arn ${FARGATE_ROLE_ARN} \
  --subnets $(echo ${PRIVATE_SUBNETS} | tr ',' ' ') \
  --selectors namespace=${FARGATE_NS} \
  --region ${AWS_REGION}
CREATE_FP_RC=$?
set -e

if [ ${CREATE_FP_RC} -ne 0 ]; then
  echo "Fargate Profile 创建失败，通常是账号配额、区域能力或子网条件限制。本 Demo 后续 Fargate 步骤跳过。"
  export SKIP_FARGATE=1
else
  export SKIP_FARGATE=0
fi

if [ "${SKIP_FARGATE}" = "0" ]; then
  echo "等待 Fargate Profile 创建（约 3-5 分钟）..."
  aws eks wait fargate-profile-active \
    --cluster-name ${CLUSTER_NAME} \
    --fargate-profile-name ${FARGATE_PROFILE} \
    --region ${AWS_REGION}

  echo "Fargate Profile 已激活"
fi
```

**预期输出**：打印"Fargate Profile 已激活"（约 3-5 分钟）

### 5. 部署应用到 Fargate

在 Fargate 命名空间部署应用，每个 Pod 会获得独立的 Fargate 微型节点（名称以 fargate-ip- 开头）。

```bash
if [ "${SKIP_FARGATE:-0}" = "1" ]; then
  echo "已跳过 Fargate 部分"
else
cat > /tmp/fargate-app.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fargate-app
  namespace: fargate-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: fargate-app
  template:
    metadata:
      labels:
        app: fargate-app
    spec:
      containers:
        - name: app
          image: 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/nginx:1.27-alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "500m"
              memory: "1Gi"
EOF

kubectl apply -f /tmp/fargate-app.yaml
kubectl -n ${FARGATE_NS} rollout status deployment/fargate-app
kubectl get nodes | grep fargate
kubectl get pods -n ${FARGATE_NS} -o wide
fi
```

**预期输出**：若 Fargate Profile 创建成功，Fargate 节点（名称以 `fargate-ip-` 开头）加入集群，2 个 Pod 运行在 Fargate 节点；若账号或子网限制导致创建失败，本部分会明确跳过。

### 6. 对比 EC2 节点与 Fargate 节点的区别

对比普通/Spot EC2 节点与 Fargate 节点——Fargate 无需管理节点、按 Pod 申请的 vCPU/内存计费。

```bash
if [ "${SKIP_FARGATE:-0}" = "1" ]; then
  echo "Fargate 部分已跳过，仅展示 EC2/Spot 节点"
fi

echo "=== EC2 节点 ==="
kubectl get nodes -l eks.amazonaws.com/compute-type!=fargate -o wide 2>/dev/null || \
  kubectl get nodes -l lifecycle=spot -o wide

echo ""
echo "=== Fargate 节点 ==="
kubectl get nodes | grep fargate || echo "无 Fargate 节点"

echo ""
echo "=== Fargate Pod 资源限制（按 vCPU/内存组合计费）==="
kubectl get pod -n ${FARGATE_NS} -o jsonpath='{range .items[*]}{.metadata.name}{" CPU:"}{.spec.containers[0].resources.requests.cpu}{" MEM:"}{.spec.containers[0].resources.requests.memory}{"\n"}{end}'
```

**预期输出**：显示 EC2 节点（普通/Spot）和 Fargate 节点的对比

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 成功创建多实例类型的 Spot 托管节点组并使其 Ready
- [ ] 将应用通过 nodeSelector/toleration 调度到 Spot 节点
- [ ] 如账号和子网条件允许，创建状态为 ACTIVE 的 Fargate Profile 并使 Pod 运行在 Fargate 节点；否则明确跳过 Fargate 部分
- [ ] 能说明 EC2 节点与 Fargate 节点在管理和计费上的区别

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws eks describe-fargate-profile --cluster-name demo --fargate-profile-name fp-demo --region cn-northwest-1 --query 'fargateProfile.status' --output text 2>/dev/null || echo SKIPPED` | `ACTIVE` 或 `SKIPPED` |
| 2 | `kubectl get nodes --show-labels \| grep lifecycle=spot \| wc -l` | `2` |
| 3 | `kubectl get pods -n fargate-demo -o jsonpath='{.items[0].spec.nodeName}' 2>/dev/null \| grep fargate || echo SKIPPED` | 包含 `fargate` 或 `SKIPPED` |
| 4 | `kubectl get deployment spot-app -o jsonpath='{.status.readyReplicas}'` | `3` |

---

## 实验总结

本实验实践了 EKS 的两种弹性计算形态：Spot 节点以折扣价承载容错型负载，Fargate 以无服务器方式免去节点管理。你掌握了用调度约束定向 Spot、用 Profile 选择器把命名空间交给 Fargate，并对比了两者的成本与运维取舍。下一个 Demo（Demo16）将进入 NetworkPolicy 的 Pod 流量隔离。

## 清理

```bash
source /tmp/demo-eks.env

kubectl delete -f /tmp/spot-app.yaml
kubectl delete -f /tmp/fargate-app.yaml
kubectl delete namespace ${FARGATE_NS}

aws eks delete-fargate-profile \
  --cluster-name ${CLUSTER_NAME} \
  --fargate-profile-name ${FARGATE_PROFILE} \
  --region ${AWS_REGION}

aws eks wait fargate-profile-deleted \
  --cluster-name ${CLUSTER_NAME} \
  --fargate-profile-name ${FARGATE_PROFILE} \
  --region ${AWS_REGION}

eksctl delete nodegroup \
  --cluster ${CLUSTER_NAME} \
  --name ${SPOT_NG} \
  --region ${AWS_REGION}

aws iam detach-role-policy \
  --role-name eks-fargate-execution-role \
  --policy-arn arn:${AWS_PARTITION}:iam::aws:policy/AmazonEKSFargatePodExecutionRolePolicy || true
aws iam delete-role --role-name eks-fargate-execution-role || true
```
