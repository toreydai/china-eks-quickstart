# GCR EKS QuickStart
## AWS 中国区 EKS 动手实验集

基于 EKS 1.35 · Amazon Linux 2023 · 宁夏区（cn-northwest-1）

---

## 与 Global QuickStart 的差异和运行前检查

本 QuickStart 面向 AWS 中国区，不能完全照搬 Global 区域的下载源、镜像源和 ARN/endpoint 习惯。开始前建议确认以下事项，避免 Demo 执行到一半被网络、分区或配额问题打断：

1. 使用 `cn-northwest-1`，IAM ARN 使用 `arn:aws-cn:`，服务 endpoint 使用 `.amazonaws.com.cn`。
2. 不依赖现场访问 GitHub、GitHub raw 或上游 Helm repo；小型 Helm chart、安装 YAML、IAM policy 已随仓库放在 `offline-assets/`。
3. Demo01 会先检查工具是否已安装；缺失时请先在可联网环境下载 Linux amd64 版本的 `eksctl`、`kubectl`、`helm.tar.gz`，再上传到操作机的 `tools/` 目录。
4. 镜像不直接拉 Docker Hub、quay.io、ghcr.io、registry.k8s.io 等上游源；Demo 镜像统一使用 `048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn` 中的预置镜像，并使用固定 tag。
5. 公网入口 Demo 使用 8080，避免 80/443 备案限制影响验证。
6. 执行 Karpenter、Cluster Autoscaler、Prometheus/Grafana、CloudWatch、ADOT、Fargate 前，先确认 EC2 vCPU、ALB、EIP、Fargate、Spot 等配额和容量。
7. 小规格节点容易遇到 Pod density 或资源不足；完整跑多个重型 Demo 前建议清理前序资源，或至少保留 2-3 个 worker nodes。

---

## Demo 列表

建议先完成 **基础**，再按角色和时间选择 **进阶**。基础覆盖 EKS 从建集群、部署应用、镜像发布、健康检查、调度伸缩、存储、身份到原生可观测性的最小闭环；进阶用于补充生产专题、平台治理和专项运维能力。

## 基础

### 1. 基础环境

* [Demo01 — 准备实验环境与创建集群](docs/demo01-cluster-setup.md)

### 2. 应用部署、镜像与健康检查

* [Demo02 — ALB Controller 与 NGINX 应用](docs/demo02-alb-controller-nginx.md)
* [Demo03 — ECR 私有镜像与镜像发布](docs/demo03-ecr-image-management.md)（保留基础发布；跨区域/跨账号段落选做）
* [Demo04 — 健康检查与故障排查](docs/demo04-health-checks-troubleshooting.md)

### 3. 发布与调度

* [Demo06 — Helm 与 GitOps 发布（ArgoCD）](docs/demo06-helm-gitops.md)
* [Demo07 — 调度策略与资源治理](docs/demo07-scheduling-resource-governance.md)
* [Demo08 — HPA 水平自动伸缩](docs/demo08-hpa-autoscaling.md)
* [Demo09 — Karpenter 节点自动伸缩](docs/demo09-karpenter.md)

### 4. 存储、身份与可观测性

* [Demo11 — CSI 有状态存储（EBS / EFS / S3）](docs/demo11-csi-stateful-storage.md)
* [Demo12 — 身份与访问控制（Pod Identity / Access Entry / Secrets Store CSI）](docs/demo12-identity-access-control.md)（Secrets Store CSI 部分选做）
* [Demo14 — CloudWatch Observability 原生可观测性](docs/demo14-cloudwatch-observability.md)

## 进阶

### 5. 可靠性、入口与计算形态

* [Demo05 — Velero 备份与恢复](docs/demo05-velero-backup-restore.md)
* [Demo10 — Cluster Autoscaler 节点自动伸缩](docs/demo10-cluster-autoscaler.md)（对比/存量 ASG 选做）
* [Demo15 — Spot 节点与 Fargate Profile](docs/demo15-spot-fargate.md)

### 6. 安全、网络与可观测性扩展

* [Demo16 — NetworkPolicy 限制 Pod 流量](docs/demo16-network-policy.md)
* [Demo17 — Pod Security Standards 工作负载安全](docs/demo17-pod-security-standards.md)
* [Demo13 — Prometheus 与 Grafana 集群监控](docs/demo13-prometheus-grafana.md)
* [Demo18 — ADOT 与 OpenTelemetry 可观测性](docs/demo18-adot-opentelemetry.md)

## 使用方式

1. 在此目录下打开 Claude Code，`CLAUDE.md` 自动加载中国区约束
2. 按顺序打开 `docs/demoXX-*.md`（XX 为 01–18），逐步执行对应 Demo
3. 每个 Demo 末尾均有**清理**步骤，实验结束后执行以避免持续计费

## 环境要求

| 工具 | 版本 |
|------|------|
| AWS CLI | 2.x（AL2023 预装） |
| eksctl | latest |
| kubectl | v1.35 |
| Helm | 3.x |
| jq | latest |

所需 IAM 权限参考 [eksctl 最小策略（中国区）](https://github.com/toreydai/eksctl-min-iam-policies-cn)。

## License

MIT - see the [LICENSE](LICENSE) file for details.

## 免责声明

- 本项目仅供学习与技术参考，不构成生产部署方案。
- 运行过程中会创建 AWS 资源并产生费用，请在实验结束后及时清理。
- 作者不对因使用本项目产生的任何费用或损失承担责任。
- 本项目与 Amazon Web Services 无官方关联，相关服务的可用性与定价以 AWS 官方文档为准。
- 生产环境使用前请根据实际需求进行安全评估与调整。