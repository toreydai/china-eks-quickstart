# Demo16 — NetworkPolicy 限制 Pod 流量

## 实验简介

本实验使用 Kubernetes NetworkPolicy 实现 Pod 间的网络隔离。中国区 EKS 同样由 VPC CNI 负责执行网络策略，需先在 addon 中开启。通过「应用策略前全通 → 应用 Ingress 策略 → 仅放行特定标签 Pod → 追加 Egress 限制」的递进演练，建立零信任网络的实践认知。

**实验目标：**
- 掌握在 VPC CNI 中开启 NetworkPolicy 支持
- 理解 Ingress/Egress 策略基于标签选择器的放行与拒绝逻辑
- 能够验证策略生效前后 Pod 间连通性的变化

**实验流程：**
1. 设置环境变量
2. 启用 VPC CNI 网络策略支持
3. 部署 Frontend、Backend 和测试 Pod
4. 验证应用策略前所有 Pod 均可访问
5. 应用 NetworkPolicy — 只允许 role=allowed 的 Pod 访问
6. 验证 NetworkPolicy 生效
7. 添加 Egress 策略（可选）

**预计 AI 执行时长：** 15-25 分钟

## 前提条件

- **工具**：AWS CLI v2、kubectl
- **权限**：EKS 权限
- **前提**：Demo01 已完成

---

## 步骤

### 1. 设置环境变量

复用集群环境并创建独立命名空间承载前后端与测试 Pod。

```bash
source /tmp/demo-eks.env
export DEMO_NS=netpol-demo
kubectl create namespace ${DEMO_NS} --dry-run=client -o yaml | kubectl apply -f -
```

### 2. 启用 VPC CNI 网络策略支持

更新 vpc-cni addon 开启 `enableNetworkPolicy`，使集群原生支持 NetworkPolicy（中国区同样由 VPC CNI 执行）。

```bash
aws eks update-addon \
  --cluster-name ${CLUSTER_NAME} \
  --addon-name vpc-cni \
  --configuration-values '{"enableNetworkPolicy":"true"}' \
  --resolve-conflicts OVERWRITE \
  --region ${AWS_REGION}

aws eks wait addon-active \
  --cluster-name ${CLUSTER_NAME} \
  --addon-name vpc-cni \
  --region ${AWS_REGION}

echo "VPC CNI 网络策略已启用"
```

**预期输出**：打印"VPC CNI 网络策略已启用"

### 3. 部署 Frontend、Backend 和测试 Pod

部署 backend 服务，以及带 role=allowed 标签和无标签两个客户端 Pod，作为策略验证的对照组。

```bash
cat > /tmp/netpol-apps.yaml <<'EOF'
# Backend 服务
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: netpol-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
        role: backend
    spec:
      containers:
        - name: backend
          image: 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/nginx:1.27-alpine
          ports:
            - containerPort: 80
          resources:
            limits:
              cpu: "100m"
              memory: "64Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: netpol-demo
spec:
  type: ClusterIP
  ports:
    - port: 80
  selector:
    app: backend
---
# Allowed 前端（有权访问 backend）
apiVersion: v1
kind: Pod
metadata:
  name: frontend-allowed
  namespace: netpol-demo
  labels:
    app: frontend
    role: allowed
spec:
  containers:
    - name: client
      image: 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/busybox:1.36
      command: ["sleep", "3600"]
      resources:
        limits:
          cpu: "50m"
          memory: "32Mi"
---
# Denied Pod（无权访问 backend）
apiVersion: v1
kind: Pod
metadata:
  name: unauthorized-client
  namespace: netpol-demo
  labels:
    app: unauthorized
spec:
  containers:
    - name: client
      image: 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/busybox:1.36
      command: ["sleep", "3600"]
      resources:
        limits:
          cpu: "50m"
          memory: "32Mi"
EOF

kubectl apply -f /tmp/netpol-apps.yaml
kubectl -n ${DEMO_NS} rollout status deployment/backend
kubectl wait pod frontend-allowed -n ${DEMO_NS} --for=condition=Ready --timeout=60s
kubectl wait pod unauthorized-client -n ${DEMO_NS} --for=condition=Ready --timeout=60s

echo "所有 Pod 就绪"
```


> ⚠️ **首次拉取 048912060910 镜像若出现 ImagePullBackOff（403）**：这是节点 ECR credential provider 冷启动的短暂问题，无需修改任何权限配置。等待约 30 秒后执行 `kubectl rollout restart deployment/<name> -n <ns>` 即可恢复。节点一旦缓存镜像后，同节点后续 Pod 直接走本地缓存不再触发。

**预期输出**：所有 Pod Running

### 4. 验证应用策略前所有 Pod 均可访问

在应用策略前确认两个客户端都能访问 backend，建立『默认全通』的基线。

