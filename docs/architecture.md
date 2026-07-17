# 架构文档

本仓库包含 18 个 Demo，这里不做全量架构图汇总，只对其中组件交互较复杂、值得可视化的 Demo 提供架构图；其余 Demo 请直接看对应的 `docs/demoXX-*.md`。

以下 3 个 Demo 涉及多组件编排（CloudFormation + IAM + Kubernetes 声明式资源联动）或跨服务遥测管道，复杂度明显高于其余以"安装单个组件 + 验证"为主的 Demo，因此单独画图：

- **Demo09 — Karpenter 节点自动伸缩**：CloudFormation 基础设施 + 发现标签 + Pod Identity 授权 + NodePool/EC2NodeClass 声明式供给的完整安装与扩容链路
- **Demo11 — CSI 有状态存储（EBS/EFS/S3）**：三种 CSI 驱动各自不同的授权模型（Pod Identity vs 节点 Role）与访问模式（RWO vs RWX），叠加中国区 NFS 安全组与建桶 LocationConstraint 约束
- **Demo12 — 身份与访问控制**：Pod Identity（Pod→AWS）、Access Entry（人→集群）、Secrets Store CSI（外部密钥→Pod）三条独立授权路径
- **Demo18 — ADOT 与 OpenTelemetry 可观测性**：应用经 OTLP 上报 Collector，再分管道导出到 X-Ray 与 CloudWatch 的遥测管道

---

## Demo09 — Karpenter 节点自动伸缩

Karpenter 直接调用 EC2 API 按需选择实例类型，比 Cluster Autoscaler 更快更省。安装链路涉及 CloudFormation 建 IAM 资源、子网/安全组发现标签、控制器 Pod Identity 授权、节点 Role 的 Access Entry 注册，最后由 NodePool + EC2NodeClass 声明式定义可供给的节点规格。

```mermaid
flowchart TB
  CFN["CloudFormation Stack\nkarpenter-<cluster>\n(节点 Role + 5 个控制器 Policy + 中断处理 SQS)"]
  Tags["子网 + 集群安全组\nkarpenter.sh/discovery=<cluster> 标签"]
  CtrlRole["控制器 IAM Role\nKarpenterControllerRole\n(附加 CFN 输出的 5 个 Policy)"]
  PodID["Pod Identity Association\n(namespace=kube-system, sa=karpenter)"]
  NodeRole["节点 IAM Role\nKarpenterNodeRole"]
  AccessEntry["EKS Access Entry\ntype=EC2_LINUX（节点 Role 加入集群）"]
  KarpenterPod["Karpenter Controller\n(Helm, kube-system)"]
  NodePool["NodePool: default\n(实例类型/容量类型/整合策略)"]
  NodeClass["EC2NodeClass: default\n(AMI + 节点Role + 子网/SG 发现)"]
  Workload["inflate Deployment\n(0→5 副本，触发调度压力)"]
  NewNode["Karpenter 新建 EC2 节点"]

  CFN --> Tags
  CFN --> CtrlRole --> PodID --> KarpenterPod
  CFN --> NodeRole --> AccessEntry
  KarpenterPod --> NodePool
  KarpenterPod --> NodeClass
  NodeClass -.按 discovery 标签发现.-> Tags
  Workload -->|Pending Pod| KarpenterPod
  KarpenterPod -->|调用 EC2 API 创建实例| NewNode
  NewNode -->|经 Access Entry 注册加入集群| AccessEntry
  NewNode --> Workload
```

---

## Demo11 — CSI 有状态存储（EBS/EFS/S3）

三种 CSI 驱动的授权模型不同：EBS 用托管 addon 自动配置 Pod Identity；EFS 也走 Pod Identity，但节点还需 NFS(2049) 安全组放行 + 挂载目标；S3 Mountpoint 不支持 Pod Identity，权限须直接挂到节点实例 Role。中国区额外约束：EFS/S3 安全组与建桶都要显式处理（S3 建桶需 `LocationConstraint=cn-northwest-1`）。EBS/EFS 两个 addon 与 EFS 文件系统创建可并行触发以节省总耗时。

```mermaid
flowchart TB
  subgraph EBS["EBS CSI（块存储，ReadWriteOnce）"]
    EBSAddon["EBS CSI Addon\n(auto-apply-pod-identity-associations)"]
    EBSSC["StorageClass gp2"]
    EBSPod["Pod ebs-writer\n(ns: ebs-demo, 单节点独占写入)"]
    EBSAddon --> EBSSC --> EBSPod
  end

  subgraph EFS["EFS CSI（共享文件系统，ReadWriteMany）"]
    EFSAddon["EFS CSI Addon\n(Pod Identity)"]
    EFSSG["安全组: eks-efs-sg\n(放行节点 SG → 2049/NFS)"]
    EFSFS["EFS 文件系统\n(encrypted, 每子网一个 Mount Target)"]
    EFSDeploy["Deployment efs-app (2 replicas)\n(ns: efs-demo, 共享写入 shared.log)"]
    EFSAddon --> EFSFS
    EFSSG --> EFSFS --> EFSDeploy
  end

  subgraph S3CSI["S3 Mountpoint CSI（对象存储挂载）"]
    NodeRole["节点实例 Role\n(附加 AmazonS3FullAccess，不支持 Pod Identity)"]
    S3Bucket["S3 桶\n(建桶需 LocationConstraint=cn-northwest-1)"]
    S3Addon["S3 CSI Addon"]
    NodeRole --> S3Addon
    S3Bucket --> S3Addon
  end
```

