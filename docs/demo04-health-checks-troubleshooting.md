# Demo04 — 健康检查与故障排查

## 实验简介

本实验通过配置三种健康检查探针，并主动制造 5 类常见 Pod 故障，系统性地训练 EKS 工作负载的排障能力。这些故障场景（镜像拉取失败、崩溃循环、内存超限、资源不足、Service 无端点）是生产中最高频的问题，掌握其特征与定位方法至关重要。

**实验目标：**
- 掌握 Liveness / Readiness / Startup 三种探针的配置与作用
- 理解 ImagePullBackOff / CrashLoopBackOff / OOMKilled / Pending / 无 Endpoints 的成因
- 能够使用 describe、logs、kubectl debug 定位并诊断 Pod 故障

**实验流程：**
1. 设置环境变量与命名空间
2. 配置三种健康检查探针
3. 复现并诊断 ImagePullBackOff
4. 复现并诊断 CrashLoopBackOff
5. 复现并诊断 OOMKilled
6. 复现并诊断 Pending（资源不足）
7. 复现并诊断 Service 无 Endpoints
8. 使用 kubectl debug 临时容器排查

**预计时长：** 25-35 分钟

---

## 前提条件

- **工具**：AWS CLI v2、kubectl
- **权限**：EKS 权限
- **前提**：Demo01 已完成
- **预计耗时**：25-35 分钟

---

## 步骤

### 1. 设置环境变量

复用集群环境并创建独立命名空间，隔离本实验制造的各类故障 Pod，便于统一清理。

```bash
source /tmp/demo-eks.env
export DEMO_NS=health-demo
kubectl create namespace ${DEMO_NS} --dry-run=client -o yaml | kubectl apply -f -
```

### 2. 配置三种健康检查（Liveness / Readiness / Startup）

三种探针各司其职：startupProbe 保护慢启动应用、readinessProbe 控制流量接入、livenessProbe 触发自愈重启。理解三者的 `initialDelaySeconds`/`periodSeconds`/`failureThreshold` 是配置健康检查的关键。

```bash
cat > /tmp/health-demo.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: health-app
  namespace: health-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: health-app
  template:
    metadata:
      labels:
        app: health-app
    spec:
      containers:
        - name: app
          image: 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/nginx:1.27-alpine
          ports:
            - containerPort: 80
          startupProbe:
            httpGet:
              path: /
              port: 80
            failureThreshold: 30
            periodSeconds: 2
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 3
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "100m"
              memory: "128Mi"
EOF

kubectl apply -f /tmp/health-demo.yaml
kubectl -n health-demo rollout status deployment/health-app
kubectl -n health-demo describe pod -l app=health-app | grep -A5 "Liveness\|Readiness\|Startup"
```


> ⚠️ **首次拉取 048912060910 镜像若出现 ImagePullBackOff（403）**：这是节点 ECR credential provider 冷启动的短暂问题，无需修改任何权限配置。等待约 30 秒后执行 `kubectl rollout restart deployment/<name> -n <ns>` 即可恢复。节点一旦缓存镜像后，同节点后续 Pod 直接走本地缓存不再触发。

**预期输出**：Pod Running，三种探针均显示配置信息

### 3. 故障场景 1 — ImagePullBackOff

用一个不存在的镜像触发拉取失败。通过 `describe` 的 Events 可看到具体拉取错误，这是镜像名拼写错误或仓库无权限时最常见的现象。

```bash
kubectl run bad-image \
  --image=nonexistent-repo/no-such-image:v999 \
  -n ${DEMO_NS} \
  --restart=Never

sleep 30
kubectl get pod bad-image -n ${DEMO_NS}
kubectl describe pod bad-image -n ${DEMO_NS} | grep -A5 "Events:"

# 清理
kubectl delete pod bad-image -n ${DEMO_NS}
```

**预期输出**：Pod 状态为 `ImagePullBackOff`，Events 显示拉取失败原因

### 4. 故障场景 2 — CrashLoopBackOff

容器启动后立即 `exit 1`，Kubelet 反复重启并逐步拉长退避间隔。用 `logs --previous` 查看上一次崩溃的日志是定位崩溃根因的关键手法。

```bash
cat > /tmp/crash-pod.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: crash-pod
  namespace: health-demo
spec:
  containers:
    - name: crash
      image: 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/busybox:1.36
      command: ["sh", "-c", "exit 1"]
      resources:
        limits:
          cpu: "50m"
          memory: "32Mi"
EOF

kubectl apply -f /tmp/crash-pod.yaml
sleep 60
kubectl get pod crash-pod -n ${DEMO_NS}
kubectl logs crash-pod -n ${DEMO_NS} --previous || true

# 清理
kubectl delete -f /tmp/crash-pod.yaml
```

**预期输出**：Pod 状态为 `CrashLoopBackOff`

