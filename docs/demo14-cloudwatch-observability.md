# Demo14 — CloudWatch Observability 原生可观测性

## 实验简介

本实验启用 AWS 原生的 CloudWatch Observability，实现对 EKS 的开箱即用可观测性：开启控制平面日志、安装 Container Insights addon（部署 CloudWatch Agent 与 Fluent Bit），自动采集指标与容器日志到 CloudWatch。相比自建 Prometheus，它无需维护监控组件，深度集成 AWS 控制台。

**实验目标：**
- 掌握 EKS 控制平面日志与 CloudWatch Observability addon 的启用
- 理解 Container Insights 的指标与日志采集链路
- 能够在 CloudWatch 中查看容器日志与性能指标

**实验流程：**
1. 设置环境变量
2. 启用 EKS 控制平面日志
3. 安装 CloudWatch Observability Addon
4. 验证 CloudWatch Agent 和 Fluent Bit 运行
5. 部署测试应用产生日志
6. 验证 CloudWatch 日志组
7. 查看 Container Insights 日志
8. 在 CloudWatch Console 查看 Container Insights

**预计时长：** 20-30 分钟

## 前提条件

- **工具**：AWS CLI v2、kubectl、eksctl
- **权限**：EKS、CloudWatch、IAM 权限
- **前提**：Demo01 已完成，Pod Identity Agent addon 已安装
- **预计耗时**：20-30 分钟

---

## 步骤

### 1. 设置环境变量

复用集群环境，定义 CloudWatch 命名空间与 Container Insights 日志组前缀。

```bash
source /tmp/demo-eks.env
export CW_NS=amazon-cloudwatch
export LOG_GROUP_PREFIX=/aws/containerinsights/${CLUSTER_NAME}
```

### 2. 启用 EKS 控制平面日志

开启 api/audit/authenticator 等控制平面日志并轮询集群回到 ACTIVE，便于审计与排障。

```bash
aws eks update-cluster-config \
  --name ${CLUSTER_NAME} \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}' \
  --region ${AWS_REGION}

# 等待更新完成
until [ "$(aws eks describe-cluster \
  --name ${CLUSTER_NAME} \
  --query 'cluster.status' --output text)" = "ACTIVE" ]; do
  echo "等待控制平面日志启用..."; sleep 15
done

echo "控制平面日志已启用"
```

**预期输出**：打印"控制平面日志已启用"

### 3. 安装 CloudWatch Observability Addon

**重要（中国区）**：addon 需要 Pod Identity 才能写入 CloudWatch。必须先创建 IAM Role、安装 addon，再为 addon 自动创建的 `cloudwatch-agent` ServiceAccount 绑定 Pod Identity Association，最后重启两个 DaemonSet 使新凭证生效。

```bash
# 3a. 创建 CloudWatch Agent IAM Role
cat > /tmp/cw-trust.json <<EOF
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"pods.eks.amazonaws.com"},"Action":["sts:AssumeRole","sts:TagSession"]}]}
EOF

aws iam create-role \
  --role-name eks-cw-observability-role \
  --assume-role-policy-document file:///tmp/cw-trust.json || true

aws iam attach-role-policy \
  --role-name eks-cw-observability-role \
  --policy-arn arn:aws-cn:iam::aws:policy/CloudWatchAgentServerPolicy

CW_ROLE_ARN=$(aws iam get-role --role-name eks-cw-observability-role --query 'Role.Arn' --output text)

# 3b. 安装 addon
aws eks create-addon \
  --cluster-name ${CLUSTER_NAME} \
  --addon-name amazon-cloudwatch-observability \
  --resolve-conflicts OVERWRITE \
  --configuration-values '{"containerLogs":{"enabled":true}}' \
  --region ${AWS_REGION}

aws eks wait addon-active \
  --cluster-name ${CLUSTER_NAME} \
  --addon-name amazon-cloudwatch-observability \
  --region ${AWS_REGION}

echo "CloudWatch Observability addon 已激活"

# 3c. 绑定 Pod Identity Association（addon 自动创建 cloudwatch-agent SA 但不会自动绑定 Role）
aws eks create-pod-identity-association \
  --cluster-name ${CLUSTER_NAME} \
  --namespace ${CW_NS} \
  --service-account cloudwatch-agent \
  --role-arn ${CW_ROLE_ARN} \
  --region ${AWS_REGION} || true

# 3d. 重启 DaemonSet 使新凭证生效
kubectl rollout restart daemonset/cloudwatch-agent -n ${CW_NS}
kubectl rollout restart daemonset/fluent-bit -n ${CW_NS}
kubectl rollout status daemonset/cloudwatch-agent -n ${CW_NS}
kubectl rollout status daemonset/fluent-bit -n ${CW_NS}
echo "DaemonSets 已使用新 Pod Identity 凭证重启"
```

