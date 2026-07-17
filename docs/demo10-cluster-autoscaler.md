# Demo10 — Cluster Autoscaler 节点自动伸缩

## 实验简介

本实验部署传统的 Cluster Autoscaler（CA），它通过调整 EKS 托管节点组背后的 Auto Scaling Group（ASG）容量来实现节点伸缩。与 Demo09 的 Karpenter 不同，CA 使用 IRSA（IAM Roles for Service Accounts）授权，需要手动配置 OIDC Provider，且中国区 OIDC 域名后缀为 `.amazonaws.com.cn`。

**实验目标：**
- 掌握中国区 IRSA 的完整配置（OIDC Provider + 信任策略）
- 理解 Cluster Autoscaler 基于 ASG 容量调整节点的机制
- 能够触发 CA 自动扩容节点并对比其与 Karpenter 的差异

**实验流程：**
1. 设置环境变量
2. 扩大 ASG 的最大节点数上限
3. 创建 Cluster Autoscaler IAM Policy
4. 配置 IRSA（OIDC Provider 和 IAM Role）
5. 部署 Cluster Autoscaler
6. 验证 Cluster Autoscaler 日志
7. 触发 Scale Out

**预计 AI 执行时长：** 20-30 分钟

---

## 前提条件

- **工具**：AWS CLI v2、kubectl
- **权限**：EKS、EC2、AutoScaling、IAM 权限
- **前提**：Demo01 已完成
- **说明**：Cluster Autoscaler 使用 IRSA（IAM Roles for Service Accounts）而非 Pod Identity

---

## 步骤

### 1. 设置环境变量

复用集群环境，定义 CA 的 Role、Policy、命名空间与 ServiceAccount 名称。

```bash
source /tmp/demo-eks.env
export CA_ROLE=eks-cluster-autoscaler-role
export CA_POLICY=k8s-asg-policy
export CA_NS=kube-system
export CA_SA=cluster-autoscaler
export ASSET_DIR=${ASSET_DIR:-$(pwd)/offline-assets}
```

### 2. 扩大 ASG 的最大节点数上限

CA 只能在 ASG 的 min/max 范围内调整容量。先通过定位集群的 ASG 并把 max-size 提到 6，否则扩容会因触顶而无法创建新节点。

```bash
ASG_NAME=$(aws autoscaling describe-auto-scaling-groups \
  --query "AutoScalingGroups[?contains(Tags[?Key=='eks:cluster-name'].Value, '${CLUSTER_NAME}')].AutoScalingGroupName" \
  --output text | head -1)

aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name ${ASG_NAME} \
  --max-size 6

echo "ASG ${ASG_NAME} 最大节点数已设为 6"
```

**预期输出**：打印 ASG 名称，更新成功

### 3. 创建 Cluster Autoscaler IAM Policy

CA 需要描述 ASG/EC2 以及设置期望容量、终止实例的权限。这些 autoscaling/ec2 动作作用于 `*` 资源，是 CA 调整节点组规模的最小权限集。

```bash
cat > /tmp/ca-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeAutoScalingInstances",
        "autoscaling:DescribeLaunchConfigurations",
        "autoscaling:DescribeScalingActivities",
        "autoscaling:DescribeTags",
        "ec2:DescribeImages",
        "ec2:DescribeInstanceTypes",
        "ec2:DescribeLaunchTemplateVersions",
        "ec2:GetInstanceTypesFromInstanceRequirements",
        "eks:DescribeNodegroup"
      ],
      "Resource": ["*"]
    },
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:SetDesiredCapacity",
        "autoscaling:TerminateInstanceInAutoScalingGroup"
      ],
      "Resource": ["*"]
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name ${CA_POLICY} \
  --policy-document file:///tmp/ca-policy.json || true

export CA_POLICY_ARN=$(aws iam list-policies \
  --query "Policies[?PolicyName=='${CA_POLICY}'].Arn" \
  --output text)

echo "CA Policy ARN: ${CA_POLICY_ARN}"
```

**预期输出**：Policy ARN 输出

### 4. 配置 IRSA — OIDC Provider 和 IAM Role

IRSA 需要先把集群 OIDC Issuer 注册为 IAM OIDC Provider，再创建信任该 Provider 的 Role。中国区 OIDC 域名带 `.amazonaws.com.cn`，且需用 openssl 抓取证书指纹；信任策略的 `sub` 必须精确匹配 `system:serviceaccount:kube-system:cluster-autoscaler`。

