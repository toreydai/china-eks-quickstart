# Demo09 — Karpenter 节点自动伸缩

## 实验简介

本实验在中国区 EKS 上部署 Karpenter，实现基于待调度 Pod 需求的快速节点自动伸缩。相比传统 Cluster Autoscaler，Karpenter 直接调用 EC2 API 按需选择实例类型，更快更省。本实验涵盖 CloudFormation 基础设施、子网/安全组发现标签、控制器 Pod Identity 授权、NodePool/EC2NodeClass 声明以及扩容验证，全程使用 `aws-cn` 分区与私有 ECR。

**实验目标：**
- 掌握 Karpenter 在中国区的完整安装链路（CFN、IAM、Access Entry、Helm）
- 理解 NodePool 与 EC2NodeClass 声明式定义节点供给的机制
- 能够通过工作负载扩容触发 Karpenter 自动创建新节点

**实验流程：**
1. 设置环境变量
2. 部署 Karpenter CloudFormation 基础设施
3. 为节点子网和安全组打发现标签
4. 配置控制器 IAM Role 和 Pod Identity
5. 将节点 Role 加入 EKS Access Entry
6. 用 Helm 安装 Karpenter
7. 创建 NodePool 和 EC2NodeClass
8. 测试节点自动伸缩

**预计时长：** 30-45 分钟

---

## 前提条件

- **工具**：AWS CLI v2、kubectl、helm
- **权限**：EKS、EC2、IAM、CloudFormation、SQS 权限
- **前提**：Demo01 已完成，Pod Identity Agent addon 已安装
- **预计耗时**：30-45 分钟

---

## 步骤

### 1. 设置环境变量

复用集群环境，定义 Karpenter 版本、命名空间以及节点 Role 和控制器 Role 的名称。

```bash
source /tmp/demo-eks.env
export KARPENTER_VERSION=1.12.1
export KARPENTER_NS=kube-system
export KARPENTER_ROLE=KarpenterNodeRole-${CLUSTER_NAME}
export KARPENTER_CONTROLLER_ROLE=KarpenterControllerRole-${CLUSTER_NAME}
export ASSET_DIR=${ASSET_DIR:-$(pwd)/offline-assets}
```

### 2. 部署 Karpenter CloudFormation 基础设施

CFN 模板创建节点 Role、控制器所需的 5 个 IAM Policy 以及中断处理 SQS 队列。通过 `Partition=${AWS_PARTITION}` 参数适配中国区 `aws-cn` 分区。

```bash
cp ${ASSET_DIR}/karpenter-${KARPENTER_VERSION}-cloudformation.yaml /tmp/karpenter-cfn.yaml

aws cloudformation deploy \
  --stack-name karpenter-${CLUSTER_NAME} \
  --template-file /tmp/karpenter-cfn.yaml \
  --parameter-overrides \
    ClusterName=${CLUSTER_NAME} \
    AccountId=${ACCOUNT_ID} \
    Partition=${AWS_PARTITION} \
  --capabilities CAPABILITY_NAMED_IAM \
  --region ${AWS_REGION}

echo "CloudFormation 部署完成"
```

**预期输出**：Stack 部署成功

### 3. 为节点子网和安全组打标签

Karpenter 通过 `karpenter.sh/discovery` 标签自动发现可用的子网和安全组。必须为集群 VPC 的所有子网及集群安全组打上该标签，否则 EC2NodeClass 找不到资源无法创建节点。

```bash
CLUSTER_VPC=$(aws eks describe-cluster \
  --name ${CLUSTER_NAME} \
  --query 'cluster.resourcesVpcConfig.vpcId' --output text)

# 标记子网
SUBNET_IDS=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=${CLUSTER_VPC}" \
  --query 'Subnets[*].SubnetId' --output text)

for SUBNET_ID in ${SUBNET_IDS}; do
  aws ec2 create-tags \
    --resources ${SUBNET_ID} \
    --tags "Key=karpenter.sh/discovery,Value=${CLUSTER_NAME}"
done

# 标记集群安全组
SG_ID=$(aws eks describe-cluster \
  --name ${CLUSTER_NAME} \
  --query 'cluster.resourcesVpcConfig.clusterSecurityGroupId' --output text)

aws ec2 create-tags \
  --resources ${SG_ID} \
  --tags "Key=karpenter.sh/discovery,Value=${CLUSTER_NAME}"

echo "子网和安全组标签已添加"
```

