# Demo06 — Helm 与 GitOps 发布（ArgoCD）

## 实验简介

本实验在中国区 EKS 上搭建基于 ArgoCD 的 GitOps 发布基础链路：自制 Helm Chart、推送到 CodeCommit 作为 Git 源，并创建 ArgoCD Application 观察同步状态。ArgoCD 相关镜像统一指向集中预置的 0910 镜像仓库（argocd/dex/redis 均已预置），避免从 quay.io/ghcr.io 等中国区不可稳定访问的上游仓库拉取。

**实验目标：**
- 掌握 Helm Chart 的编写、安装与版本化管理
- 理解 ArgoCD GitOps「以 Git 为唯一事实源」的自动同步机制
- 能够在中国区使用 CodeCommit 作为 Git 源并处理上游镜像仓库不可访问问题

**实验流程：**
1. 设置环境变量
2. 安装 ArgoCD
3. 创建本地 Helm Chart
4. 安装并测试 Helm Chart
5. 创建 CodeCommit 仓库作为 Git 源
6. 在 ArgoCD 中创建 Application 实现自动同步

**预计时长：** 30-40 分钟

---

## 前提条件

- **工具**：AWS CLI v2、kubectl、helm、git
- **权限**：EKS、ECR、CodeCommit、IAM 权限
- **前提**：Demo01 已完成；Demo02 的 ALB Controller 已安装（可选）
- **说明**：ArgoCD 镜像需使用 0910 镜像仓库中的预置版本
- **预计耗时**：30-40 分钟

---

## 步骤

### 1. 设置环境变量

复用集群环境，定义 ArgoCD 命名空间、Helm 应用命名空间与 CodeCommit 仓库名。

```bash
source /tmp/demo-eks.env
export ARGOCD_NS=argocd
export HELM_APP_NS=helm-demo
export REPO_NAME=eks-gitops-app
export ASSET_DIR=${ASSET_DIR:-$(pwd)/offline-assets}
```

### 2. 安装 ArgoCD

用预置的 install.yaml 安装 ArgoCD（`--server-side` 避免大型 CRD 注解超限）。下载后显式把 ArgoCD、Dex、Redis 镜像替换为 0910 仓库中的预置镜像，无需手动推送镜像。等待 argocd-server 就绪后读取初始 admin 密码，密码不直接打印以免泄露。

```bash
cp ${ASSET_DIR}/argocd-install.yaml /tmp/argocd-install.yaml

sed -i "s|quay.io/argoproj/argocd:v2.13.2|048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/quay/argoproj/argocd:v2.13.2|g" /tmp/argocd-install.yaml
sed -i "s|ghcr.io/dexidp/dex:v2.41.1|048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ghcr/dexidp/dex:v2.41.1|g" /tmp/argocd-install.yaml
sed -i "s|redis:7.0.15-alpine|048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/redis:7.0.15-alpine|g" /tmp/argocd-install.yaml

kubectl create namespace ${ARGOCD_NS} --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -n ${ARGOCD_NS} -f /tmp/argocd-install.yaml --server-side

kubectl -n ${ARGOCD_NS} rollout status deployment/argocd-server

# 获取初始 admin 密码
ARGOCD_PASSWORD=$(kubectl get secret argocd-initial-admin-secret \
  -n ${ARGOCD_NS} \
  -o jsonpath='{.data.password}' | base64 -d)
echo "ArgoCD admin 密码已获取（不显示）"
```


> ⚠️ **首次拉取 048912060910 镜像若出现 ImagePullBackOff（403）**：这是节点 ECR credential provider 冷启动的短暂问题，无需修改任何权限配置。等待约 30 秒后执行 `kubectl rollout restart deployment/<name> -n <ns>` 即可恢复。节点一旦缓存镜像后，同节点后续 Pod 直接走本地缓存不再触发。

**预期输出**：ArgoCD Deployment 就绪，密码已获取

### 3. 创建本地 Helm Chart

手写一个最小 Helm Chart（Chart.yaml + values.yaml + Deployment/Service 模板），用模板变量参数化镜像、副本数和 Service 类型，作为 GitOps 部署的应用源。

```bash
mkdir -p /tmp/myapp/templates

cat > /tmp/myapp/Chart.yaml <<'EOF'
apiVersion: v2
name: myapp
description: EKS China Demo App
version: 0.1.0
appVersion: "1.0"
EOF

cat > /tmp/myapp/values.yaml <<EOF
image:
  repository: 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/nginx
  tag: "1.27-alpine"
  pullPolicy: IfNotPresent
replicas: 2
service:
  type: ClusterIP
  port: 80
EOF

cat > /tmp/myapp/templates/deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-app
  labels:
    app: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicas }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "100m"
              memory: "128Mi"
EOF

cat > /tmp/myapp/templates/service.yaml <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-svc
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: 80
  selector:
    app: {{ .Release.Name }}
EOF
```

**预期输出**：Helm chart 目录结构创建完成

### 4. 安装并测试 Helm Chart

先用 `helm install` 在本地直接部署一次，验证 Chart 可正常渲染并运行，再交给 ArgoCD 做 GitOps 管理。`helm list` 应显示 Release 状态为 deployed。

