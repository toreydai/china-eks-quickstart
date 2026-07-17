# Demo13 — Prometheus 与 Grafana 集群监控

## 实验简介

本实验在中国区 EKS 上部署 Prometheus + Grafana 监控栈：Prometheus 采集集群指标，Grafana 可视化展示。两者镜像统一从集中预置的 0910 仓库拉取；服务通过 NodePort 暴露并配合安全组放行实现外部访问。

**实验目标：**
- 掌握用 Helm 部署 Prometheus 与 Grafana 并对接数据源
- 理解 NodePort + 安全组在中国区暴露监控面板的方式
- 能够验证指标采集并导入社区 Dashboard

**实验流程：**
1. 设置环境变量
2. 安装 Prometheus
3. 配置 EC2 安全组允许 Prometheus 端口访问
4. 验证 Prometheus 采集数据
5. 安装 Grafana
6. 导入 Dashboard

**预计 AI 执行时长：** 25-35 分钟

## 前提条件

- **工具**：AWS CLI v2、kubectl、helm
- **权限**：EKS、EC2 权限
- **前提**：Demo01 已完成

---

## 步骤

### 1. 设置环境变量

复用集群环境，定义监控命名空间以及 Prometheus/Grafana 的 Helm chart 版本号。

```bash
source /tmp/demo-eks.env
export MONITORING_NS=monitoring
export PROMETHEUS_VERSION=29.8.0
export GRAFANA_VERSION=10.5.15
export ASSET_DIR=${ASSET_DIR:-$(pwd)/offline-assets}

kubectl create namespace ${MONITORING_NS} --dry-run=client -o yaml | kubectl apply -f -
```

### 2. 安装 Prometheus

Prometheus/Grafana 镜像已集中预置在 0910 仓库（`dockerhub/prom/prometheus:v3.9.1`、`dockerhub/grafana/grafana:12.3.1`），无需推送。用 Helm 安装精简版 Prometheus（关闭 alertmanager、pushgateway 等组件），以 NodePort 30090 暴露便于外部访问。

```bash
cp ${ASSET_DIR}/prometheus-${PROMETHEUS_VERSION}.tgz /tmp/

helm upgrade --install prometheus /tmp/prometheus-${PROMETHEUS_VERSION}.tgz \
  -n ${MONITORING_NS} \
  --set server.image.repository=048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/dockerhub/prom/prometheus \
  --set server.image.tag=v3.9.1 \
  --set server.persistentVolume.enabled=false \
  --set alertmanager.enabled=false \
  --set prometheus-pushgateway.enabled=false \
  --set kube-state-metrics.enabled=false \
  --set prometheus-node-exporter.enabled=false \
  --set configmapReload.prometheus.enabled=false \
  --set server.service.type=NodePort \
  --set server.service.nodePort=30090

kubectl -n ${MONITORING_NS} rollout status deployment/prometheus-server
```


> ⚠️ **首次拉取 048912060910 镜像若出现 ImagePullBackOff（403）**：这是节点 ECR credential provider 冷启动的短暂问题，无需修改任何权限配置。等待约 30 秒后执行 `kubectl rollout restart deployment/<name> -n <ns>` 即可恢复。节点一旦缓存镜像后，同节点后续 Pod 直接走本地缓存不再触发。

**预期输出**：`deployment "prometheus-server" successfully rolled out`

### 3. 配置 EC2 安全组允许 Prometheus 端口访问

放行当前公网 IP 对节点 30090 端口的访问，并打印由节点公有 IP 组成的访问地址。如果节点没有公网 IP，则使用 `kubectl port-forward` 作为兜底方式。

