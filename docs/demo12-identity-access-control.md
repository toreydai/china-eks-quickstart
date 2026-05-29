# Demo12 — 身份与访问控制（Pod Identity / Access Entry / Secrets Store CSI）

## 实验简介

本实验聚焦中国区 EKS 的三种身份与访问控制机制：Pod Identity（为 Pod 授予 AWS 权限访问 S3）、Access Entry（取代 aws-auth 控制谁能访问集群）、Secrets Store CSI（将 Secrets Manager 密钥安全注入 Pod）。三者覆盖了「Pod→AWS」「人→集群」「外部密钥→Pod」三条授权路径，是生产安全的基石。

**实验目标：**
- 掌握 Pod Identity、Access Entry、Secrets Store CSI 三种机制的配置
- 理解 Pod Identity 与 IRSA 在不同组件中的适用差异
- 能够将外部密钥安全地以卷的方式挂载到 Pod

**实验流程：**
1. 设置环境变量
2. （Part A）创建测试 S3 Bucket
3. 创建 Pod Identity IAM Role
4. 创建 Pod Identity Association 并验证
5. （Part B）创建 IAM User 并配置 Access Entry
6. （Part C）安装 Secrets Store CSI Driver
7. 配置 IRSA 权限
8. 创建 Secret 并挂载到 Pod

**预计时长：** 30-40 分钟

## 前提条件

- **工具**：AWS CLI v2、kubectl、helm
- **权限**：EKS、IAM、S3、Secrets Manager 权限
- **前提**：Demo01 已完成，Pod Identity Agent addon 已安装
- **预计耗时**：30-40 分钟

---

## 步骤

### 1. 设置环境变量

复用集群环境，定义本 Demo 三部分（Pod Identity / Access Entry / Secrets Store CSI）各自的命名空间与测试 S3 桶名。

```bash
source /tmp/demo-eks.env
export POD_ID_NS=pod-identity-demo
export ACCESS_ENTRY_NS=access-entry-demo
export SECRETS_NS=secrets-demo
export S3_BUCKET=eks-podidentity-test-${ACCOUNT_ID}
export ASSET_DIR=${ASSET_DIR:-$(pwd)/offline-assets}
```

---

## Part A — Pod Identity（AWS SDK 访问 S3）

### 2. 创建测试 S3 Bucket

创建测试桶并放入对象，供后续 Pod 通过 Pod Identity 访问验证。中国区建桶需带 LocationConstraint。

```bash
aws s3api create-bucket \
  --bucket ${S3_BUCKET} \
  --region ${AWS_REGION} \
  --create-bucket-configuration LocationConstraint=${AWS_REGION} || true

echo "test-object" | aws s3 cp - s3://${S3_BUCKET}/test.txt
echo "S3 bucket 准备完成"
```

**预期输出**：打印"S3 bucket 准备完成"

### 3. 创建 Pod Identity IAM Role

创建信任 `pods.eks.amazonaws.com` 的 Role 并附加 S3 只读托管策略（ARN 用 `aws-cn` 分区）。

```bash
cat > /tmp/s3-trust.json <<EOF
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
  --role-name eks-s3-access-role \
  --assume-role-policy-document file:///tmp/s3-trust.json || true

aws iam attach-role-policy \
  --role-name eks-s3-access-role \
  --policy-arn arn:${AWS_PARTITION}:iam::aws:policy/AmazonS3ReadOnlyAccess

S3_ROLE_ARN=$(aws iam get-role \
  --role-name eks-s3-access-role \
  --query 'Role.Arn' --output text)

echo "S3 Role ARN: ${S3_ROLE_ARN}"
```

**预期输出**：S3 Role ARN 输出

### 4. 创建 Pod Identity Association 并验证

将 Role 绑定到 s3-reader ServiceAccount，再用一个 awscli Pod 验证它无需任何静态密钥即可列出 S3 桶内容。

