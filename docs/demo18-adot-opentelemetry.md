# Demo18 — ADOT 与 OpenTelemetry 可观测性

## 实验简介

本实验部署 AWS Distro for OpenTelemetry（ADOT）Collector，以 OpenTelemetry 标准统一采集追踪与指标，并分别导出到 X-Ray 和 CloudWatch。示例应用通过 OTLP 协议上报遥测数据，演示云原生应用的端到端可观测性。Collector 与示例应用镜像统一从集中预置的 0910 仓库拉取。

**实验目标：**
- 掌握 ADOT Collector 的 Pod Identity 授权与部署
- 理解 OTLP 接收、batch/resource 处理、awsxray/awsemf 导出的管道结构
- 能够让应用经 OTLP 上报并在 X-Ray/CloudWatch 验证遥测数据

**实验流程：**
1. 设置环境变量
2. 创建 ADOT Collector IAM Policy 和 Role
3. 创建 Service Account 和 Pod Identity Association
4. 创建 ADOT Collector ConfigMap
5. 部署 ADOT Collector Deployment
6. 部署 Demo 应用（发送遥测数据）
7. 生成流量并验证追踪

**预计 AI 执行时长：** 30-40 分钟

## 前提条件

- **工具**：AWS CLI v2、kubectl、eksctl
- **权限**：EKS、CloudWatch、X-Ray、IAM 权限
- **前提**：Demo01 已完成，Pod Identity Agent addon 已安装

---

## 步骤

### 1. 设置环境变量

复用集群环境，定义 ADOT 命名空间、Collector Role 名与示例应用命名空间。

```bash
source /tmp/demo-eks.env
export ADOT_NS=adot-demo
export ADOT_ROLE=eks-adot-collector-role
export DEMO_APP_NS=otel-app

kubectl create namespace ${ADOT_NS} --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace ${DEMO_APP_NS} --dry-run=client -o yaml | kubectl apply -f -
```

### 2. 创建 ADOT Collector IAM Policy 和 Role

创建包含 CloudWatch、X-Ray、Logs 写入权限的 Policy，并绑定到信任 `pods.eks.amazonaws.com` 的 Pod Identity Role。

```bash
cat > /tmp/adot-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "cloudwatch:PutMetricData",
        "ec2:DescribeVolumes",
        "ec2:DescribeTags",
        "logs:PutLogEvents",
        "logs:DescribeLogStreams",
        "logs:DescribeLogGroups",
        "logs:CreateLogStream",
        "logs:CreateLogGroup",
        "xray:PutTraceSegments",
        "xray:PutTelemetryRecords",
        "xray:GetSamplingRules",
        "xray:GetSamplingTargets",
        "xray:GetSamplingStatisticSummaries",
        "ssm:GetParameters"
      ],
      "Resource": "*"
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name ADOTCollectorPolicy \
  --policy-document file:///tmp/adot-policy.json || true

export ADOT_POLICY_ARN=$(aws iam list-policies \
  --query "Policies[?PolicyName=='ADOTCollectorPolicy'].Arn" \
  --output text)

# 创建 Pod Identity Role
cat > /tmp/adot-trust.json <<EOF
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
  --role-name ${ADOT_ROLE} \
  --assume-role-policy-document file:///tmp/adot-trust.json || true

aws iam attach-role-policy \
  --role-name ${ADOT_ROLE} \
  --policy-arn ${ADOT_POLICY_ARN}

export ADOT_ROLE_ARN=$(aws iam get-role \
  --role-name ${ADOT_ROLE} \
  --query 'Role.Arn' --output text)

echo "ADOT Role ARN: ${ADOT_ROLE_ARN}"
```

**预期输出**：ADOT Role ARN 输出

### 3. 创建 Service Account 和 Pod Identity Association

创建 adot-collector ServiceAccount 并与 Role 建立 Pod Identity 关联，使 Collector 能写入 X-Ray/CloudWatch。

```bash
kubectl create serviceaccount adot-collector -n ${ADOT_NS} \
  --dry-run=client -o yaml | kubectl apply -f -

aws eks create-pod-identity-association \
  --cluster-name ${CLUSTER_NAME} \
  --namespace ${ADOT_NS} \
  --service-account adot-collector \
  --role-arn ${ADOT_ROLE_ARN} \
  --region ${AWS_REGION} || true

echo "Pod Identity Association 已创建"
```