```bash
CLUSTER_SG=$(aws eks describe-cluster \
  --name ${CLUSTER_NAME} \
  --query 'cluster.resourcesVpcConfig.clusterSecurityGroupId' --output text)

# 获取当前 IP；如外部探测不可用，则跳过安全组放行，后续使用 port-forward
MY_IP=$(curl -s https://checkip.amazonaws.com/ 2>/dev/null || true)

if [ -n "${MY_IP}" ]; then
  aws ec2 authorize-security-group-ingress \
    --group-id ${CLUSTER_SG} \
    --protocol tcp \
    --port 30090 \
    --cidr ${MY_IP}/32 \
    --region ${AWS_REGION} 2>/dev/null || true
fi

# 获取一个节点的公有 IP
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="ExternalIP")].address}')
if [ -n "${NODE_IP}" ]; then
  echo "Prometheus URL: http://${NODE_IP}:30090"
else
  echo "节点没有公网 IP，使用 port-forward: kubectl -n ${MONITORING_NS} port-forward svc/prometheus-server 30090:80"
fi
```

**预期输出**：打印 Prometheus 访问地址

### 4. 验证 Prometheus 采集数据

调用 Prometheus 的 `up` 查询接口，确认已采集到监控目标。

```bash
if [ -n "${NODE_IP}" ]; then
  PROM_URL="http://${NODE_IP}:30090"
else
  kubectl -n ${MONITORING_NS} port-forward svc/prometheus-server 30090:80 >/tmp/prom-port-forward.log 2>&1 &
  PF_PID=$!
  sleep 5
  PROM_URL="http://127.0.0.1:30090"
fi

curl -s "${PROM_URL}/api/v1/query?query=up" | \
  python3 -c "import json,sys; d=json.load(sys.stdin); print(f'监控目标数: {len(d[\"data\"][\"result\"])}')"

[ -n "${PF_PID:-}" ] && kill ${PF_PID}
```

**预期输出**：打印监控目标数量（大于 0）

### 5. 安装 Grafana

用 Helm 安装 Grafana，预置 Prometheus 数据源，以 NodePort 30300 暴露并放行安全组。

```bash
cp ${ASSET_DIR}/grafana-${GRAFANA_VERSION}.tgz /tmp/

helm upgrade --install grafana /tmp/grafana-${GRAFANA_VERSION}.tgz \
  -n ${MONITORING_NS} \
  --set image.registry=048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn \
  --set image.repository=dockerhub/grafana/grafana \
  --set image.tag=12.3.1 \
  --set adminPassword=admin123 \
  --set service.type=NodePort \
  --set service.nodePort=30300 \
  --set persistence.enabled=false \
  --set datasources."datasources\.yaml".apiVersion=1 \
  --set datasources."datasources\.yaml".datasources[0].name=Prometheus \
  --set datasources."datasources\.yaml".datasources[0].type=prometheus \
  --set datasources."datasources\.yaml".datasources[0].url="http://prometheus-server.${MONITORING_NS}.svc.cluster.local" \
  --set datasources."datasources\.yaml".datasources[0].isDefault=true

kubectl -n ${MONITORING_NS} rollout status deployment/grafana

if [ -n "${MY_IP}" ]; then
  aws ec2 authorize-security-group-ingress \
    --group-id ${CLUSTER_SG} \
    --protocol tcp \
    --port 30300 \
    --cidr ${MY_IP}/32 \
    --region ${AWS_REGION} 2>/dev/null || true
fi

if [ -n "${NODE_IP}" ]; then
  echo "Grafana URL: http://${NODE_IP}:30300 (admin/admin123)"
else
  echo "节点没有公网 IP，使用 port-forward: kubectl -n ${MONITORING_NS} port-forward svc/grafana 30300:80"
fi
```

**预期输出**：Grafana 就绪，打印访问地址和凭据

### 6. 导入 Dashboard

通过 Grafana API 导入社区 Dashboard（3119/6417）展示集群与 Pod 监控，必要时在 UI 手动导入。