```bash
kubectl create namespace ${POD_ID_NS} --dry-run=client -o yaml | kubectl apply -f -

aws eks create-pod-identity-association \
  --cluster-name ${CLUSTER_NAME} \
  --namespace ${POD_ID_NS} \
  --service-account s3-reader \
  --role-arn ${S3_ROLE_ARN} \
  --region ${AWS_REGION}

# 创建 Service Account
kubectl create serviceaccount s3-reader -n ${POD_ID_NS}

# 创建测试 Pod（使用 awscli 访问 S3）
cat > /tmp/pod-identity-test.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: s3-test-pod
  namespace: ${POD_ID_NS}
spec:
  serviceAccountName: s3-reader
  containers:
    - name: aws-cli
      image: 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/aws-cli/aws-cli:2.27.41
      command: ["sh", "-c", "aws s3 ls s3://${S3_BUCKET}/ --region ${AWS_REGION} && echo 'S3 access OK'"]
      resources:
        limits:
          cpu: "100m"
          memory: "128Mi"
  restartPolicy: Never
EOF

kubectl apply -f /tmp/pod-identity-test.yaml
kubectl wait pod s3-test-pod -n ${POD_ID_NS} --for=condition=Ready --timeout=60s || true
sleep 10
kubectl logs s3-test-pod -n ${POD_ID_NS}
```

**预期输出**：Pod 日志显示 S3 bucket 内容和 `S3 access OK`

---

## Part B — Access Entry（EKS 集群访问控制）

### 5. 创建 IAM User 并配置 Access Entry

Access Entry 取代旧的 aws-auth ConfigMap，为一个 IAM User 授予限定命名空间的只读集群访问权限（AmazonEKSViewPolicy）。

```bash
kubectl create namespace ${ACCESS_ENTRY_NS} --dry-run=client -o yaml | kubectl apply -f -

# 创建只读用户（假设用于演示的 IAM User 已存在或临时创建）
READONLY_USER_ARN="arn:${AWS_PARTITION}:iam::${ACCOUNT_ID}:user/eks-readonly-demo"
aws iam create-user --user-name eks-readonly-demo || true

# 创建 Access Entry — STANDARD 类型
aws eks create-access-entry \
  --cluster-name ${CLUSTER_NAME} \
  --principal-arn ${READONLY_USER_ARN} \
  --type STANDARD \
  --region ${AWS_REGION} || true

# 关联 AmazonEKSViewPolicy（只读权限），限定在特定 namespace
aws eks associate-access-policy \
  --cluster-name ${CLUSTER_NAME} \
  --principal-arn ${READONLY_USER_ARN} \
  --policy-arn arn:${AWS_PARTITION}:iam::aws:policy/AmazonEKSViewPolicy \
  --access-scope type=namespace,namespaces=${ACCESS_ENTRY_NS} \
  --region ${AWS_REGION}

echo "Access Entry 配置完成"
aws eks list-access-entries --cluster-name ${CLUSTER_NAME} --region ${AWS_REGION}
```

**预期输出**：Access Entry 列表中包含只读用户

---

## Part C — Secrets Store CSI（从 Secrets Manager 注入密钥）

### 6. 安装 Secrets Store CSI Driver

用 Helm 安装 CSI Driver 并部署 AWS Provider，使集群能把外部 Secrets Manager 密钥挂载为卷。CSI Driver 链路涉及 4 个镜像（driver、driver-crds、node-driver-registrar、livenessprobe）和 AWS Provider 镜像，均从集中预置的 0910 仓库拉取，因此需用 `--set` 覆盖 chart 默认的 `registry.k8s.io` 镜像（注意 registrar/livenessprobe 的 tag 须用 0910 实际预置的 `v2.14.0`/`v2.16.0`），并对 provider installer YAML 做 `sed` 替换。

```bash
cp ${ASSET_DIR}/secrets-store-csi-driver-1.6.0.tgz /tmp/

kubectl create namespace ${SECRETS_NS} --dry-run=client -o yaml | kubectl apply -f -

REG=048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn

helm upgrade --install secrets-store-csi-driver /tmp/secrets-store-csi-driver-1.6.0.tgz \
  -n ${SECRETS_NS} \
  --set syncSecret.enabled=true \
  --set enableSecretRotation=true \
  --set linux.image.repository=${REG}/registryk8sio/csi-secrets-store/driver \
  --set linux.image.tag=v1.6.0 \
  --set linux.crds.image.repository=${REG}/registryk8sio/csi-secrets-store/driver-crds \
  --set linux.crds.image.tag=v1.6.0 \
  --set linux.registrarImage.repository=${REG}/registryk8sio/sig-storage/csi-node-driver-registrar \
  --set linux.registrarImage.tag=v2.14.0 \
  --set linux.livenessProbeImage.repository=${REG}/registryk8sio/sig-storage/livenessprobe \
  --set linux.livenessProbeImage.tag=v2.16.0

kubectl -n ${SECRETS_NS} rollout status deployment/secrets-store-csi-driver

# 安装 AWS Provider（把镜像地址替换为 0910 预置的精确镜像）
cp ${ASSET_DIR}/aws-provider-installer.yaml /tmp/
sed -i -E "s|public.ecr.aws/aws-secrets-manager/secrets-store-csi-driver-provider-aws:[^\"' ]*|${REG}/ecrpublic/aws-secrets-manager/secrets-store-csi-driver-provider-aws:3.1.0|g" /tmp/aws-provider-installer.yaml
kubectl apply -f /tmp/aws-provider-installer.yaml
```


