# Demo17 — Pod Security Standards 工作负载安全

## 实验简介

本实验使用 Kubernetes 内置的 Pod Security Standards（PSS）与 Pod Security Admission，在命名空间级别强制工作负载的安全基线。通过对 baseline 与 restricted 两个级别的拒绝/放行测试，以及 warn 模式的演示，掌握如何在准入阶段拦截不安全的 Pod。

**实验目标：**
- 掌握用命名空间标签启用 PSS 的 enforce/warn 模式
- 理解 baseline 与 restricted 两个安全级别的差异
- 能够编写符合 restricted 级别的合规 Pod

**实验流程：**
1. 设置环境变量
2. 创建带不同 PSS 级别标签的命名空间
3. 测试 baseline 命名空间 — 特权容器被拒绝
4. 测试 restricted 命名空间 — 不合规 Pod 被拒绝
5. 部署符合 restricted 级别的 Pod
6. 验证安全上下文
7. 测试 Warn 模式（不阻止但警告）

**预计时长：** 15-20 分钟

## 前提条件

- **工具**：AWS CLI v2、kubectl
- **权限**：EKS 权限
- **前提**：Demo01 已完成
- **预计耗时**：15-20 分钟

---

## 步骤

### 1. 设置环境变量

复用集群环境，定义本 Demo 使用的命名空间前缀。

```bash
source /tmp/demo-eks.env
export PSS_NS=pss-demo
```

### 2. 创建带不同 PSS 级别标签的命名空间

创建 baseline 与 restricted 两个命名空间并打 enforce/warn 标签，由内置的 Pod Security Admission 在准入阶段执行。

```bash
# Baseline 级别（拒绝特权容器，允许大部分配置）
kubectl create namespace pss-baseline --dry-run=client -o yaml | \
  kubectl apply -f - 

kubectl label namespace pss-baseline \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=restricted \
  --overwrite

# Restricted 级别（最严格，要求 runAsNonRoot 等）
kubectl create namespace pss-restricted --dry-run=client -o yaml | \
  kubectl apply -f -

kubectl label namespace pss-restricted \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/warn=restricted \
  --overwrite

kubectl get namespace pss-baseline pss-restricted --show-labels
```

**预期输出**：两个命名空间显示 PSS 标签

### 3. 测试 baseline 命名空间 — 特权容器被拒绝

尝试创建特权容器，验证 baseline 级别会拒绝（baseline 主要防止明显的特权提升）。

```bash
echo "=== 测试 1：特权容器（在 baseline 中应被拒绝）==="

cat > /tmp/privileged-pod.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
  namespace: pss-baseline
spec:
  containers:
    - name: app
      image: 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/nginx:1.27-alpine
      securityContext:
        privileged: true
      resources:
        limits:
          cpu: "100m"
          memory: "64Mi"
EOF

kubectl apply -f /tmp/privileged-pod.yaml 2>&1 && echo "允许（意外）" || echo "✓ 已被 PSS 拒绝"
```

**预期输出**：Pod 创建被拒绝，提示违反 baseline 策略

### 4. 测试 restricted 命名空间 — 不合规 Pod 被拒绝

在 restricted 命名空间创建普通 Pod，因缺少 runAsNonRoot/seccompProfile 等被拒绝。

```bash
echo "=== 测试 2：普通 Pod（在 restricted 中被拒绝）==="

cat > /tmp/plain-pod.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: plain-pod
  namespace: pss-restricted
spec:
  containers:
    - name: app
      image: 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/nginx:1.27-alpine
      ports:
        - containerPort: 80
      resources:
        limits:
          cpu: "100m"
          memory: "64Mi"
EOF

kubectl apply -f /tmp/plain-pod.yaml 2>&1 && echo "允许（意外）" || echo "✓ 已被 PSS restricted 拒绝"
```

**预期输出**：Pod 被拒绝（缺少 `runAsNonRoot`、`seccompProfile` 等）

### 5. 部署符合 restricted 级别的 Pod

部署满足 restricted 全部要求（非 root、drop ALL、禁止提权、RuntimeDefault seccomp）的合规 Pod。