**预期输出**：打印"CloudWatch Observability addon 已激活"与"DaemonSets 已使用新 Pod Identity 凭证重启"

### 4. 验证 CloudWatch Agent 和 Fluent Bit 运行

确认 amazon-cloudwatch 命名空间下两个 DaemonSet 已在所有节点运行。

```bash
kubectl get namespace ${CW_NS}
kubectl get pods -n ${CW_NS}
kubectl get daemonset -n ${CW_NS}
```

**预期输出**：`cloudwatch-agent` 和 `fluent-bit` DaemonSet 在所有节点上运行

### 5. 部署测试应用产生日志

部署持续输出心跳日志的应用，制造可被 Fluent Bit 采集的容器日志。

```bash
kubectl create namespace log-test --dry-run=client -o yaml | kubectl apply -f -

cat > /tmp/log-generator.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: log-generator
  namespace: log-test
spec:
  replicas: 2
  selector:
    matchLabels:
      app: log-gen
  template:
    metadata:
      labels:
        app: log-gen
    spec:
      containers:
        - name: logger
          image: 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/busybox:1.36
          command: ["sh", "-c",
            "while true; do echo \"[$(date -u +%Y-%m-%dT%H:%M:%SZ)] INFO: Application heartbeat - pod $(hostname)\"; sleep 5; done"]
          resources:
            requests:
              cpu: "10m"
              memory: "16Mi"
            limits:
              cpu: "50m"
              memory: "32Mi"
EOF

kubectl apply -f /tmp/log-generator.yaml
kubectl -n log-test rollout status deployment/log-generator
```


> ⚠️ **首次拉取 048912060910 镜像若出现 ImagePullBackOff（403）**：这是节点 ECR credential provider 冷启动的短暂问题，无需修改任何权限配置。等待约 30 秒后执行 `kubectl rollout restart deployment/<name> -n <ns>` 即可恢复。节点一旦缓存镜像后，同节点后续 Pod 直接走本地缓存不再触发。

**预期输出**：2 个 Pod Running

### 6. 验证 CloudWatch 日志组

等待日志上送后，确认 Container Insights 的 application/performance/dataplane 等日志组已创建。

```bash
echo "等待日志发送到 CloudWatch（轮询，通常 60 秒内出现）..."
for i in $(seq 1 18); do
  COUNT=$(aws logs describe-log-groups \
    --log-group-name-prefix "${LOG_GROUP_PREFIX}" \
    --region ${AWS_REGION} \
    --query 'length(logGroups)' --output text 2>/dev/null || echo 0)
  [ "${COUNT}" -gt 0 ] && echo "✅ 日志组已创建（${COUNT} 个）" && break
  echo "等待中（${i}/18）..."; sleep 10
done

aws logs describe-log-groups \
  --log-group-name-prefix "${LOG_GROUP_PREFIX}" \
  --region ${AWS_REGION} \
  --query 'logGroups[].logGroupName' \
  --output table
```

**预期输出**：显示多个日志组，包括 `/aws/containerinsights/demo/application`、`/performance`、`/dataplane` 等

### 7. 查看 Container Insights 日志

