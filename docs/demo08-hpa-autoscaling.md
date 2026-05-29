# Demo08 — HPA 水平自动伸缩

## 实验简介

本实验配置 Horizontal Pod Autoscaler（HPA），根据 CPU 利用率自动增减 Pod 副本数。HPA 依赖 Metrics Server 提供资源指标，本实验通过持续压测触发扩容、停止负载观察缩容，完整演示应用层弹性伸缩。压测应用使用中国区可访问的 `public.ecr.aws` PHP 镜像替代被屏蔽的 `registry.k8s.io/hpa-example`。

**实验目标：**
- 掌握 Metrics Server addon 安装与 HPA 配置
- 理解 HPA 基于 CPU 利用率目标值的扩缩容决策逻辑
- 能够通过压测观察 Pod 的 Scale Out 与 Scale In

**实验流程：**
1. 设置环境变量与命名空间
2. 安装 Metrics Server addon
3. 部署 CPU 压测应用
4. 配置 HPA
5. 产生负载触发 Scale Out
6. 停止负载观察 Scale In

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

复用集群环境并创建独立命名空间承载压测应用与 HPA。

```bash
source /tmp/demo-eks.env
export DEMO_NS=hpa-demo
kubectl create namespace ${DEMO_NS} --dry-run=client -o yaml | kubectl apply -f -
```

### 2. 安装 Metrics Server（EKS Managed Addon）

HPA 需要 Metrics Server 提供 Pod/节点的实时 CPU、内存数据。用 EKS 托管 addon 安装，等待 ACTIVE 后 `kubectl top` 能查到数据才算就绪。

```bash
aws eks create-addon \
  --cluster-name ${CLUSTER_NAME} \
  --addon-name metrics-server \
  --region ${AWS_REGION} || true

aws eks wait addon-active \
  --cluster-name ${CLUSTER_NAME} \
  --addon-name metrics-server \
  --region ${AWS_REGION}

# 验证 metrics 可用
sleep 30
kubectl top nodes
```

**预期输出**：节点 CPU/内存使用率数据正常显示

### 3. 部署负载测试应用

使用 nginx + PHP 应用进行 CPU 压测（替代 `registry.k8s.io/hpa-example`，使用可访问镜像）：

> ⚠️ 中国区屏蔽 `registry.k8s.io`，因此用 `048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/php:8.2-apache` 加 ConfigMap 注入计算密集脚本来制造 CPU 负载。

```bash
cat > /tmp/php-apache.yaml <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: php-app
  namespace: hpa-demo
data:
  index.php: |
    <?php
      $x = 0.0001;
      for ($i = 0; $i <= 1000000; $i++) {
        $x += sqrt($x);
      }
      echo "OK!";
    ?>
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
  namespace: hpa-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: php-apache
  template:
    metadata:
      labels:
        app: php-apache
    spec:
      containers:
        - name: php-apache
          image: 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/php:8.2-apache
          ports:
            - containerPort: 80
          volumeMounts:
            - name: php-app
              mountPath: /var/www/html
          resources:
            requests:
              cpu: "200m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
      volumes:
        - name: php-app
          configMap:
            name: php-app
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
  namespace: hpa-demo
spec:
  type: ClusterIP
  ports:
    - port: 80
  selector:
    app: php-apache
EOF

kubectl apply -f /tmp/php-apache.yaml
kubectl -n ${DEMO_NS} rollout status deployment/php-apache
```


> ⚠️ **首次拉取 048912060910 镜像若出现 ImagePullBackOff（403）**：这是节点 ECR credential provider 冷启动的短暂问题，无需修改任何权限配置。等待约 30 秒后执行 `kubectl rollout restart deployment/<name> -n <ns>` 即可恢复。节点一旦缓存镜像后，同节点后续 Pod 直接走本地缓存不再触发。

**预期输出**：Deployment 就绪，1 个 Pod Running

### 4. 配置 HPA

创建 autoscaling/v2 的 HPA，目标为 CPU 平均利用率 50%，副本范围 1-5。HPA 控制器会周期性对比实际利用率与目标值，计算所需副本数。

