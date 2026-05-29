# Demo07 — 调度策略与资源治理

## 实验简介

本实验系统演练 Kubernetes 的资源治理与高级调度能力：用 LimitRange/ResourceQuota 约束命名空间资源用量，用 nodeSelector、podAntiAffinity、topologySpreadConstraints、Taints/Tolerations 精确控制 Pod 落在哪些节点。这些是多租户集群稳定性与高可用布局的核心手段。

**实验目标：**
- 掌握 LimitRange 默认值注入与 ResourceQuota 配额限制
- 理解 nodeSelector、亲和性、拓扑分散、污点容忍四种调度机制的差异
- 能够让工作负载按预期分布到指定或分散的节点上

**实验流程：**
1. 设置环境变量与命名空间
2. 配置 LimitRange
3. 配置 ResourceQuota
4. 验证 LimitRange 默认值注入
5. nodeSelector 指定节点调度
6. podAntiAffinity 跨节点分散
7. topologySpreadConstraints 均匀分布
8. Taints 与 Tolerations

**预计时长：** 20-30 分钟

---

## 前提条件

- **工具**：AWS CLI v2、kubectl
- **权限**：EKS 权限
- **前提**：Demo01 已完成
- **预计耗时**：20-30 分钟

---

## 步骤

### 1. 设置环境变量

复用集群环境并创建独立命名空间，后续的配额、调度实验都隔离在此命名空间内。

```bash
source /tmp/demo-eks.env
export DEMO_NS=scheduling-demo
kubectl create namespace ${DEMO_NS} --dry-run=client -o yaml | kubectl apply -f -
```

### 2. 配置 LimitRange

LimitRange 为命名空间内容器设定默认 request/limit 及上下限。未显式声明资源的 Pod 会被自动注入 default 值，避免「裸 Pod」抢占节点资源。

```bash
cat > /tmp/limitrange.yaml <<'EOF'
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: scheduling-demo
spec:
  limits:
    - type: Container
      default:
        cpu: "200m"
        memory: "256Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "2"
        memory: "2Gi"
      min:
        cpu: "50m"
        memory: "64Mi"
EOF

kubectl apply -f /tmp/limitrange.yaml
kubectl describe limitrange default-limits -n ${DEMO_NS}
```

**预期输出**：LimitRange 创建成功，显示 default/max/min 配置

### 3. 配置 ResourceQuota

ResourceQuota 为整个命名空间设定资源总量上限（Pod 数、CPU/内存请求与限制、对象数量）。超过配额的创建请求会被直接拒绝，实现多租户隔离。

```bash
cat > /tmp/resourcequota.yaml <<'EOF'
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ns-quota
  namespace: scheduling-demo
spec:
  hard:
    pods: "20"
    requests.cpu: "4"
    requests.memory: "4Gi"
    limits.cpu: "8"
    limits.memory: "8Gi"
    count/deployments.apps: "10"
    count/services: "10"
EOF

kubectl apply -f /tmp/resourcequota.yaml
kubectl describe resourcequota ns-quota -n ${DEMO_NS}
```

**预期输出**：ResourceQuota 创建成功，显示 hard 限额

### 4. 验证 LimitRange 默认值注入

部署一个完全不声明 resources 的 Pod，观察它被 LimitRange 自动注入 defaultRequest（100m/128Mi）。这验证了 LimitRange 对裸 Pod 的兜底保护。

```bash
# 部署未指定 resources 的 Pod，验证默认值注入
kubectl run test-defaults \
  --image=048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/nginx:1.27-alpine \
  -n ${DEMO_NS} \
  --restart=Never

kubectl get pod test-defaults -n ${DEMO_NS} \
  -o jsonpath='{.spec.containers[0].resources}'

kubectl delete pod test-defaults -n ${DEMO_NS}
```

**预期输出**：Pod resources 中已注入 LimitRange 的 default 值（`100m`/`128Mi`）

### 5. nodeSelector — 指定节点标签调度

nodeSelector 是最简单的硬性调度：给节点打标签，Pod 声明匹配标签后只会调度到这些节点。常用于把特定工作负载固定到专用节点池。

```bash
# 给一个节点打标签
NODE_NAME=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
kubectl label node ${NODE_NAME} workload=web --overwrite

# 部署使用 nodeSelector 的应用
cat > /tmp/nodeselector-deploy.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-nodeselector
  namespace: scheduling-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-ns
  template:
    metadata:
      labels:
        app: web-ns
    spec:
      nodeSelector:
        workload: web
      containers:
        - name: web
          image: 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/nginx:1.27-alpine
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "200m"
              memory: "256Mi"
EOF

kubectl apply -f /tmp/nodeselector-deploy.yaml
kubectl -n ${DEMO_NS} rollout status deployment/web-nodeselector
kubectl get pod -n ${DEMO_NS} -l app=web-ns -o wide
```


> ⚠️ **首次拉取 048912060910 镜像若出现 ImagePullBackOff（403）**：这是节点 ECR credential provider 冷启动的短暂问题，无需修改任何权限配置。等待约 30 秒后执行 `kubectl rollout restart deployment/<name> -n <ns>` 即可恢复。节点一旦缓存镜像后，同节点后续 Pod 直接走本地缓存不再触发。