**预期输出**：打印"Pod Identity Association 已创建"

### 4. 创建 ADOT Collector ConfigMap

定义 Collector 管道：OTLP 接收器 + batch/resource 处理器 + awsxray/awsemf 导出器，把追踪发往 X-Ray、指标发往 CloudWatch EMF。

```bash
cat > /tmp/adot-config.yaml <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: adot-collector-config
  namespace: ${ADOT_NS}
data:
  config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318

    processors:
      batch:
        timeout: 5s
        send_batch_size: 50
      resource:
        attributes:
          - key: aws.region
            value: ${AWS_REGION}
            action: insert
          - key: k8s.cluster.name
            value: ${CLUSTER_NAME}
            action: insert

    exporters:
      awsxray:
        region: ${AWS_REGION}
        index_all_attributes: true
      awsemf:
        region: ${AWS_REGION}
        namespace: EKS/OTelDemo
        log_group_name: /aws/eks/${CLUSTER_NAME}/otel-metrics
        log_stream_name: adot-metrics
        resource_to_telemetry_conversion:
          enabled: true

    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch, resource]
          exporters: [awsxray]
        metrics:
          receivers: [otlp]
          processors: [batch, resource]
          exporters: [awsemf]
EOF

kubectl apply -f /tmp/adot-config.yaml
```

**预期输出**：ConfigMap 创建成功

### 5. 部署 ADOT Collector Deployment

部署 Collector 及其 Service，暴露 OTLP 4317/4318 端口供应用上报遥测数据。Collector 镜像已集中预置在 0910 仓库（`ecrpublic/aws-observability/aws-otel-collector:v0.43.3`）。

```bash
cat > /tmp/adot-deployment.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: adot-collector
  namespace: ${ADOT_NS}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: adot-collector
  template:
    metadata:
      labels:
        app: adot-collector
    spec:
      serviceAccountName: adot-collector
      containers:
        - name: adot-collector
          image: 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/aws-observability/aws-otel-collector:v0.43.3
          args: ["--config=/conf/config.yaml"]
          ports:
            - containerPort: 4317
              name: otlp-grpc
            - containerPort: 4318
              name: otlp-http
          volumeMounts:
            - name: config
              mountPath: /conf
          resources:
            requests:
              cpu: "200m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
      volumes:
        - name: config
          configMap:
            name: adot-collector-config
            items:
              - key: config.yaml
                path: config.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: adot-collector
  namespace: ${ADOT_NS}
spec:
  type: ClusterIP
  ports:
    - name: otlp-grpc
      port: 4317
      targetPort: 4317
    - name: otlp-http
      port: 4318
      targetPort: 4318
  selector:
    app: adot-collector
EOF

kubectl apply -f /tmp/adot-deployment.yaml
kubectl -n ${ADOT_NS} rollout status deployment/adot-collector
```


> ⚠️ **首次拉取 048912060910 镜像若出现 ImagePullBackOff（403）**：这是节点 ECR credential provider 冷启动的短暂问题，无需修改任何权限配置。等待约 30 秒后执行 `kubectl rollout restart deployment/<name> -n <ns>` 即可恢复。节点一旦缓存镜像后，同节点后续 Pod 直接走本地缓存不再触发。

**预期输出**：`deployment "adot-collector" successfully rolled out`

### 6. 部署 Demo 应用（发送遥测数据）

部署示例 Java 应用（镜像已预置在 0910 仓库 `ecrpublic/aws-otel-test/aws-otel-java-spark:1.17.0`），通过环境变量 OTEL_EXPORTER_OTLP_ENDPOINT 指向 Collector，使其自动上报追踪与指标。

```bash
ADOT_ENDPOINT="adot-collector.${ADOT_NS}.svc.cluster.local"

cat > /tmp/demo-app.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
  namespace: ${DEMO_APP_NS}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels:
        app: demo-app
    spec:
      containers:
        - name: app
          image: 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/aws-otel-test/aws-otel-java-spark:1.17.0
          env:
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://${ADOT_ENDPOINT}:4317"
            - name: OTEL_RESOURCE_ATTRIBUTES
              value: "service.name=eks-demo-app,service.version=1.0"
          ports:
            - containerPort: 4567
          resources:
            requests:
              cpu: "200m"
              memory: "512Mi"
            limits:
              cpu: "500m"
              memory: "1Gi"
EOF

kubectl apply -f /tmp/demo-app.yaml
kubectl -n ${DEMO_APP_NS} rollout status deployment/demo-app
```

