# Demo03 — ECR 私有镜像与镜像发布

## 实验简介

本实验完整演练在中国区使用 ECR 私有镜像仓库的镜像生命周期：构建并推送多版本镜像、部署到 EKS、漏洞扫描、生命周期清理策略以及跨区复制。中国区 ECR 域名后缀为 `.amazonaws.com.cn`，是承载所有自建与第三方镜像的核心服务。

**实验目标：**
- 掌握 ECR 仓库创建、登录、多版本镜像构建与推送
- 理解 ECR 镜像扫描、生命周期策略与跨区复制的作用
- 能够将私有镜像部署到 EKS 并验证运行

**实验流程：**
1. 设置环境变量
2. 创建启用扫描的 ECR 仓库
3. 构建并推送 v1 镜像
4. 构建并推送 v2 镜像
5. 部署 v1 应用到 EKS
6. 扫描镜像漏洞
7. 配置生命周期策略
8. 配置跨区复制（可选）

**预计时长：** 20-30 分钟

---

## 前提条件

- **工具**：AWS CLI v2、kubectl、docker
- **权限**：ECR、EKS 权限
- **前提**：Demo01 已完成
- **预计耗时**：20-30 分钟

---

## 步骤

### 1. 设置环境变量

`source` 复用集群环境，并定义本 Demo 的 ECR 仓库名与应用命名空间。

```bash
source /tmp/demo-eks.env
export ECR_REPO=demo-eks-web
export APP_NS=ecr-demo
```

### 2. 创建 ECR 仓库

创建仓库时开启 `scanOnPush=true`，使每次推送自动触发漏洞扫描。`|| true` 保证仓库已存在时不中断。

```bash
aws ecr create-repository \
  --repository-name ${ECR_REPO} \
  --image-scanning-configuration scanOnPush=true \
  --region ${AWS_REGION} || true

export REPO_URI=${ECR_REGISTRY}/${ECR_REPO}
echo "仓库 URI: ${REPO_URI}"
```

**预期输出**：仓库 URI 输出

### 3. 构建并推送 v1 镜像

基于中国区可访问的 `public.ecr.aws` nginx 基础镜像构建自定义应用，推送 v1 并额外打 `stable` 标签。推送前必须先 `docker login` 到 ECR。

```bash
mkdir -p /tmp/app-src
cat > /tmp/app-src/index.html <<'HTML'
<!DOCTYPE html>
<html><body>
<h1>EKS China QuickStart v1</h1>
<p>Hostname: HOSTNAME_PLACEHOLDER</p>
</body></html>
HTML

cat > /tmp/app-src/Dockerfile <<'DOCKER'
FROM 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/nginx:1.27-alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
DOCKER

aws ecr get-login-password --region ${AWS_REGION} | \
  docker login --username AWS --password-stdin ${ECR_REGISTRY}

docker build -t ${REPO_URI}:v1 /tmp/app-src/
docker push ${REPO_URI}:v1
docker tag ${REPO_URI}:v1 ${REPO_URI}:stable
docker push ${REPO_URI}:stable

echo "v1 镜像推送完成"
```

**预期输出**：打印"v1 镜像推送完成"

### 4. 构建并推送 v2 镜像

修改页面内容后构建 v2，模拟应用版本迭代，为后续滚动升级和镜像数量验证准备多个版本。

```bash
sed -i 's/v1/v2/' /tmp/app-src/index.html

docker build -t ${REPO_URI}:v2 /tmp/app-src/
docker push ${REPO_URI}:v2

echo "v2 镜像推送完成"
```

**预期输出**：打印"v2 镜像推送完成"

### 5. 部署 v1 应用到 EKS

将私有 ECR 中的 v1 镜像部署为 Deployment+Service。EKS 节点角色默认具备拉取本账号 ECR 的权限，无需额外配置 imagePullSecret。

```bash
kubectl create namespace ${APP_NS} --dry-run=client -o yaml | kubectl apply -f -

cat > /tmp/ecr-demo-deploy.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: ${APP_NS}
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: ${REPO_URI}:v1
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "128Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: ${APP_NS}
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 80
  selector:
    app: web
EOF

kubectl apply -f /tmp/ecr-demo-deploy.yaml
kubectl -n ${APP_NS} rollout status deployment/web
kubectl -n ${APP_NS} get pods -o wide
```


> ⚠️ **首次拉取 048912060910 镜像若出现 ImagePullBackOff（403）**：这是节点 ECR credential provider 冷启动的短暂问题，无需修改任何权限配置。等待约 30 秒后执行 `kubectl rollout restart deployment/<name> -n <ns>` 即可恢复。节点一旦缓存镜像后，同节点后续 Pod 直接走本地缓存不再触发。