**预期输出**：打印"子网和安全组标签已添加"

### 4. 配置 Karpenter 控制器 IAM Role 和 Pod Identity

手动创建控制器 Role（信任 `pods.eks.amazonaws.com`），附加 CFN 输出的 5 个专用 Policy，再建立 Pod Identity Association 让 karpenter ServiceAccount 获得权限。中国区无法用 eksctl 直接关联托管 Policy，故手动 attach。

```bash
# 获取 CloudFormation 输出的 Policy ARNs
CONTROLLER_POLICY_ARNS=$(aws cloudformation describe-stacks \
  --stack-name karpenter-${CLUSTER_NAME} \
  --query 'Stacks[0].Outputs[?OutputKey==`KarpenterControllerPolicyArns`].OutputValue' \
  --output text)

cat > /tmp/karpenter-controller-trust.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "pods.eks.amazonaws.com"},
    "Action": ["sts:AssumeRole","sts:TagSession"]
  }]
}
EOF

aws iam create-role \
  --role-name ${KARPENTER_CONTROLLER_ROLE} \
  --assume-role-policy-document file:///tmp/karpenter-controller-trust.json || true

# 附加 5 个专用 policy
for POLICY_ARN in \
  arn:${AWS_PARTITION}:iam::${ACCOUNT_ID}:policy/KarpenterControllerEKSIntegrationPolicy \
  arn:${AWS_PARTITION}:iam::${ACCOUNT_ID}:policy/KarpenterControllerIAMIntegrationPolicy \
  arn:${AWS_PARTITION}:iam::${ACCOUNT_ID}:policy/KarpenterControllerInterruptionPolicy \
  arn:${AWS_PARTITION}:iam::${ACCOUNT_ID}:policy/KarpenterControllerNodeLifecyclePolicy \
  arn:${AWS_PARTITION}:iam::${ACCOUNT_ID}:policy/KarpenterControllerResourceDiscoveryPolicy; do
  aws iam attach-role-policy \
    --role-name ${KARPENTER_CONTROLLER_ROLE} \
    --policy-arn ${POLICY_ARN} || true
done

CONTROLLER_ROLE_ARN=$(aws iam get-role \
  --role-name ${KARPENTER_CONTROLLER_ROLE} \
  --query 'Role.Arn' --output text)

# Pod Identity Association
aws eks create-pod-identity-association \
  --cluster-name ${CLUSTER_NAME} \
  --namespace ${KARPENTER_NS} \
  --service-account karpenter \
  --role-arn ${CONTROLLER_ROLE_ARN} \
  --region ${AWS_REGION} || true

echo "Karpenter Controller Role 配置完成"
```

**预期输出**：打印"Karpenter Controller Role 配置完成"

### 5. 将 Karpenter 节点 Role 加入 EKS Access Entry

Karpenter 创建的节点用 KarpenterNodeRole 加入集群，必须为该 Role 创建 `EC2_LINUX` 类型的 Access Entry，否则新节点无法注册到集群（一直 NotReady）。

```bash
KARPENTER_NODE_ROLE_ARN=$(aws cloudformation describe-stacks \
  --stack-name karpenter-${CLUSTER_NAME} \
  --query 'Stacks[0].Outputs[?OutputKey==`KarpenterNodeRoleArn`].OutputValue' \
  --output text)

aws eks create-access-entry \
  --cluster-name ${CLUSTER_NAME} \
  --principal-arn ${KARPENTER_NODE_ROLE_ARN} \
  --type EC2_LINUX \
  --region ${AWS_REGION} || true

echo "节点 Role ARN: ${KARPENTER_NODE_ROLE_ARN}"
```