**预期输出**：Demo App 就绪

### 7. 生成流量并验证追踪

通过 port-forward 产生请求触发遥测，再从 X-Ray 与 CloudWatch EKS/OTelDemo 命名空间确认追踪与指标已上报。

```bash
DEMO_POD=$(kubectl get pod -n ${DEMO_APP_NS} -l app=demo-app \
  -o jsonpath='{.items[0].metadata.name}')

# 通过 port-forward 访问应用
kubectl port-forward -n ${DEMO_APP_NS} pod/${DEMO_POD} 4567:4567 &
sleep 5

for i in $(seq 1 10); do
  curl -s http://localhost:4567/ > /dev/null 2>&1 || true
done

kill %1 2>/dev/null || true

echo "等待遥测数据发送（约 1 分钟）..."
sleep 60

# 验证 X-Ray 追踪
aws xray get-trace-summaries \
  --start-time $(date -d '5 minutes ago' +%s) \
  --end-time $(date +%s) \
  --region ${AWS_REGION} \
  --query 'TraceSummaries[0].Id' --output text 2>/dev/null || echo "X-Ray 追踪需在控制台查看"

# 验证 CloudWatch 指标
aws cloudwatch list-metrics \
  --namespace EKS/OTelDemo \
  --region ${AWS_REGION} \
  --query 'Metrics[*].MetricName' --output text | head -5
```

**预期输出**：X-Ray 有追踪数据，CloudWatch `EKS/OTelDemo` 命名空间有指标

---

## 验收标准

完成本实验后，你应当能够：
- [ ] ADOT Collector 在 adot-demo 命名空间正常运行
- [ ] adot-collector ServiceAccount 已配置有效的 Pod Identity Association
- [ ] Collector ConfigMap 中包含 awsxray/awsemf 导出器配置
- [ ] 生成流量后在 X-Ray 看到追踪、在 CloudWatch EKS/OTelDemo 看到指标

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `kubectl get deployment adot-collector -n adot-demo -o jsonpath='{.status.readyReplicas}'` | `1` |
| 2 | `aws eks list-pod-identity-associations --cluster-name demo --namespace adot-demo --region cn-northwest-1 --query 'associations[?serviceAccount==\`adot-collector\`].associationId' --output text \| wc -w` | `1` |
| 3 | `kubectl get configmap adot-collector-config -n adot-demo -o jsonpath='{.data.config\.yaml}' \| grep -c awsxray` | 大于 `0`（awsxray 在 ConfigMap 中出现 2 次：定义 + 引用） |
| 4 | `aws cloudwatch list-metrics --namespace EKS/OTelDemo --region cn-northwest-1 --query 'length(Metrics)' --output text` | 大于 `0`（流量生成后） |

---

## 实验总结

本实验用 ADOT Collector 以 OpenTelemetry 标准统一采集应用追踪与指标，并导出到 X-Ray 与 CloudWatch，打通了云原生应用的端到端可观测性。结合 Demo13/14 的集群级监控，你已掌握从基础设施到应用层的完整观测能力。至此 18 个 Demo 覆盖了中国区 EKS 从建集群、应用发布、存储、伸缩、安全到可观测性的全栈实践。完成后请记得执行各 Demo 的清理步骤并删除集群以免产生费用。

## 清理

```bash
source /tmp/demo-eks.env

kubectl delete namespace ${DEMO_APP_NS} ${ADOT_NS}

aws eks delete-pod-identity-association \
  --cluster-name ${CLUSTER_NAME} \
  --association-id $(aws eks list-pod-identity-associations \
    --cluster-name ${CLUSTER_NAME} \
    --namespace adot-demo \
    --service-account adot-collector \
    --query 'associations[0].associationId' --output text) \
  --region ${AWS_REGION} || true

aws iam detach-role-policy \
  --role-name ${ADOT_ROLE} \
  --policy-arn ${ADOT_POLICY_ARN}
aws iam delete-role --role-name ${ADOT_ROLE}
aws iam delete-policy --policy-arn ${ADOT_POLICY_ARN}

aws logs delete-log-group \
  --log-group-name /aws/eks/${CLUSTER_NAME}/otel-metrics \
  --region ${AWS_REGION} || true
```
