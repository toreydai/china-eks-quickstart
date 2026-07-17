# Demo01 — 准备实验环境与创建集群

## 实验简介

本实验从零开始在 AWS 中国区（宁夏 cn-northwest-1）搭建一个 EKS 集群，并完成后续所有 Demo 所依赖的基础环境。中国区与全球区在分区标识（`aws-cn`）、ECR 域名和工具下载方式上存在差异，本实验通过本地工具文件离线安装规避了网络限制。

**实验目标：**
- 掌握使用 eksctl 在中国区创建托管 EKS 集群
- 理解中国区特有的 `aws-cn` 分区、ECR 域名与本地工具文件离线安装方式
- 能够安装 Pod Identity Agent addon 并管理节点组规模

**实验流程：**
1. 设置中国区环境变量并持久化到文件
2. 检查并按需从本地 tools 目录安装 eksctl / kubectl / helm
3. 用 eksctl 创建 EKS 集群
4. 验证集群连接与节点状态
5. 将集群关键信息写入环境变量文件
6. 安装 EKS Pod Identity Agent addon
7. 扩展节点组验证节点管理

**预计 AI 执行时长：** 25-35 分钟

---

## 前提条件

- **工具**：AWS CLI v2、eksctl、kubectl、helm（缺失时从本地 `tools/` 目录安装）
- **权限**：EKS、EC2、IAM、CloudFormation、ECR 完整权限
- **区域**：cn-northwest-1（宁夏）

---

## 步骤

### 1. 设置环境变量

中国区使用 `aws-cn` 分区，ECR 域名后缀为 `.amazonaws.com.cn`，与全球区不同。将这些变量持久化到 `/tmp/demo-eks.env`，方便后续步骤和其它 Demo 通过 `source` 复用，避免变量丢失。

```bash
export AWS_REGION=cn-northwest-1
export AWS_DEFAULT_REGION=${AWS_REGION}
export AWS_PARTITION=aws-cn
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export CLUSTER_NAME=demo
export NODE_TYPE=t3.medium
export ECR_REGISTRY=${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com.cn

cat > /tmp/demo-eks.env <<EOF
export AWS_REGION=cn-northwest-1
export AWS_DEFAULT_REGION=cn-northwest-1
export AWS_PARTITION=aws-cn
export ACCOUNT_ID=${ACCOUNT_ID}
export CLUSTER_NAME=${CLUSTER_NAME}
export NODE_TYPE=${NODE_TYPE}
export ECR_REGISTRY=${ECR_REGISTRY}
EOF

echo "环境变量已保存到 /tmp/demo-eks.env"
```

**预期输出**：打印"环境变量已保存到 /tmp/demo-eks.env"

### 2. 检查并安装工具（从本地 tools 目录）

中国区无法稳定访问 GitHub 等上游下载地址。先检查操作机是否已有 `eksctl`、`kubectl`、`helm`；如果缺失，请先在可联网环境分别下载 Linux amd64 版本，并上传到当前操作机的本仓库 `tools/` 目录：

- `tools/eksctl`
- `tools/kubectl`
- `tools/helm.tar.gz`

如工具文件放在其它位置，可通过 `TOOL_DIR=/path/to/tools` 指定。

```bash
TOOL_DIR=${TOOL_DIR:-$(pwd)/tools}

if command -v eksctl >/dev/null 2>&1; then
  echo "eksctl 已安装"
else
  test -f ${TOOL_DIR}/eksctl || { echo "缺少 ${TOOL_DIR}/eksctl，请先下载并上传到操作机"; exit 1; }
  sudo install ${TOOL_DIR}/eksctl /usr/local/bin/eksctl
fi
eksctl version

if command -v kubectl >/dev/null 2>&1; then
  echo "kubectl 已安装"
else
  test -f ${TOOL_DIR}/kubectl || { echo "缺少 ${TOOL_DIR}/kubectl，请先下载并上传到操作机"; exit 1; }
  sudo install ${TOOL_DIR}/kubectl /usr/local/bin/kubectl
fi
kubectl version --client=true

if command -v helm >/dev/null 2>&1; then
  echo "helm 已安装"
else
  test -f ${TOOL_DIR}/helm.tar.gz || { echo "缺少 ${TOOL_DIR}/helm.tar.gz，请先下载并上传到操作机"; exit 1; }
  tar -xzf ${TOOL_DIR}/helm.tar.gz -C /tmp/
  sudo install /tmp/linux-amd64/helm /usr/local/bin/helm
fi
helm version --short
```

**预期输出**：eksctl、kubectl、helm 版本信息正常输出

### 3. 创建 EKS 集群

用声明式的 ClusterConfig 创建集群，指定 1.35 版本与托管节点组。eksctl 会通过 CloudFormation 创建 VPC、子网和节点组，这一步耗时最长，需耐心等待并轮询节点状态。

```bash
cat > /tmp/cluster.yaml <<EOF
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: ${CLUSTER_NAME}
  region: ${AWS_REGION}
  version: "1.35"
managedNodeGroups:
  - name: ng-default
    instanceType: ${NODE_TYPE}
    minSize: 2
    desiredCapacity: 2
    maxSize: 5
    volumeSize: 30
    labels:
      role: worker
    tags:
      Project: eks-china-quickstart
EOF

eksctl create cluster -f /tmp/cluster.yaml
```

**预期输出**：集群创建完成，输出 `EKS cluster "demo" in "cn-northwest-1" region is ready`（约 15-20 分钟）

> ⚠️ **命令完整性**：`eksctl create cluster` 必须作为单条完整命令执行，不要和 `source` 命令误合并到同一行，否则会留下没有 Outputs 的损坏 CF 栈，导致后续 `create nodegroup` 报错。若已发生，`eksctl delete cluster --wait` 删除后用单条干净命令重建。