用 `aws logs tail` 查看应用日志，确认心跳信息已进入 CloudWatch。

```bash
# 查看应用日志
aws logs tail "${LOG_GROUP_PREFIX}/application" \
  --since 5m \
  --format short \
  --region ${AWS_REGION} | head -20

echo ""
echo "=== 节点性能指标日志 ==="
aws logs describe-log-streams \
  --log-group-name "${LOG_GROUP_PREFIX}/performance" \
  --region ${AWS_REGION} \
  --query 'logStreams[].logStreamName' \
  --output table 2>/dev/null | head -10 || echo "performance 日志组尚未创建"
```

**预期输出**：应用日志中包含 log-generator Pod 的心跳信息

### 8. 在 CloudWatch Console 查看 Container Insights

给出控制台查看路径，并通过 list-metrics 验证 ContainerInsights 指标已上送。

```bash
echo "============================================"
echo "Container Insights 查看路径："
echo "AWS 控制台 → CloudWatch → Container Insights"
echo "→ Performance monitoring → EKS Pods"
echo "集群名称: ${CLUSTER_NAME}"
echo "区域: ${AWS_REGION}"
echo "============================================"

# 验证 metrics 已发送
aws cloudwatch list-metrics \
  --namespace ContainerInsights \
  --dimensions Name=ClusterName,Value=${CLUSTER_NAME} \
  --region ${AWS_REGION} \
  --query 'Metrics[].MetricName' \
  --output text | tr '\t' '\n' | head -10
```

**预期输出**：列出 ContainerInsights 命名空间下的指标名称（如 `node_cpu_utilization` 等）

---

## 验收标准

完成本实验后，你应当能够：
- [ ] CloudWatch Observability addon 处于 ACTIVE 状态
- [ ] EKS 控制平面日志已启用
- [ ] CloudWatch Agent 与 Fluent Bit DaemonSet 在所有节点运行
- [ ] Container Insights 日志组已创建且能查到应用日志与指标

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws eks describe-addon --cluster-name demo --addon-name amazon-cloudwatch-observability --region cn-northwest-1 --query 'addon.status' --output text` | `ACTIVE` |
| 2 | `aws eks describe-cluster --name demo --region cn-northwest-1 --query 'cluster.logging.clusterLogging[0].enabled' --output text` | `True` |
| 3 | `kubectl get daemonset cloudwatch-agent -n amazon-cloudwatch -o jsonpath='{.status.numberReady}'` | 等于节点总数（如 `2` 或 `3`） |
| 4 | `aws logs describe-log-groups --log-group-name-prefix /aws/containerinsights/demo --region cn-northwest-1 --query 'length(logGroups)' --output text` | 大于 `0` |

---

## 实验总结

本实验启用了 AWS 原生 CloudWatch Observability，完成控制平面日志、Container Insights 指标与容器日志的端到端采集，并在 CloudWatch 中查看了结果。与 Demo13 的自建方案相比，它以零运维换取与 AWS 生态的深度集成。下一个 Demo（Demo15）将进入 Spot 节点与 Fargate 的计算形态管理。

## 清理

```bash
source /tmp/demo-eks.env

kubectl delete namespace log-test

aws eks delete-addon \
  --cluster-name ${CLUSTER_NAME} \
  --addon-name amazon-cloudwatch-observability \
  --region ${AWS_REGION} || true

# 关闭控制平面日志
aws eks update-cluster-config \
  --name ${CLUSTER_NAME} \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":false}]}' \
  --region ${AWS_REGION}

# 删除 CloudWatch 日志组（可选，日志按量计费）
for LOG_GROUP in $(aws logs describe-log-groups \
  --log-group-name-prefix "${LOG_GROUP_PREFIX}" \
  --region ${AWS_REGION} \
  --query 'logGroups[].logGroupName' --output text); do
  aws logs delete-log-group --log-group-name ${LOG_GROUP} --region ${AWS_REGION}
  echo "已删除日志组: ${LOG_GROUP}"
done
```