```bash
cat > /tmp/compliant-pod.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: compliant-pod
  namespace: pss-restricted
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 101
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/nginx:1.27-alpine
      ports:
        - containerPort: 8080
      securityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop:
            - ALL
        readOnlyRootFilesystem: false
      resources:
        requests:
          cpu: "50m"
          memory: "64Mi"
        limits:
          cpu: "100m"
          memory: "128Mi"
      volumeMounts:
        - name: tmp-dir
          mountPath: /tmp
        - name: cache-dir
          mountPath: /var/cache/nginx
        - name: run-dir
          mountPath: /var/run
  volumes:
    - name: tmp-dir
      emptyDir: {}
    - name: cache-dir
      emptyDir: {}
    - name: run-dir
      emptyDir: {}
EOF

kubectl apply -f /tmp/compliant-pod.yaml
kubectl wait pod compliant-pod -n pss-restricted --for=condition=Ready --timeout=60s
kubectl get pod compliant-pod -n pss-restricted
```

**预期输出**：compliant-pod 成功创建并运行

### 6. 验证安全上下文

检查合规 Pod 的 securityContext，确认各项安全约束均已生效。

```bash
kubectl get pod compliant-pod -n pss-restricted \
  -o jsonpath='{.spec.securityContext}' | python3 -m json.tool

kubectl get pod compliant-pod -n pss-restricted \
  -o jsonpath='{.spec.containers[0].securityContext}' | python3 -m json.tool
```

**预期输出**：显示 `runAsNonRoot: true`、`seccompProfile`、`allowPrivilegeEscalation: false`、`capabilities.drop: ALL`

### 7. 测试 Warn 模式（不阻止但警告）

用仅设 warn 的命名空间部署不合规 Pod，观察 kubectl 输出警告但不阻止创建，适合策略的渐进式收紧。

```bash
# 创建一个 warn-only 命名空间
kubectl create namespace pss-warn --dry-run=client -o yaml | kubectl apply -f -
kubectl label namespace pss-warn \
  pod-security.kubernetes.io/warn=restricted \
  --overwrite

# 部署不合规 Pod（不会被拒绝，但会有警告）
# 注意：新版 kubectl 已移除 --limits flag，直接运行即可观察 Warning 输出
kubectl run warn-pod \
  --image=048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/nginx:1.27-alpine \
  -n pss-warn \
  --restart=Never

echo "注意 kubectl 输出中的 Warning 信息"
kubectl get pod warn-pod -n pss-warn
kubectl delete pod warn-pod -n pss-warn
```

**预期输出**：Pod 创建成功，但 kubectl 输出显示 `Warning: would violate PodSecurity "restricted"`

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 用命名空间标签为 baseline 与 restricted 启用 PSS enforce
- [ ] 观察到特权容器在 baseline、普通 Pod 在 restricted 被拒绝
- [ ] 部署一个满足 restricted 全部要求的合规 Pod 并运行
- [ ] 在 warn 模式下看到警告但不阻止 Pod 创建

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `kubectl get namespace pss-restricted -o jsonpath='{.metadata.labels.pod-security\.kubernetes\.io/enforce}'` | `restricted` |
| 2 | `kubectl get namespace pss-baseline -o jsonpath='{.metadata.labels.pod-security\.kubernetes\.io/enforce}'` | `baseline` |
| 3 | `kubectl get pod compliant-pod -n pss-restricted -o jsonpath='{.status.phase}'` | `Running` |
| 4 | `kubectl get pod compliant-pod -n pss-restricted -o jsonpath='{.spec.securityContext.runAsNonRoot}'` | `true` |

---

## 实验总结

本实验用内置的 Pod Security Admission 在准入阶段强制了工作负载安全基线，演示了 baseline/restricted 的拒绝逻辑、合规 Pod 的写法，以及 warn 模式用于渐进式收紧。这是无需第三方组件的轻量级安全加固手段。下一个 Demo（Demo18）将进入 ADOT 与 OpenTelemetry 的统一可观测性。

## 清理

```bash
kubectl delete namespace pss-baseline pss-restricted pss-warn
```
