# Demo02 — ALB Controller 与 NGINX 应用

## 实验简介

本实验在中国区 EKS 集群上安装 AWS Load Balancer Controller，并通过 Ingress 自动创建 ALB 将 NGINX 应用暴露到公网。镜像统一从集中预置的镜像仓库（048912060910）拉取，且 ALB 监听端口受 ICP 备案限制（80/443 需备案），因此本实验使用 8080 端口。

**实验目标：**
- 掌握用 Pod Identity 为 ALB Controller 授权的完整流程
- 理解 Ingress 与 ALB Controller 协同自动创建 ALB 的机制
- 能够引用集中预置的 0910 镜像仓库并通过 Helm 安装控制器

**实验流程：**
1. 设置环境变量
2. 创建 ALB Controller 中国区专用 IAM Policy
3. 创建 Pod Identity IAM Role 并绑定 Policy
4. 创建 Pod Identity Association
5. 下载 ALB Controller Helm Chart
6. 用 Helm 安装 ALB Controller
7. 部署 NGINX 示例应用与 Ingress
8. 获取 ALB 地址并验证访问

**预计时长：** 20-30 分钟

---

## 前提条件

- **工具**：AWS CLI v2、eksctl、kubectl、helm
- **权限**：EKS、EC2、ELBv2、IAM、ECR 权限
- **前提**：Demo01 已完成，集群 `demo` 运行中，Pod Identity Agent addon 已安装
- **预计耗时**：20-30 分钟

---

## 步骤

### 1. 设置环境变量

先 `source` Demo01 保存的环境文件复用集群信息，再定义本 Demo 专用的 ALB Controller 版本、ServiceAccount 和 Role 名称。

```bash
source /tmp/demo-eks.env
export ALB_CONTROLLER_VERSION=v3.3.0
export ALB_SA=aws-load-balancer-controller
export ALB_NS=kube-system
export ALB_ROLE=eks-alb-controller-role
export ASSET_DIR=${ASSET_DIR:-$(pwd)/offline-assets}
```

### 2. 创建 ALB Controller IAM Policy

中国区的 ALB Controller 需要专用的 IAM Policy（ARN 使用 `arn:aws-cn:`、ELB 端点不同），已预置在本仓库 `offline-assets/` 目录。`|| true` 避免 Policy 已存在时报错中断。

```bash
cp ${ASSET_DIR}/iam_policy_cn.json /tmp/iam_policy_cn.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file:///tmp/iam_policy_cn.json || true

export ALB_POLICY_ARN=$(aws iam list-policies \
  --query "Policies[?PolicyName=='AWSLoadBalancerControllerIAMPolicy'].Arn" \
  --output text)

echo "ALB Policy ARN: ${ALB_POLICY_ARN}"
```

**预期输出**：Policy ARN 输出

### 3. 创建 Pod Identity IAM Role

Pod Identity 的信任主体是 `pods.eks.amazonaws.com`（注意中国区此处不带 `.cn`）。因 AWS 托管 Policy 无法直接用 eksctl 关联，这里手动创建 Role 并 attach Policy。

```bash
cat > /tmp/alb-trust.json <<EOF
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
  --role-name ${ALB_ROLE} \
  --assume-role-policy-document file:///tmp/alb-trust.json || true

aws iam attach-role-policy \
  --role-name ${ALB_ROLE} \
  --policy-arn ${ALB_POLICY_ARN}

export ALB_ROLE_ARN=$(aws iam get-role \
  --role-name ${ALB_ROLE} \
  --query 'Role.Arn' --output text)

echo "ALB Role ARN: ${ALB_ROLE_ARN}"
```

**预期输出**：Role ARN 输出

### 4. 创建 Pod Identity Association

将 IAM Role 与 kube-system 命名空间下的 ServiceAccount 绑定，Pod 使用该 SA 即可自动获得 AWS 权限。必须先确保 ServiceAccount 存在再创建关联。