```bash
if [ -n "${NODE_IP}" ]; then
  GRAFANA_URL="http://admin:admin123@${NODE_IP}:30300"
else
  kubectl -n ${MONITORING_NS} port-forward svc/grafana 30300:80 >/tmp/grafana-port-forward.log 2>&1 &
  GRAFANA_PF_PID=$!
  sleep 5
  GRAFANA_URL="http://admin:admin123@127.0.0.1:30300"
fi

# Dashboard 3119 (Kubernetes cluster monitoring) 和 6417 (Kubernetes Pod Monitoring)
# 通过 Grafana API 导入
for DASHBOARD_ID in 3119 6417; do
  echo "导入 Dashboard ID: ${DASHBOARD_ID}"
  curl -s -X POST \
    "${GRAFANA_URL}/api/dashboards/import" \
    -H "Content-Type: application/json" \
    -d "{\"folderId\":0,\"overwrite\":true,\"inputs\":[{\"name\":\"DS_PROMETHEUS\",\"type\":\"datasource\",\"pluginId\":\"prometheus\",\"value\":\"Prometheus\"}],\"dashboard\":{\"id\":null,\"uid\":null,\"title\":\"Dashboard ${DASHBOARD_ID}\"}}" \
    2>/dev/null || echo "Dashboard ${DASHBOARD_ID} 需要手动在 UI 导入"
done

if [ -n "${NODE_IP}" ]; then
  echo "请访问 http://${NODE_IP}:30300 → Dashboards → Import → ID: 3119 或 6417"
else
  echo "请访问 http://127.0.0.1:30300 → Dashboards → Import → ID: 3119 或 6417"
  kill ${GRAFANA_PF_PID}
fi
```

**预期输出**：提示手动导入 Dashboard 的地址和 ID

---

## 验收标准

完成本实验后，你应当能够：
- [ ] Prometheus 与 Grafana 均在 monitoring 命名空间正常运行
- [ ] Prometheus 通过 NodePort 暴露且能查询到监控目标
- [ ] Grafana 通过 NodePort 30300 暴露并已对接 Prometheus 数据源
- [ ] 导入社区 Dashboard 展示集群与 Pod 监控数据

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `kubectl get deployment prometheus-server -n monitoring -o jsonpath='{.status.readyReplicas}'` | `1` |
| 2 | `kubectl get deployment grafana -n monitoring -o jsonpath='{.status.readyReplicas}'` | `1` |
| 3 | `kubectl get service prometheus-server -n monitoring -o jsonpath='{.spec.type}'` | `NodePort` |
| 4 | `kubectl get service grafana -n monitoring -o jsonpath='{.spec.ports[0].nodePort}'` | `30300` |

---

## 实验总结

本实验用 Prometheus + Grafana 搭建了开源监控栈，完成了指标采集、数据源对接与 Dashboard 展示，并实践了从集中预置的 0910 仓库拉取镜像与 NodePort+安全组的外部暴露方式。这是 EKS 自建可观测性的经典方案；下一个 Demo（Demo14）将转向 AWS 原生的 CloudWatch Container Insights 可观测性。

## 清理

```bash
source /tmp/demo-eks.env

helm uninstall grafana -n ${MONITORING_NS}
helm uninstall prometheus -n ${MONITORING_NS}
kubectl delete namespace ${MONITORING_NS}

CLUSTER_SG=$(aws eks describe-cluster \
  --name ${CLUSTER_NAME} \
  --query 'cluster.resourcesVpcConfig.clusterSecurityGroupId' --output text)

aws ec2 revoke-security-group-ingress \
  --group-id ${CLUSTER_SG} \
  --protocol tcp \
  --port 30090 \
  --cidr ${MY_IP}/32 \
  --region ${AWS_REGION} 2>/dev/null || true

aws ec2 revoke-security-group-ingress \
  --group-id ${CLUSTER_SG} \
  --protocol tcp \
  --port 30300 \
  --cidr ${MY_IP}/32 \
  --region ${AWS_REGION} 2>/dev/null || true
```