**预期输出**：2 个 Pod 状态为 `Running`

### 6. 扫描镜像漏洞

主动触发并轮询扫描状态直到 COMPLETE，再查看按严重程度分类的漏洞统计。若 scanOnPush 已触发，重复 start 会无害失败，故用 `|| true`。

> ⚠️ **中国区限制**：`aws ecr start-image-scan` 在中国区会返回 `ValidationException: This feature is disabled`，扫描状态显示为 `ACTIVE`（不会变为 `COMPLETE`）。中国区仅支持 `scanOnPush=true` 自动触发扫描，无需等待 `COMPLETE` 状态，直接查询 findings 即可获取漏洞统计。

```bash
# 中国区：start-image-scan 不可用，直接查询 scanOnPush 自动产生的结果
aws ecr describe-image-scan-findings \
  --repository-name ${ECR_REPO} \
  --image-id imageTag=v1 \
  --region ${AWS_REGION} \
  --query '{status: imageScanStatus, counts: imageScanFindings.findingSeverityCounts}'
```

**预期输出**：漏洞摘要（按严重程度分类，状态为 `ACTIVE`）

### 7. 配置生命周期策略

生命周期策略自动清理旧镜像控制存储成本：保留最新 10 个带 `v` 前缀的 tag 镜像，并清理 7 天前的未打 tag 镜像。

```bash
cat > /tmp/ecr-lifecycle.json <<'JSON'
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "保留最新 10 个带 tag 的镜像",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["v"],
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": {"type": "expire"}
    },
    {
      "rulePriority": 2,
      "description": "清理 7 天前未打 tag 的镜像",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 7
      },
      "action": {"type": "expire"}
    }
  ]
}
JSON

aws ecr put-lifecycle-policy \
  --repository-name ${ECR_REPO} \
  --lifecycle-policy-text file:///tmp/ecr-lifecycle.json \
  --region ${AWS_REGION}

echo "生命周期策略已设置"
```

**预期输出**：打印"生命周期策略已设置"

### 8. 配置跨区复制（可选，复制到 cn-north-1）

配置 ECR 跨区复制规则，将匹配前缀的镜像自动同步到中国另一区域 cn-north-1，用于多区域容灾。此步骤可选。

```bash
aws ecr put-replication-configuration \
  --replication-configuration '{
    "rules": [{
      "destinations": [{
        "region": "cn-north-1",
        "registryId": "'"${ACCOUNT_ID}"'"
      }],
      "repositoryFilters": [{
        "filter": "demo-eks-web",
        "filterType": "PREFIX_MATCH"
      }]
    }]
  }' \
  --region ${AWS_REGION}

echo "跨区复制配置完成"
```

**预期输出**：打印"跨区复制配置完成"

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 在 ECR 中创建仓库并推送至少两个版本（v1、v2）的镜像
- [ ] 触发并查看镜像漏洞扫描结果
- [ ] 将私有 ECR 镜像成功部署到 EKS（2 个 Pod Running）
- [ ] 为仓库配置包含 2 条规则的生命周期策略

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws ecr describe-repositories --repository-names demo-eks-web --region cn-northwest-1 --query 'repositories[0].repositoryName' --output text` | `demo-eks-web` |
| 2 | `aws ecr list-images --repository-name demo-eks-web --region cn-northwest-1 --filter tagStatus=TAGGED --query 'length(imageIds)' --output text` | 大于等于 `2` |
| 3 | `kubectl get deployment web -n ecr-demo -o jsonpath='{.status.readyReplicas}'` | `2` |
| 4 | `aws ecr get-lifecycle-policy --repository-name demo-eks-web --region cn-northwest-1 --query 'lifecyclePolicyText' --output text \| python3 -c "import json,sys; p=json.load(sys.stdin); print(len(p['rules']))"` | `2` |

---

## 实验总结

本实验完整走通了中国区 ECR 私有镜像的发布与治理流程：多版本构建推送、漏洞扫描、生命周期清理与跨区复制，并将私有镜像部署到 EKS。你建立了对镜像供应链管理的理解，这是中国区所有依赖私有镜像的 Demo 的基础。下一个 Demo（Demo04）将聚焦应用健康检查与常见故障场景的排查。

---

## 清理

```bash
source /tmp/demo-eks.env

kubectl delete -f /tmp/ecr-demo-deploy.yaml
kubectl delete namespace ${APP_NS}

aws ecr delete-repository \
  --repository-name ${ECR_REPO} \
  --force \
  --region ${AWS_REGION}
```