> ⚠️ **首次拉取 048912060910 镜像若出现 ImagePullBackOff（403）**：这是节点 ECR credential provider 冷启动的短暂问题，无需修改任何权限配置。等待约 30 秒后执行 `kubectl rollout restart deployment/<name> -n <ns>` 即可恢复。节点一旦缓存镜像后，同节点后续 Pod 直接走本地缓存不再触发。

**预期输出**：CSI Driver 就绪

### 7. 配置 IRSA 权限

Secrets Store CSI 通过 IRSA 读取 Secrets Manager，需配置 OIDC 信任策略（中国区域名带 `.cn`）并给 ServiceAccount 打 role-arn 注解。

```bash
OIDC_URL=$(aws eks describe-cluster \
  --name ${CLUSTER_NAME} \
  --query 'cluster.identity.oidc.issuer' --output text)
OIDC_ID=$(echo ${OIDC_URL} | awk -F'/' '{print $NF}')

cat > /tmp/secrets-trust.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:${AWS_PARTITION}:iam::${ACCOUNT_ID}:oidc-provider/oidc.eks.${AWS_REGION}.amazonaws.com.cn/id/${OIDC_ID}"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "oidc.eks.${AWS_REGION}.amazonaws.com.cn/id/${OIDC_ID}:sub": "system:serviceaccount:${SECRETS_NS}:secrets-reader",
        "oidc.eks.${AWS_REGION}.amazonaws.com.cn/id/${OIDC_ID}:aud": "sts.amazonaws.com"
      }
    }
  }]
}
EOF

aws iam create-role \
  --role-name eks-secrets-reader-role \
  --assume-role-policy-document file:///tmp/secrets-trust.json || true

aws iam attach-role-policy \
  --role-name eks-secrets-reader-role \
  --policy-arn arn:${AWS_PARTITION}:iam::aws:policy/SecretsManagerReadWrite

SECRETS_ROLE_ARN=$(aws iam get-role \
  --role-name eks-secrets-reader-role \
  --query 'Role.Arn' --output text)

kubectl create serviceaccount secrets-reader -n ${SECRETS_NS} --dry-run=client -o yaml | \
  kubectl annotate --local -f - \
  eks.amazonaws.com/role-arn=${SECRETS_ROLE_ARN} -o yaml | kubectl apply -f -
```

**预期输出**：ServiceAccount 已创建并有 IRSA 注解

### 8. 创建 Secret 并挂载到 Pod

在 Secrets Manager 创建密钥，用 SecretProviderClass 定义字段映射，Pod 挂载该 CSI 卷后即可在文件系统读取密钥值。