```bash
echo "=== 应用 NetworkPolicy 前 ==="

echo "frontend-allowed → backend（应成功）:"
kubectl exec -n ${DEMO_NS} frontend-allowed -- wget -qO- --timeout=5 http://backend.${DEMO_NS}.svc.cluster.local/ 2>/dev/null | head -1

echo "unauthorized-client → backend（此时也成功）:"
kubectl exec -n ${DEMO_NS} unauthorized-client -- wget -qO- --timeout=5 http://backend.${DEMO_NS}.svc.cluster.local/ 2>/dev/null | head -1
```

**预期输出**：两个 Pod 都能访问 backend

### 5. 应用 NetworkPolicy — 只允许 role=allowed 的 Pod 访问

创建 Ingress 策略只放行带 role=allowed 标签的 Pod，其余入站流量被默认拒绝。

```bash
cat > /tmp/backend-netpol.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow-only
  namespace: netpol-demo
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: allowed
      ports:
        - protocol: TCP
          port: 80
EOF

kubectl apply -f /tmp/backend-netpol.yaml
kubectl get networkpolicy -n ${DEMO_NS}

# 等待 CNI 策略生效
echo "等待 NetworkPolicy 生效（约 60-70 秒，VPC CNI BPF 规则同步需时）..."
sleep 70
```

**预期输出**：NetworkPolicy 创建成功

### 6. 验证 NetworkPolicy 生效

复测两个客户端——allowed 仍可访问、unauthorized 被拒绝，证明策略已生效。

```bash
echo "=== 应用 NetworkPolicy 后 ==="

echo "frontend-allowed → backend（应成功）:"
kubectl exec -n ${DEMO_NS} frontend-allowed -- wget -qO- --timeout=5 http://backend.${DEMO_NS}.svc.cluster.local/ 2>/dev/null | head -1 && echo "✓ 访问成功" || echo "✗ 访问被拒绝"

echo ""
echo "unauthorized-client → backend（应被拒绝）:"
kubectl exec -n ${DEMO_NS} unauthorized-client -- wget -qO- --timeout=5 http://backend.${DEMO_NS}.svc.cluster.local/ 2>/dev/null | head -1 && echo "✗ 仍然可以访问（等待更长时间）" || echo "✓ 访问已被拒绝"
```

**预期输出**：`frontend-allowed` 访问成功；`unauthorized-client` 访问超时/被拒绝

### 7. 添加 Egress 策略（可选）

追加一条空 egress 策略彻底禁止 unauthorized Pod 的出站流量，演示出站方向的隔离。

```bash
cat > /tmp/egress-netpol.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: netpol-demo
spec:
  podSelector:
    matchLabels:
      app: unauthorized
  policyTypes:
    - Egress
  egress: []
EOF

kubectl apply -f /tmp/egress-netpol.yaml

echo "Egress NetworkPolicy 已应用，unauthorized Pod 无法发起出站连接"
sleep 10

echo "unauthorized-client → backend（Egress 拒绝）:"
kubectl exec -n ${DEMO_NS} unauthorized-client -- wget -qO- --timeout=5 http://backend.${DEMO_NS}.svc.cluster.local/ 2>/dev/null || echo "✓ Egress 访问已拒绝"
```

**预期输出**：`unauthorized-client` 出站被拒绝

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 在 vpc-cni addon 中成功开启 NetworkPolicy 支持
- [ ] 应用策略前两个客户端 Pod 都能访问 backend
- [ ] 应用 Ingress 策略后只有 role=allowed 的 Pod 能访问 backend
- [ ] 追加 Egress 策略后 unauthorized Pod 的出站流量被拒绝

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws eks describe-addon --cluster-name demo --addon-name vpc-cni --region cn-northwest-1 --query 'addon.configurationValues' --output text \| python3 -c "import json,sys; print(json.loads(sys.stdin.read()).get(\"enableNetworkPolicy\",\"false\"))"` | `true` |
| 2 | `kubectl get networkpolicy backend-allow-only -n netpol-demo -o jsonpath='{.spec.policyTypes[0]}'` | `Ingress` |
| 3 | `kubectl exec -n netpol-demo frontend-allowed -- wget -qO- --timeout=5 http://backend.netpol-demo.svc.cluster.local/ 2>/dev/null \| wc -c` | 大于 `0`（HTTP 响应内容非空） |

---

## 实验总结

本实验用 NetworkPolicy 实现了 Pod 级的网络微隔离：从默认全通到基于标签的 Ingress 放行，再到 Egress 出站限制，完整演示了零信任网络的收敛过程。你也理解了中国区由 VPC CNI 执行策略的机制。下一个 Demo（Demo17）将进入 Pod Security Standards 的工作负载安全加固。

## 清理

```bash
kubectl delete namespace netpol-demo

# 如需关闭网络策略（可选）
# aws eks update-addon \
#   --cluster-name ${CLUSTER_NAME} \
#   --addon-name vpc-cni \
#   --configuration-values '{"enableNetworkPolicy":"false"}' \
#   --resolve-conflicts OVERWRITE \
#   --region ${AWS_REGION}
```