```bash
# 先确保 Service Account 存在
kubectl create serviceaccount ${ALB_SA} -n ${ALB_NS} --dry-run=client -o yaml | kubectl apply -f -

aws eks create-pod-identity-association \
  --cluster-name ${CLUSTER_NAME} \
  --namespace ${ALB_NS} \
  --service-account ${ALB_SA} \
  --role-arn ${ALB_ROLE_ARN} \
  --region ${AWS_REGION}

echo "Pod Identity Association 已创建"
```

**预期输出**：打印"Pod Identity Association 已创建"

### 5. 下载 ALB Controller Helm Chart

控制器镜像已集中预置在 0910 镜像仓库（`ecrpublic/eks/aws-load-balancer-controller:v3.3.0`），无需再下载 tar、retag 或推送。Helm chart 已放在本仓库 `offline-assets/` 目录，安装时通过 `--set image.*` 指向 0910 镜像。

```bash
cp ${ASSET_DIR}/aws-load-balancer-controller.tgz /tmp/aws-load-balancer-controller.tgz
```

**预期输出**：Helm chart 下载完成

> ⚠️ **跨账号 ECR 仓库策略前置条件**：节点 Role 具备 `AmazonEC2ContainerRegistryPullOnly`，但 `048912060910` 账号的 ECR 私有仓库**默认无跨账号策略**，节点拉取会收到 `403 Forbidden (ImagePullBackOff)`。需在 `048912060910` 账号对所有 Demo 涉及的仓库执行一次性 `set-repository-policy`，Principal 设为 `arn:aws-cn:iam::<YOUR_ACCOUNT_ID>:root`，Action 含 `ecr:GetDownloadUrlForLayer`、`ecr:BatchGetImage`、`ecr:BatchCheckLayerAvailability`。这是**所有 Demo 的共同前置**，一次设置后 Demo03-24 均受益，无需重复操作。

### 6. 安装 ALB Controller

通过 Helm 安装控制器，关键参数包括集群名、VPC ID、区域以及指向私有 ECR 的镜像地址；`serviceAccount.create=false` 复用上一步已绑定 Pod Identity 的 SA。

```bash
VPC_ID=$(aws eks describe-cluster \
  --name ${CLUSTER_NAME} \
  --query 'cluster.resourcesVpcConfig.vpcId' --output text)

helm upgrade --install aws-load-balancer-controller \
  /tmp/aws-load-balancer-controller.tgz \
  -n ${ALB_NS} \
  --set clusterName=${CLUSTER_NAME} \
  --set serviceAccount.create=false \
  --set serviceAccount.name=${ALB_SA} \
  --set region=${AWS_REGION} \
  --set vpcId=${VPC_ID} \
  --set enableShield=false \
  --set enableWaf=false \
  --set enableWafv2=false \
  --set image.repository=048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/eks/aws-load-balancer-controller \
  --set image.tag=${ALB_CONTROLLER_VERSION} \
  --set image.pullPolicy=IfNotPresent

kubectl -n ${ALB_NS} rollout status deployment/aws-load-balancer-controller
```


> ⚠️ **首次拉取 048912060910 镜像若出现 ImagePullBackOff（403）**：这是节点 ECR credential provider 冷启动的短暂问题，无需修改任何权限配置。等待约 30 秒后执行 `kubectl rollout restart deployment/<name> -n <ns>` 即可恢复。节点一旦缓存镜像后，同节点后续 Pod 直接走本地缓存不再触发。

**预期输出**：`deployment "aws-load-balancer-controller" successfully rolled out`

> ⚠️ **rollout status 超时**：若节点 ECR 跨账号策略未配置，`rollout status` 会因 `ImagePullBackOff` 超时失败。出现 `error: timed out waiting for the condition` 后，执行 `kubectl describe pod -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller` 确认是否为 `403 Forbidden` 错误，再按步骤 5 尾部的 ⚠️ 提示修复仓库策略，然后 `kubectl rollout restart deployment/aws-load-balancer-controller -n kube-system` 重试。

### 7. 部署 NGINX 示例应用

部署 NGINX Deployment、NodePort Service 和 Ingress。Ingress 注解中 `listen-ports` 设为 8080 而非 80，规避中国区 80/443 端口的 ICP 备案要求；`target-type: ip` 让 ALB 直接路由到 Pod IP。