```bash
# 获取 OIDC Issuer
OIDC_URL=$(aws eks describe-cluster \
  --name ${CLUSTER_NAME} \
  --query 'cluster.identity.oidc.issuer' --output text)
OIDC_ID=$(echo ${OIDC_URL} | awk -F'/' '{print $NF}')

# 创建 OIDC Provider（如果不存在）
aws iam create-open-id-connect-provider \
  --url ${OIDC_URL} \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list "$(openssl s_client -connect oidc.eks.${AWS_REGION}.amazonaws.com.cn:443 \
    -servername oidc.eks.${AWS_REGION}.amazonaws.com.cn \
    -showcerts </dev/null 2>/dev/null | \
    openssl x509 -fingerprint -sha1 -noout | \
    sed 's/://g' | awk -F= '{print tolower($2)}')" \
  --region ${AWS_REGION} 2>/dev/null || echo "OIDC Provider 已存在"

# 创建 Trust Policy
cat > /tmp/ca-trust.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:${AWS_PARTITION}:iam::${ACCOUNT_ID}:oidc-provider/oidc.eks.${AWS_REGION}.amazonaws.com.cn/id/${OIDC_ID}"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "oidc.eks.${AWS_REGION}.amazonaws.com.cn/id/${OIDC_ID}:sub": "system:serviceaccount:${CA_NS}:${CA_SA}",
        "oidc.eks.${AWS_REGION}.amazonaws.com.cn/id/${OIDC_ID}:aud": "sts.amazonaws.com"
      }
    }
  }]
}
EOF

aws iam create-role \
  --role-name ${CA_ROLE} \
  --assume-role-policy-document file:///tmp/ca-trust.json || true

aws iam attach-role-policy \
  --role-name ${CA_ROLE} \
  --policy-arn ${CA_POLICY_ARN}

export CA_ROLE_ARN=$(aws iam get-role \
  --role-name ${CA_ROLE} \
  --query 'Role.Arn' --output text)

echo "CA Role ARN: ${CA_ROLE_ARN}"
```

**预期输出**：CA Role ARN 输出

### 5. 部署 Cluster Autoscaler

CA 镜像已集中预置在 0910 仓库（`registryk8sio/autoscaling/cluster-autoscaler:v1.35.0`），无需推送。下载 YAML 后用 sed 把镜像地址替换为 0910、集群名替换为实际值，部署后再给 ServiceAccount 打 IRSA role-arn 注解，使 CA Pod 获得 ASG 操作权限。

```bash
cp ${ASSET_DIR}/cluster-autoscaler-autodiscover.yaml /tmp/ca.yaml

# 将 Cluster Autoscaler 镜像地址替换为 0910，并替换集群名
sed -i -E "s|image: .*cluster-autoscaler:v1.35.0|image: 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/registryk8sio/autoscaling/cluster-autoscaler:v1.35.0|g" /tmp/ca.yaml
sed -i "s|<YOUR CLUSTER NAME>|${CLUSTER_NAME}|g" /tmp/ca.yaml
# 修正自动发现 tag（autodiscover yaml 默认用 eksworkshop，需改为实际集群名）
sed -i "s|k8s.io/cluster-autoscaler/eksworkshop|k8s.io/cluster-autoscaler/${CLUSTER_NAME}|g" /tmp/ca.yaml

# 应用 IRSA 注解
kubectl apply -f /tmp/ca.yaml

kubectl annotate serviceaccount ${CA_SA} \
  -n ${CA_NS} \
  eks.amazonaws.com/role-arn=${CA_ROLE_ARN} \
  --overwrite

# 注入 AWS_REGION 环境变量（CA 不支持 --aws-region 命令行参数）
kubectl set env deployment/cluster-autoscaler \
  -n ${CA_NS} \
  AWS_REGION=${AWS_REGION} \
  AWS_DEFAULT_REGION=${AWS_REGION}

kubectl -n ${CA_NS} rollout status deployment/cluster-autoscaler
```

> ⚠️ **中国区 CA 配置注意事项**：`cluster-autoscaler-autodiscover.yaml` 中的 ASG 自动发现 tag 为 `k8s.io/cluster-autoscaler/eksworkshop`，需替换为实际集群名 `demo`，否则 CA 找不到任何 ASG。CA v1.35.0 不支持 `--aws-region` 命令行参数，须通过 `kubectl set env` 注入 `AWS_REGION` 环境变量。