**预期输出**：Access Entry 创建成功

### 6. 安装 Karpenter

控制器镜像已集中预置在 0910 仓库（`ecrpublic/karpenter/controller:1.12.1`），无需再推送。Helm chart 已放在本仓库 `offline-assets/` 目录。关键参数包括集群名、中断队列名、指向 0910 的镜像地址，以及通过 `env[0]` 注入 `AWS_PARTITION=aws-cn`。

```bash
cp ${ASSET_DIR}/karpenter-${KARPENTER_VERSION}.tgz /tmp/

helm upgrade --install karpenter /tmp/karpenter-${KARPENTER_VERSION}.tgz \
  -n ${KARPENTER_NS} \
  --set settings.clusterName=${CLUSTER_NAME} \
  --set settings.interruptionQueue=karpenter-${CLUSTER_NAME} \
  --set controller.image.repository=048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/karpenter/controller \
  --set controller.image.tag=${KARPENTER_VERSION} \
  --set controller.image.digest="" \
  --set serviceAccount.create=true \
  --set serviceAccount.name=karpenter \
  --set env[0].name=AWS_PARTITION \
  --set env[0].value=${AWS_PARTITION}

kubectl -n ${KARPENTER_NS} rollout status deployment/karpenter
```


> ⚠️ **首次拉取 048912060910 镜像若出现 ImagePullBackOff（403）**：这是节点 ECR credential provider 冷启动的短暂问题，无需修改任何权限配置。等待约 30 秒后执行 `kubectl rollout restart deployment/<name> -n <ns>` 即可恢复。节点一旦缓存镜像后，同节点后续 Pod 直接走本地缓存不再触发。

**预期输出**：`deployment "karpenter" successfully rolled out`

### 7. 创建 NodePool 和 EC2NodeClass

NodePool 定义可供给的实例类型与容量类型、资源上限和整合策略；EC2NodeClass 定义 AMI、节点 Role 以及通过发现标签选择子网/安全组。两者共同声明「Karpenter 可以创建什么样的节点」。

```bash
AMI_ID=$(aws ssm get-parameter \
  --name "/aws/service/eks/optimized-ami/1.35/amazon-linux-2023/x86_64/standard/recommended/image_id" \
  --region ${AWS_REGION} \
  --query 'Parameter.Value' --output text)

cat > /tmp/karpenter-nodepool.yaml <<EOF
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand"]
        - key: node.kubernetes.io/instance-type
          operator: In
          values: ["t3.medium","t3.large","t3.xlarge"]
  limits:
    cpu: 100
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 1m
---
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2023
  role: ${KARPENTER_ROLE}
  amiSelectorTerms:
    - id: \${AMI_ID}
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: ${CLUSTER_NAME}
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: ${CLUSTER_NAME}
  tags:
    Project: eks-china-quickstart
EOF

kubectl apply -f /tmp/karpenter-nodepool.yaml
kubectl get nodepool default
kubectl get ec2nodeclass default
```

> ⚠️ **Karpenter v1.12.1 EC2NodeClass 规范变更**：`amiFamily: AL2023` 必须同时提供 `amiSelectorTerms`（用 SSM 查出的 AMI ID），否则报 `spec: Invalid value: must specify amiFamily if amiSelectorTerms does not contain an alias`。同时 Helm chart 默认携带 `controller.image.digest`，须用 `--set controller.image.digest=""` 清空，否则 kubelet 会用 sha256 摘要拉取导致 403。

**预期输出**：NodePool 和 EC2NodeClass 创建成功

### 8. 测试节点自动伸缩

部署一个初始 0 副本的 inflate 工作负载，再扩到 5 副本制造大量待调度 Pod。现有节点资源不足时，Karpenter 会在几分钟内自动创建新节点并把 Pod 调度上去。