```bash
kubectl create namespace alb-demo --dry-run=client -o yaml | kubectl apply -f -

cat > /tmp/alb-demo-app.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: alb-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: nginx
    spec:
      containers:
        - name: nginx
          image: 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/nginx:1.27-alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "200m"
              memory: "256Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: alb-demo
spec:
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  type: NodePort
  selector:
    app.kubernetes.io/name: nginx
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx
  namespace: alb-demo
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 8080}]'
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx
                port:
                  number: 80
EOF

kubectl apply -f /tmp/alb-demo-app.yaml
kubectl -n alb-demo rollout status deployment/nginx
```

**预期输出**：Deployment 就绪

> ⚠️ **Ingress 注解弃用警告**：`kubectl apply` 会输出 `Warning: annotation "kubernetes.io/ingress.class" is deprecated, please use 'spec.ingressClassName'`，这是 Kubernetes 上游的弃用提示，不影响 ALB Controller 实际处理，可忽略。生产环境建议将注解改为 `spec.ingressClassName: alb`。

### 8. 获取 ALB 地址并验证

ALB 创建需要 2-3 分钟，通过轮询 Ingress 的 `status.loadBalancer.ingress[0].hostname` 等待其就绪，再用 curl 验证返回 HTTP 200。

```bash
echo "等待 ALB 创建（约 2-3 分钟）..."
for i in $(seq 1 20); do
  ALB_DNS=$(kubectl get ingress nginx -n alb-demo \
    -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null)
  [ -n "${ALB_DNS}" ] && break
  sleep 15
done

echo "ALB DNS: ${ALB_DNS}"
curl -m10 -s -o /dev/null -w "HTTP %{http_code}" http://${ALB_DNS}:8080
```

**预期输出**：`HTTP 200`；浏览器访问 `http://<ALB_DNS>:8080` 显示 NGINX 欢迎页

---

## 验收标准

完成本实验后，你应当能够：
- [ ] ALB Controller Deployment 在 kube-system 中正常运行（readyReplicas 为 2）
- [ ] NGINX Ingress 已自动创建一个 internet-facing ALB 并获得 DNS 名称
- [ ] 通过 ALB 的 8080 端口访问能返回 HTTP 200 并看到 NGINX 欢迎页
- [ ] ALB Controller 的 ServiceAccount 已存在有效的 Pod Identity Association

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `kubectl get deployment aws-load-balancer-controller -n kube-system -o jsonpath='{.status.readyReplicas}'` | `2` |
| 2 | `kubectl get ingress nginx -n alb-demo -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' \| grep -c alb` | `1` |
| 3 | `aws eks list-pod-identity-associations --cluster-name demo --namespace kube-system --region cn-northwest-1 --query 'associations[?serviceAccount==\`aws-load-balancer-controller\`].associationId' --output text \| wc -w` | `1` |

---

## 实验总结

本实验在中国区集群上安装了 AWS Load Balancer Controller，并通过 Ingress 自动创建 ALB 将 NGINX 应用暴露到公网。你掌握了 Pod Identity 授权链路、从集中预置的 0910 仓库拉取控制器镜像，以及中国区端口备案规避（使用 8080）等关键实践。下一个 Demo（Demo03）将聚焦 ECR 私有镜像的构建、版本发布、漏洞扫描与生命周期管理。

---

## 清理

```bash
source /tmp/demo-eks.env

kubectl delete -f /tmp/alb-demo-app.yaml
kubectl delete namespace alb-demo

helm uninstall aws-load-balancer-controller -n kube-system

aws eks delete-pod-identity-association \
  --cluster-name ${CLUSTER_NAME} \
  --association-id $(aws eks list-pod-identity-associations \
    --cluster-name ${CLUSTER_NAME} \
    --namespace kube-system \
    --service-account aws-load-balancer-controller \
    --query 'associations[0].associationId' --output text) \
  --region ${AWS_REGION}

aws iam detach-role-policy \
  --role-name ${ALB_ROLE} \
  --policy-arn ${ALB_POLICY_ARN}
aws iam delete-role --role-name ${ALB_ROLE}
aws iam delete-policy --policy-arn ${ALB_POLICY_ARN}
```