---

## Demo12 — 身份与访问控制

聚焦中国区 EKS 三种独立的身份与访问控制机制，分别解决三个不同方向的授权问题：

```mermaid
flowchart TB
  subgraph A["Part A: Pod Identity（Pod → AWS）"]
    S3Role["IAM Role: eks-s3-access-role\n(信任 pods.eks.amazonaws.com, 挂 S3ReadOnlyAccess)"]
    PIA["Pod Identity Association\n(ns=pod-identity-demo, sa=s3-reader)"]
    S3Pod["Pod: s3-test-pod\n(aws-cli, serviceAccountName=s3-reader)"]
    S3Bucket["S3 Bucket\neks-podidentity-test-<acct>"]
    S3Role --> PIA --> S3Pod -->|无静态密钥, 直接 aws s3 ls| S3Bucket
  end

  subgraph B["Part B: Access Entry（人 → 集群）"]
    IAMUser["IAM User: eks-readonly-demo"]
    AccessEntry["EKS Access Entry\ntype=STANDARD"]
    Policy["associate-access-policy\nAmazonEKSViewPolicy\naccess-scope: namespace=access-entry-demo"]
    IAMUser --> AccessEntry --> Policy
  end

  subgraph C["Part C: Secrets Store CSI（外部密钥 → Pod）"]
    SecretsMgr["Secrets Manager\neks-demo-secret"]
    CSIDriver["Secrets Store CSI Driver\n(Helm) + AWS Provider"]
    IRSARole["IAM Role: eks-secrets-reader-role\n(IRSA, OIDC federated, aud=sts.amazonaws.com)"]
    SA["ServiceAccount: secrets-reader\n(eks.amazonaws.com/role-arn 注解)"]
    SPC["SecretProviderClass: aws-secrets\n(jmesPath 映射 username/password)"]
    SecretsPod["Pod: secrets-test-pod\n(挂载 CSI 卷 /mnt/secrets)"]

    IRSARole --> SA
    CSIDriver --> SPC
    SecretsMgr --> SPC
    SA --> SecretsPod
    SPC -->|CSI 卷挂载| SecretsPod
  end
```

---

## Demo18 — ADOT 与 OpenTelemetry 可观测性

ADOT Collector 以 OpenTelemetry 标准统一接收应用上报的追踪与指标数据，经 batch/resource 处理后分别导出到 X-Ray（追踪）与 CloudWatch EMF（指标），实现云原生应用的端到端可观测性。

```mermaid
flowchart LR
  IAMRole["IAM Role: eks-adot-collector-role\n(Pod Identity, 挂 ADOTCollectorPolicy:\ncloudwatch/xray/logs PutXxx)"]
  PIA["Pod Identity Association\n(ns=adot-demo, sa=adot-collector)"]
  Collector["ADOT Collector Deployment\n(Service: otlp-grpc:4317, otlp-http:4318)"]

  subgraph Pipeline["Collector Pipeline (ConfigMap)"]
    Recv["Receiver: otlp\n(grpc 4317 / http 4318)"]
    Proc["Processors: batch + resource\n(注入 aws.region / k8s.cluster.name)"]
    ExpXray["Exporter: awsxray"]
    ExpEmf["Exporter: awsemf\n(namespace=EKS/OTelDemo)"]
  end

  App["demo-app (Java)\nOTEL_EXPORTER_OTLP_ENDPOINT=\nhttp://adot-collector:4317"]
  XRay["AWS X-Ray\n(get-trace-summaries)"]
  CW["CloudWatch\nEKS/OTelDemo 命名空间\n+ 日志组 /aws/eks/<cluster>/otel-metrics"]

  IAMRole --> PIA --> Collector
  App -->|OTLP gRPC 上报 Trace+Metrics| Recv --> Proc
  Proc --> ExpXray --> XRay
  Proc --> ExpEmf --> CW
  Collector --> Pipeline
```

---

## 中国区特有约束（适用于以上及其余 Demo）

- **分区与 endpoint**：所有 IAM ARN 使用 `arn:aws-cn:`，ECR/服务 endpoint 使用 `.amazonaws.com.cn`
- **离线优先**：不现场访问 GitHub Release，Helm chart / IAM policy / CLI 二进制均来自仓库内 `offline-assets/` 与 `tools/`
- **镜像固定化**：所有容器镜像统一走 `048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn` 的固定 tag，不依赖 Docker Hub/quay.io/registry.k8s.io/ghcr.io
- **公网入口用 8080**：规避中国区域名备案对 80/443 端口的限制
- **配额前置检查**：Karpenter、Prometheus/Grafana、CloudWatch、ADOT、Fargate 等资源密集型 Demo 执行前需先确认 EC2 vCPU、ALB、EIP、Fargate、Spot 配额与容量