```bash
# 创建测试 Secret
aws secretsmanager create-secret \
  --name eks-demo-secret \
  --secret-string '{"username":"admin","password":"mypassword123"}' \
  --region ${AWS_REGION} || true

# 创建 SecretProviderClass
cat > /tmp/spc.yaml <<EOF
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: aws-secrets
  namespace: ${SECRETS_NS}
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "eks-demo-secret"
        objectType: "secretsmanager"
        jmesPath:
          - path: username
            objectAlias: db-username
          - path: password
            objectAlias: db-password
  secretObjects:
    - secretName: eks-demo-secret
      type: Opaque
      data:
        - objectName: db-username
          key: username
        - objectName: db-password
          key: password
EOF

kubectl apply -f /tmp/spc.yaml

# 创建使用 Secret 的 Pod
cat > /tmp/secrets-pod.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: secrets-test-pod
  namespace: ${SECRETS_NS}
spec:
  serviceAccountName: secrets-reader
  containers:
    - name: app
      image: 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/busybox:1.36
      command: ["sh", "-c", "ls /mnt/secrets && cat /mnt/secrets/db-username && sleep 3600"]
      volumeMounts:
        - name: secrets-vol
          mountPath: /mnt/secrets
          readOnly: true
      resources:
        limits:
          cpu: "100m"
          memory: "64Mi"
  volumes:
    - name: secrets-vol
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: aws-secrets
EOF

kubectl apply -f /tmp/secrets-pod.yaml
kubectl wait pod secrets-test-pod -n ${SECRETS_NS} --for=condition=Ready --timeout=60s
kubectl logs secrets-test-pod -n ${SECRETS_NS}
```

**预期输出**：Pod 日志输出 `admin`（db-username 的值）

---

## 验收标准

完成本实验后，你应当能够：
- [ ] Pod 通过 Pod Identity 无需密钥即可访问 S3（日志输出 S3 access OK）
- [ ] 通过 Access Entry 为 IAM User 授予限定命名空间的只读集群权限
- [ ] Secrets Store CSI Driver 在集群中正常运行
- [ ] Pod 成功挂载并读取来自 Secrets Manager 的密钥值

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws eks list-pod-identity-associations --cluster-name demo --namespace pod-identity-demo --region cn-northwest-1 --query 'length(associations)' --output text` | `1` |
| 2 | `kubectl logs s3-test-pod -n pod-identity-demo 2>/dev/null \| grep -c 'S3 access OK'` | `1` |
| 3 | `aws eks list-access-entries --cluster-name demo --region cn-northwest-1 --query 'length(accessEntries)' --output text` | 大于等于 `1` |
| 4 | `kubectl get daemonset secrets-store-csi-driver-windows -n secrets-demo 2>/dev/null \| wc -l; kubectl get daemonset secrets-store-csi-driver -n secrets-demo -o jsonpath='{.status.numberReady}'` | 大于 `0` |

---

## 实验总结

本实验打通了三条授权路径：Pod Identity 让工作负载安全访问 AWS 服务，Access Entry 以声明式方式管理人对集群的访问，Secrets Store CSI 把外部密钥安全注入 Pod。你也再次实践了中国区 `aws-cn` 分区与 `.amazonaws.com.cn` OIDC 域名等差异。下一个 Demo（Demo13）将进入 Prometheus 与 Grafana 集群监控。

## 清理

```bash
source /tmp/demo-eks.env

kubectl delete namespace ${POD_ID_NS} ${ACCESS_ENTRY_NS} ${SECRETS_NS} 2>/dev/null || true

aws eks delete-pod-identity-association \
  --cluster-name ${CLUSTER_NAME} \
  --association-id $(aws eks list-pod-identity-associations \
    --cluster-name ${CLUSTER_NAME} \
    --namespace pod-identity-demo \
    --service-account s3-reader \
    --query 'associations[0].associationId' --output text) \
  --region ${AWS_REGION} || true

aws eks delete-access-entry \
  --cluster-name ${CLUSTER_NAME} \
  --principal-arn arn:${AWS_PARTITION}:iam::${ACCOUNT_ID}:user/eks-readonly-demo \
  --region ${AWS_REGION} || true
aws iam delete-user --user-name eks-readonly-demo || true

for role in eks-s3-access-role eks-secrets-reader-role; do
  aws iam detach-role-policy --role-name ${role} \
    --policy-arn arn:${AWS_PARTITION}:iam::aws:policy/AmazonS3ReadOnlyAccess 2>/dev/null || true
  aws iam detach-role-policy --role-name ${role} \
    --policy-arn arn:${AWS_PARTITION}:iam::aws:policy/SecretsManagerReadWrite 2>/dev/null || true
  aws iam delete-role --role-name ${role} 2>/dev/null || true
done

aws secretsmanager delete-secret \
  --secret-id eks-demo-secret \
  --force-delete-without-recovery \
  --region ${AWS_REGION} || true

aws s3 rm s3://${S3_BUCKET} --recursive || true
aws s3api delete-bucket --bucket ${S3_BUCKET} --region ${AWS_REGION} || true
```