> ⚠️ **中国区 describe API 可能返回不一致读**：创建过程中 `describe-cluster`/`describe-stacks` 可能间歇性返回过期或不存在的状态。判定完成请以 eksctl 日志的 `EKS cluster ... is ready` 与 CloudFormation 栈 `CREATE_COMPLETE` 为准，不要依赖单次轮询结果。CN 区控制面创建偶尔需 10-15 分钟，属正常。

### 4. 验证集群连接

eksctl 创建集群后会自动写入 kubeconfig。确认能列出节点且全部 `Ready`，再继续后续步骤；若节点不 Ready 说明节点组未就绪，需排查。

```bash
kubectl get nodes -o wide
kubectl get pods -A
```

**预期输出**：2 个节点状态为 `Ready`，系统 Pod 运行正常（节点 `ROLES` 列显示 `<none>` 属正常，自定义标签 `role: worker` 不计入内置 ROLES）

> ⚠️ **kubectl 报 401 `You must be logged in to the server`**：若操作机 `~/.kube/config` 里残留同名但不同区/账号的旧 `demo` context，可能落在错误 context 上导致鉴权失败。用 `kubectl config current-context` 确认指向 `cn-northwest-1`；如有陈旧条目先清理，再重新 `aws eks update-kubeconfig`，并确认 exec 凭据块带 `AWS_PROFILE=cn`。

### 5. 将集群信息写入环境变量文件

VPC ID 和节点 IAM Role 会被后续多个 Demo（ALB Controller、Karpenter 等）反复引用。提前查询并追加到环境变量文件，避免每次重新查找。

```bash
export VPC_ID=$(aws eks describe-cluster \
  --name ${CLUSTER_NAME} \
  --query 'cluster.resourcesVpcConfig.vpcId' --output text)

export NODE_ROLE=$(aws cloudformation describe-stack-resources \
  --stack-name eksctl-${CLUSTER_NAME}-nodegroup-ng-default \
  --query 'StackResources[?ResourceType==`AWS::IAM::Role`].PhysicalResourceId' \
  --output text)

cat >> /tmp/demo-eks.env <<EOF
export VPC_ID=${VPC_ID}
export NODE_ROLE=${NODE_ROLE}
EOF

echo "VPC_ID: ${VPC_ID}"
echo "NODE_ROLE: ${NODE_ROLE}"
```

**预期输出**：VPC ID 和节点 IAM Role 名输出

> ⚠️ `NODE_ROLE` 依赖节点组 CF 栈 `eksctl-demo-nodegroup-ng-default` 已 `CREATE_COMPLETE`。若该栈不存在（如上一步集群创建被打断、节点组未真正建出），此查询会返回空值——应先确认 `eksctl get nodegroup --cluster demo` 能列出 `ng-default` 且节点 `Ready` 后再执行本步。

### 6. 安装 EKS Pod Identity Agent addon

Pod Identity 是 EKS 推荐的为 Pod 授予 AWS 权限的机制，后续多个 Demo 依赖它。必须等待 addon 状态变为 ACTIVE 才算安装成功。

```bash
aws eks create-addon \
  --cluster-name ${CLUSTER_NAME} \
  --addon-name eks-pod-identity-agent \
  --region ${AWS_REGION}

aws eks wait addon-active \
  --cluster-name ${CLUSTER_NAME} \
  --addon-name eks-pod-identity-agent \
  --region ${AWS_REGION}

echo "Pod Identity Agent 已激活"
```

**预期输出**：打印"Pod Identity Agent 已激活"

### 7. 扩展到 3 个节点（可选，验证节点管理）

通过 eksctl 调整节点组规模，验证托管节点组的弹性。这一步可选，用于熟悉节点扩缩容操作。

```bash
eksctl scale nodegroup \
  --cluster ${CLUSTER_NAME} \
  --name ng-default \
  --nodes 3 \
  --region ${AWS_REGION}

kubectl wait nodes --all --for=condition=Ready --timeout=300s
kubectl get nodes
```

**预期输出**：3 个节点均为 `Ready` 状态

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 在中国区成功创建一个 1.35 版本、状态为 ACTIVE 的 EKS 集群
- [ ] 使用 kubectl 列出全部处于 Ready 状态的工作节点
- [ ] 确认 eks-pod-identity-agent addon 已处于 ACTIVE 状态
- [ ] 集群关键信息（VPC_ID、NODE_ROLE 等）已持久化到 /tmp/demo-eks.env

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws eks describe-cluster --name demo --region cn-northwest-1 --query 'cluster.status' --output text` | `ACTIVE` |
| 2 | `aws eks describe-cluster --name demo --region cn-northwest-1 --query 'cluster.version' --output text` | `1.35` |
| 3 | `aws eks describe-addon --cluster-name demo --addon-name eks-pod-identity-agent --region cn-northwest-1 --query 'addon.status' --output text` | `ACTIVE` |
| 4 | `kubectl get nodes --no-headers \| wc -l` | `2` 或 `3`（取决于是否执行步骤 7） |

---

## 实验总结

本实验在中国区从零创建了 EKS 集群，并安装了 Pod Identity Agent，构建出后续所有 Demo 共用的基础环境。你掌握了中国区特有的 `aws-cn` 分区、ECR 域名规则以及通过本地工具文件离线安装工具的方式。集群信息已持久化，下一个 Demo（Demo02）将在此集群上部署 AWS Load Balancer Controller 并通过 ALB 暴露 NGINX 应用。

---

## 清理

```bash
source /tmp/demo-eks.env

eksctl delete cluster \
  --name ${CLUSTER_NAME} \
  --region ${AWS_REGION}
```

> **注意**：此清理命令会删除整个集群，建议在所有 Demo 完成后执行。