**预期输出**：Pod 调度到带 `workload=web` 标签的节点

### 6. podAntiAffinity — 跨节点分散

podAntiAffinity 让同一应用的副本尽量不落在同一节点，以 `kubernetes.io/hostname` 为拓扑键。这里用 `preferred`（软约束），即使节点不足也能调度，提高可用性而不牺牲可调度性。

```bash
cat > /tmp/antiaffinity-deploy.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spread-app
  namespace: scheduling-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: spread
  template:
    metadata:
      labels:
        app: spread
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: spread
                topologyKey: kubernetes.io/hostname
      containers:
        - name: app
          image: 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/nginx:1.27-alpine
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "200m"
              memory: "256Mi"
EOF

kubectl apply -f /tmp/antiaffinity-deploy.yaml
kubectl -n ${DEMO_NS} rollout status deployment/spread-app
kubectl get pod -n ${DEMO_NS} -l app=spread -o wide
```

**预期输出**：2 个 Pod 分布在不同节点上

### 7. topologySpreadConstraints — 均匀分布

topologySpreadConstraints 通过 `maxSkew` 控制各拓扑域之间 Pod 数量的最大差值，实现更精细的均匀分布。`whenUnsatisfiable: DoNotSchedule` 表示无法满足时拒绝调度。

```bash
cat > /tmp/spread-deploy.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: even-spread
  namespace: scheduling-demo
spec:
  replicas: 4
  selector:
    matchLabels:
      app: even
  template:
    metadata:
      labels:
        app: even
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: even
      containers:
        - name: app
          image: 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/nginx:1.27-alpine
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "100m"
              memory: "128Mi"
EOF

kubectl apply -f /tmp/spread-deploy.yaml
kubectl -n ${DEMO_NS} rollout status deployment/even-spread
kubectl get pod -n ${DEMO_NS} -l app=even -o wide
```

**预期输出**：4 个 Pod 均匀分布，每节点 Pod 数量差不超过 1

### 8. Taints 与 Tolerations

Taint 给节点打「排斥」标记，只有声明了对应 Toleration 的 Pod 才能调度上去。这是 nodeSelector 的反向机制，用于预留专用节点（如 GPU 节点）。测试后记得用 `NoSchedule-` 移除 taint。

```bash
# 给节点打 taint
NODE_NAME=$(kubectl get nodes -o jsonpath='{.items[1].metadata.name}')
kubectl taint node ${NODE_NAME} dedicated=gpu:NoSchedule

# 普通 Pod 无法调度到 tainted 节点
kubectl run no-toleration \
  --image=048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/nginx:1.27-alpine \
  -n ${DEMO_NS} \
  --restart=Never

sleep 10
kubectl get pod no-toleration -n ${DEMO_NS} -o wide

# 有 toleration 的 Pod 可以调度
cat > /tmp/toleration-pod.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: with-toleration
  namespace: scheduling-demo
spec:
  tolerations:
    - key: dedicated
      operator: Equal
      value: gpu
      effect: NoSchedule
  containers:
    - name: app
      image: 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/nginx:1.27-alpine
      resources:
        requests:
          cpu: "50m"
          memory: "64Mi"
        limits:
          cpu: "100m"
          memory: "128Mi"
EOF

kubectl apply -f /tmp/toleration-pod.yaml
sleep 15
kubectl get pod with-toleration -n ${DEMO_NS} -o wide

# 清理 taint
kubectl taint node ${NODE_NAME} dedicated=gpu:NoSchedule-
kubectl delete pod no-toleration with-toleration -n ${DEMO_NS}
```

**预期输出**：`no-toleration` 调度到未 tainted 节点；`with-toleration` 可以调度到 tainted 节点

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 配置 LimitRange 并验证默认资源值被自动注入到裸 Pod
- [ ] 配置 ResourceQuota 限制命名空间的资源总量
- [ ] 使用 nodeSelector 将 Pod 调度到指定标签节点
- [ ] 使用 podAntiAffinity / topologySpreadConstraints 让副本跨节点分散
- [ ] 使用 Taints/Tolerations 控制 Pod 能否调度到受污染节点

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `kubectl get limitrange default-limits -n scheduling-demo -o jsonpath='{.spec.limits[0].default.cpu}'` | `200m` |
| 2 | `kubectl get resourcequota ns-quota -n scheduling-demo -o jsonpath='{.spec.hard.pods}'` | `20` |
| 3 | `kubectl get deployment spread-app -n scheduling-demo -o jsonpath='{.status.readyReplicas}'` | `2` |
| 4 | `kubectl get deployment even-spread -n scheduling-demo -o jsonpath='{.status.readyReplicas}'` | `4` |

---

## 实验总结

本实验覆盖了 Kubernetes 资源治理与调度的核心工具集：LimitRange/ResourceQuota 保障多租户资源边界，nodeSelector/亲和性/拓扑分散/污点容忍则实现从粗到细的 Pod 布局控制。这些能力是后续自动伸缩（Demo08-10）合理放置新 Pod 与新节点的基础。下一个 Demo（Demo08）将进入基于 HPA 的应用水平自动伸缩。

---

## 清理

```bash
kubectl delete namespace scheduling-demo
```