### 5. 故障场景 3 — OOMKilled

容器申请的内存超过 limit（64Mi）触发 OOM。`describe` 中 Last State 的 Reason 显示 `OOMKilled`，提示需调高 limit 或优化应用内存占用。

```bash
cat > /tmp/oom-pod.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: oom-pod
  namespace: health-demo
spec:
  containers:
    - name: memory-hog
      image: 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/python:3.11-slim
      command: ["python3", "-c", "x = bytearray(200 * 1024 * 1024)"]
      resources:
        limits:
          cpu: "100m"
          memory: "64Mi"
EOF

kubectl apply -f /tmp/oom-pod.yaml
sleep 30
kubectl get pod oom-pod -n ${DEMO_NS}
kubectl describe pod oom-pod -n ${DEMO_NS} | grep -E "OOMKilled|Last State|Reason"

# 清理
kubectl delete -f /tmp/oom-pod.yaml
```

**预期输出**：Pod 状态 `OOMKilled`，上次退出原因 `OOMKilled`

### 6. 故障场景 4 — Pending（资源不足）

请求 100 核 CPU、100Gi 内存远超节点容量，调度器无法分配节点，Pod 卡在 Pending。Events 中的 `Insufficient cpu` 是判断资源不足的直接依据。

```bash
cat > /tmp/pending-pod.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pending-pod
  namespace: health-demo
spec:
  containers:
    - name: big
      image: 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/nginx:1.27-alpine
      resources:
        requests:
          cpu: "100"
          memory: "100Gi"
EOF

kubectl apply -f /tmp/pending-pod.yaml
sleep 10
kubectl get pod pending-pod -n ${DEMO_NS}
kubectl describe pod pending-pod -n ${DEMO_NS} | grep -A10 "Events:"

# 清理
kubectl delete -f /tmp/pending-pod.yaml
```

**预期输出**：Pod 状态为 `Pending`，Events 显示 `Insufficient cpu`

### 7. 故障场景 5 — Service 无 Endpoints

Service 的 selector 匹配不到任何 Pod，导致 Endpoints 为空、流量无处转发。这是"服务能解析但连不通"类问题的典型根因，排查时优先检查 selector 与 Pod label 是否一致。

```bash
cat > /tmp/no-endpoint.yaml <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: no-match-svc
  namespace: health-demo
spec:
  type: ClusterIP
  ports:
    - port: 80
  selector:
    app: does-not-exist
EOF

kubectl apply -f /tmp/no-endpoint.yaml
kubectl get endpoints no-match-svc -n ${DEMO_NS}

# 清理
kubectl delete -f /tmp/no-endpoint.yaml
```

**预期输出**：`no-match-svc` Endpoints 列为空（`<none>`）

### 8. 使用 kubectl debug 排查（Ephemeral Container）

`kubectl debug` 向运行中的 Pod 注入临时容器，共享目标容器的网络命名空间，无需修改原 Pod 即可在容器内部执行诊断命令，适合排查无 shell 的精简镜像。

```bash
# 先确保 health-app pod 在运行
POD_NAME=$(kubectl get pod -n ${DEMO_NS} -l app=health-app -o jsonpath='{.items[0].metadata.name}')

kubectl debug -it ${POD_NAME} \
  --image=048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/busybox:1.36 \
  --target=app \
  -n ${DEMO_NS} \
  -- sh -c "wget -qO- http://localhost:80 && echo '连接成功'"
```

**预期输出**：返回 nginx 默认页面内容，打印"连接成功"

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 部署一个同时配置了 Liveness / Readiness / Startup 三种探针的应用
- [ ] 复现并识别 ImagePullBackOff / CrashLoopBackOff / OOMKilled / Pending 四种故障状态
- [ ] 通过 describe Events 和 logs --previous 定位故障根因
- [ ] 使用 kubectl debug 临时容器进入运行中 Pod 进行诊断

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `kubectl get deployment health-app -n health-demo -o jsonpath='{.status.readyReplicas}'` | `1` |
| 2 | `kubectl get pod -l app=health-app -n health-demo -o jsonpath='{.items[0].status.phase}'` | `Running` |
| 3 | `kubectl get pod -l app=health-app -n health-demo -o jsonpath='{.items[0].spec.containers[0].livenessProbe.httpGet.path}'` | `/` |

---

## 实验总结

本实验通过三种健康检查探针与五类典型故障的复现，建立了 EKS 工作负载排障的完整方法论：从 `get`/`describe` 观察状态与 Events，到 `logs --previous` 追溯崩溃，再到 `kubectl debug` 注入临时容器深入诊断。这套技能将贯穿后续所有 Demo 的问题定位。下一个 Demo（Demo05）将进入 Velero 的备份与灾难恢复。

---

## 清理

```bash
kubectl delete namespace health-demo
```