```bash
kubectl create namespace ${HELM_APP_NS} --dry-run=client -o yaml | kubectl apply -f -

helm install myapp /tmp/myapp \
  -n ${HELM_APP_NS} \
  --set replicas=2

kubectl -n ${HELM_APP_NS} rollout status deployment/myapp-app

# 验证
helm list -n ${HELM_APP_NS}
kubectl get all -n ${HELM_APP_NS}
```

**预期输出**：Helm Release 状态为 `deployed`，2 个 Pod Running

### 5. 创建 CodeCommit 仓库作为 Git 源

中国区使用 CodeCommit 作为 Git 源（URL 后缀 `.amazonaws.com.cn`），通过 `aws codecommit credential-helper` 完成 git 认证，将 Helm Chart 推送上去作为 ArgoCD 的同步源。

```bash
aws codecommit create-repository \
  --repository-name ${REPO_NAME} \
  --region ${AWS_REGION} || true

cd /tmp
mkdir -p gitops-repo && cd gitops-repo

git init
git config user.email "eks-demo@example.com"
git config user.name "eks-demo"

mkdir -p apps/myapp
cp -r /tmp/myapp/* apps/myapp/

git add .
git commit -m "initial helm chart"
git branch -M main

git config --global credential.helper '!aws codecommit credential-helper $@'
git config --global credential.UseHttpPath true

CC_URL="https://git-codecommit.${AWS_REGION}.amazonaws.com.cn/v1/repos/${REPO_NAME}"
git remote add origin ${CC_URL}
git push -u origin main

echo "代码已推送到 CodeCommit"
```

**预期输出**：打印"代码已推送到 CodeCommit"

### 6. 在 ArgoCD 中创建 Application

直接用 kubectl 创建 Application CRD（无需 argocd CLI），指向 CodeCommit 中的 Helm 路径，开启 automated sync（prune + selfHeal）。本步骤用于观察 ArgoCD 如何声明 GitOps 同步对象；CodeCommit 是私有仓库，如未额外给 ArgoCD 配置 Git 凭证，Application 可能显示 `OutOfSync` 或 repo 认证错误，这是预期的私有仓库权限边界，不影响本 Demo 前面已经完成的 Helm Chart 打包、CodeCommit 推送和 ArgoCD 安装验证。

```bash
# 使用 kubectl 创建 Application CRD（无需 argocd CLI）
CC_URL="https://git-codecommit.${AWS_REGION}.amazonaws.com.cn/v1/repos/${REPO_NAME}"

cat > /tmp/argocd-app.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: ${ARGOCD_NS}
spec:
  project: default
  source:
    repoURL: ${CC_URL}
    targetRevision: main
    path: apps/myapp
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: ${HELM_APP_NS}
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF

kubectl apply -f /tmp/argocd-app.yaml
sleep 30
kubectl get application myapp -n ${ARGOCD_NS}
kubectl describe application myapp -n ${ARGOCD_NS} | sed -n '/Conditions:/,/Events:/p'
```

**预期输出**：Application 对象创建成功；如未配置 ArgoCD 访问 CodeCommit 的仓库凭证，状态可能为 `OutOfSync` 或显示 repo 认证相关条件。

---

## 验收标准

完成本实验后，你应当能够：
- [ ] ArgoCD 在集群中正常运行（argocd-server readyReplicas 为 1）
- [ ] 用自制 Helm Chart 成功部署应用并看到 Release 状态为 deployed
- [ ] 在 CodeCommit 创建仓库并将 Helm Chart 推送上去
- [ ] 创建 ArgoCD Application 并观察其与私有 Git 源的同步或认证状态

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `kubectl get deployment argocd-server -n argocd -o jsonpath='{.status.readyReplicas}'` | `1` |
| 2 | `helm list -n helm-demo -o json \| python3 -c "import json,sys; r=json.load(sys.stdin); print(r[0]['status'])"` | `deployed` |
| 3 | `aws codecommit get-repository --repository-name eks-gitops-app --region cn-northwest-1 --query 'repositoryMetadata.repositoryName' --output text` | `eks-gitops-app` |
| 4 | `kubectl get application myapp -n argocd -o jsonpath='{.metadata.name}'` | `myapp` |

---

## 实验总结

本实验在中国区搭建了 GitOps 发布基础链路：Helm Chart 打包、CodeCommit 托管、ArgoCD 安装与 Application 声明，并通过 0910 预置镜像解决了上游镜像仓库不可访问问题。CodeCommit 私有仓库的 ArgoCD 凭证配置属于生产集成项，本 Demo 保留为状态观察。下一个 Demo（Demo07）将转向调度策略与资源治理（LimitRange、亲和性、污点容忍等）。

---

## 清理

```bash
source /tmp/demo-eks.env

kubectl delete application myapp -n ${ARGOCD_NS} || true
helm uninstall myapp -n ${HELM_APP_NS} || true
kubectl delete namespace ${HELM_APP_NS} || true

kubectl delete -f /tmp/argocd-install.yaml || true
kubectl delete namespace ${ARGOCD_NS} || true

aws codecommit delete-repository \
  --repository-name ${REPO_NAME} \
  --region ${AWS_REGION}
```