> ⚠️ **首次拉取 048912060910 镜像若出现 ImagePullBackOff（403）**：这是节点 ECR credential provider 冷启动的短暂问题，无需修改任何权限配置。等待约 30 秒后执行 `kubectl rollout restart deployment/<name> -n <ns>` 即可恢复。节点一旦缓存镜像后，同节点后续 Pod 直接走本地缓存不再触发。

**预期输出**：`deployment "cluster-autoscaler" successfully rolled out`

### 6. 验证 Cluster Autoscaler 日志

查看 CA 日志确认它已正常启动并识别到目标集群。日志中应能看到版本号和集群名，说明 IRSA 授权与 ASG 自动发现配置正确。

```bash
kubectl -n ${CA_NS} logs deployment/cluster-autoscaler --tail=20 | grep -E "Starting|version|cluster"
```

**预期输出**：日志中显示 Cluster Autoscaler 启动信息和集群名称

### 7. 触发 Scale Out

部署一个 0 副本的 inflate 工作负载后扩到 8 副本，制造大量待调度 Pod。CA 检测到 Pending Pod 后调高 ASG 期望容量，几分钟内新节点上线并承载这些 Pod。

```bash
cat > /tmp/ca-inflate.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ca-inflate
  namespace: default
spec:
  replicas: 0
  selector:
    matchLabels:
      app: ca-inflate
  template:
    metadata:
      labels:
        app: ca-inflate
    spec:
      containers:
        - name: inflate
          image: 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/busybox:1.36
          command: ["sleep", "infinity"]
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
EOF

kubectl apply -f /tmp/ca-inflate.yaml
kubectl scale deployment ca-inflate --replicas=8

echo "等待 CA 触发节点扩展（约 3-5 分钟）..."
for i in $(seq 1 15); do
  echo "=== $(date -u +%H:%M:%S) ==="
  kubectl get nodes
  kubectl get pods -l app=ca-inflate | tail -5
  sleep 30
done
```

**预期输出**：节点数量增加，Pod 全部变为 Running

---

## 验收标准

完成本实验后，你应当能够：
- [ ] Cluster Autoscaler 在 kube-system 中正常运行（readyReplicas 为 1）
- [ ] ServiceAccount 上已正确配置 IRSA role-arn 注解
- [ ] 集群 ASG 的 MaxSize 已调整为 6
- [ ] 扩容 inflate 工作负载后观察到节点数量增加且 Pod 全部 Running

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `kubectl get deployment cluster-autoscaler -n kube-system -o jsonpath='{.status.readyReplicas}'` | `1` |
| 2 | `kubectl get serviceaccount cluster-autoscaler -n kube-system -o jsonpath='{.metadata.annotations.eks\.amazonaws\.com/role-arn}' \| grep -c cluster-autoscaler-role` | `1` |
| 3 | `aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names $(aws autoscaling describe-auto-scaling-groups --query "AutoScalingGroups[?contains(Tags[?Key==\`eks:cluster-name\`].Value, \`demo\`)].AutoScalingGroupName" --output text --region cn-northwest-1) --region cn-northwest-1 --query 'AutoScalingGroups[0].MaxSize' --output text` | `6` |

---

## 实验总结

本实验用 Cluster Autoscaler 通过调整 ASG 容量实现了节点伸缩，并完整走通了中国区 IRSA 的 OIDC Provider 与信任策略配置。对比 Demo09 的 Karpenter，CA 受限于预定义的节点组与实例类型、伸缩较慢，而 Karpenter 更灵活高效——这帮助你在不同场景选择合适的节点伸缩方案。下一个 Demo（Demo11）将进入 CSI 有状态存储（EBS/EFS/S3）。

---

## 清理

```bash
source /tmp/demo-eks.env

kubectl scale deployment ca-inflate --replicas=0
kubectl delete -f /tmp/ca-inflate.yaml

kubectl delete -f /tmp/ca.yaml || true

aws iam detach-role-policy \
  --role-name ${CA_ROLE} \
  --policy-arn ${CA_POLICY_ARN}
aws iam delete-role --role-name ${CA_ROLE}
aws iam delete-policy --policy-arn ${CA_POLICY_ARN}
```