```bash
# 部署需要大量资源的工作负载触发节点扩展
cat > /tmp/inflate.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inflate
  namespace: default
spec:
  replicas: 0
  selector:
    matchLabels:
      app: inflate
  template:
    metadata:
      labels:
        app: inflate
    spec:
      terminationGracePeriodSeconds: 0
      containers:
        - name: inflate
          image: 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/busybox:1.36
          command: ["sleep", "infinity"]
          resources:
            requests:
              cpu: "1"
              memory: "1Gi"
EOF

kubectl apply -f /tmp/inflate.yaml

# 扩展到 5 个副本，触发新节点创建
kubectl scale deployment inflate --replicas=5

echo "等待 Karpenter 创建新节点（约 2-5 分钟）..."
for i in $(seq 1 15); do
  echo "=== $(date -u +%H:%M:%S) ==="
  kubectl get nodes
  kubectl get pods -l app=inflate
  sleep 30
done
```

**预期输出**：Karpenter 自动创建新节点，inflate Pod 调度到新节点

---

## 验收标准

完成本实验后，你应当能够：
- [ ] Karpenter 控制器在 kube-system 中正常运行（readyReplicas 为 2）
- [ ] NodePool 与 EC2NodeClass 的 Ready 状态均为 True
- [ ] 扩容 inflate 工作负载后观察到 Karpenter 自动创建新节点
- [ ] 新节点成功注册到集群并承载待调度的 Pod

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `kubectl get deployment karpenter -n kube-system -o jsonpath='{.status.readyReplicas}'` | `2` |
| 2 | `kubectl get nodepool default -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}'` | `True` |
| 3 | `kubectl get ec2nodeclass default -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}'` | `True` |

---

## 实验总结

本实验在中国区部署了 Karpenter，实现了由待调度 Pod 直接驱动的快速节点供给，并验证了扩容时新节点的自动创建。你掌握了中国区特有的分区适配、发现标签、Access Entry 注册等关键点。Karpenter 是当前 EKS 推荐的节点伸缩方案；下一个 Demo（Demo10）将对比演示传统的 Cluster Autoscaler 实现节点伸缩。

---

## 清理

```bash
source /tmp/demo-eks.env

kubectl scale deployment inflate --replicas=0
kubectl delete -f /tmp/inflate.yaml

kubectl delete -f /tmp/karpenter-nodepool.yaml

helm uninstall karpenter -n ${KARPENTER_NS}

aws eks delete-pod-identity-association \
  --cluster-name ${CLUSTER_NAME} \
  --association-id $(aws eks list-pod-identity-associations \
    --cluster-name ${CLUSTER_NAME} \
    --namespace kube-system \
    --service-account karpenter \
    --query 'associations[0].associationId' --output text) \
  --region ${AWS_REGION} || true

for POLICY_ARN in \
  arn:${AWS_PARTITION}:iam::${ACCOUNT_ID}:policy/KarpenterControllerEKSIntegrationPolicy \
  arn:${AWS_PARTITION}:iam::${ACCOUNT_ID}:policy/KarpenterControllerIAMIntegrationPolicy \
  arn:${AWS_PARTITION}:iam::${ACCOUNT_ID}:policy/KarpenterControllerInterruptionPolicy \
  arn:${AWS_PARTITION}:iam::${ACCOUNT_ID}:policy/KarpenterControllerNodeLifecyclePolicy \
  arn:${AWS_PARTITION}:iam::${ACCOUNT_ID}:policy/KarpenterControllerResourceDiscoveryPolicy; do
  aws iam detach-role-policy \
    --role-name ${KARPENTER_CONTROLLER_ROLE} \
    --policy-arn ${POLICY_ARN} || true
done
aws iam delete-role --role-name ${KARPENTER_CONTROLLER_ROLE} || true

aws eks delete-access-entry \
  --cluster-name ${CLUSTER_NAME} \
  --principal-arn ${KARPENTER_NODE_ROLE_ARN} \
  --region ${AWS_REGION} || true

aws cloudformation delete-stack \
  --stack-name karpenter-${CLUSTER_NAME} \
  --region ${AWS_REGION}
```