```bash
cat > /tmp/hpa.yaml <<'EOF'
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
  namespace: hpa-demo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 60
EOF

kubectl apply -f /tmp/hpa.yaml
kubectl get hpa php-apache -n ${DEMO_NS}
```

**预期输出**：HPA 创建成功，TARGETS 显示当前 CPU 利用率

### 5. 产生负载触发 Scale Out

用一个不断 curl 压测应用的 load-generator Pod 制造持续 CPU 负载。HPA 检测到利用率超过 50% 后，约 3-5 分钟内逐步扩容副本，循环打印观察过程。

```bash
# 在后台产生持续 HTTP 请求
cat > /tmp/load-gen.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: load-generator
  namespace: hpa-demo
spec:
  containers:
    - name: load-gen
      image: 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/amazonlinux/amazonlinux:2023
      command: ["sh", "-c",
        "while true; do curl -s http://php-apache.hpa-demo.svc.cluster.local/ > /dev/null 2>&1; done"]
      resources:
        requests:
          cpu: "50m"
          memory: "32Mi"
        limits:
          cpu: "100m"
          memory: "64Mi"
EOF

kubectl apply -f /tmp/load-gen.yaml

echo "等待 HPA 触发扩容（约 3-5 分钟）..."
for i in $(seq 1 20); do
  echo "=== $(date -u +%H:%M:%S) ==="
  kubectl get hpa php-apache -n ${DEMO_NS}
  kubectl get pods -n ${DEMO_NS} -l app=php-apache
  sleep 30
done
```

**预期输出**：HPA 扩容，Pod 数量从 1 增加到 2-5 个

### 6. 停止负载观察 Scale In

删除压测 Pod 后 CPU 回落，HPA 在默认的稳定窗口（约 5 分钟）后才开始缩容，避免抖动。观察副本数逐步回到 minReplicas=1。

```bash
kubectl delete pod load-generator -n ${DEMO_NS}

echo "等待 HPA 缩容（约 1-2 分钟，stabilizationWindowSeconds=60）..."
for i in $(seq 1 10); do
  echo "=== $(date -u +%H:%M:%S) ==="
  kubectl get hpa php-apache -n ${DEMO_NS}
  REPLICAS=$(kubectl get deployment php-apache -n ${DEMO_NS} -o jsonpath='{.status.readyReplicas}' 2>/dev/null)
  kubectl get pods -n ${DEMO_NS} -l app=php-apache
  [ "${REPLICAS}" = "1" ] && echo "✅ 已缩回 1 副本" && break
  sleep 30
done
```

**预期输出**：负载停止后，Pod 数量逐渐降至 1（minReplicas）

---

## 验收标准

完成本实验后，你应当能够：
- [ ] Metrics Server addon 处于 ACTIVE 且 kubectl top 能返回数据
- [ ] 配置一个 CPU 目标 50%、副本范围 1-5 的 HPA
- [ ] 通过压测观察到 Pod 副本数从 1 扩容到多个
- [ ] 停止压测后观察到 Pod 副本数回缩到 minReplicas

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws eks describe-addon --cluster-name demo --addon-name metrics-server --region cn-northwest-1 --query 'addon.status' --output text` | `ACTIVE` |
| 2 | `kubectl get hpa php-apache -n hpa-demo -o jsonpath='{.spec.minReplicas}'` | `1` |
| 3 | `kubectl get hpa php-apache -n hpa-demo -o jsonpath='{.spec.maxReplicas}'` | `5` |
| 4 | `kubectl top nodes --no-headers \| wc -l` | 大于 `0`（metrics server 正常工作） |

---

## 实验总结

本实验通过 Metrics Server + HPA 实现了应用层（Pod 副本）的自动弹性，完整观察了压测驱动的 Scale Out 与负载消退后的 Scale In。HPA 解决的是「Pod 不够」的问题，但当节点资源耗尽时还需节点层伸缩。下一个 Demo（Demo09）将引入 Karpenter 实现节点级自动伸缩。

---

## 清理

```bash
kubectl delete namespace hpa-demo

# 如不再使用 metrics-server 可以删除 addon
# aws eks delete-addon --cluster-name demo --addon-name metrics-server --region cn-northwest-1
```
