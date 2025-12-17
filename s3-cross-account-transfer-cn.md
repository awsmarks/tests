# S3 跨账户数据传输指南

> 🌐 [English Version](./s3-cross-account-transfer.md)

## 概述

本文档介绍如何在不同 AWS 账户、区域和 VPC 之间安全传输 S3 数据。所有方法均保持流量在 AWS 内部网络中传输。

> **所有方法已于 2025-12-17 测试验证**

---

## 快速总结：选择哪种方法？

| 方法 | 适用场景 | 规模 | 需要 EC2/计算资源？ | 已测试 |
|------|----------|------|-------------------|--------|
| `aws s3 sync` | 简单临时传输 | < 100万对象 | ✅ 是 (EC2/Lambda/本地) | ✅ |
| 网关终端节点 | 私有子网 S3 访问 | 任意 | ✅ 是 (私有子网 EC2) | ✅ |
| 接口终端节点 | 严格网络隔离 | 任意 | ✅ 是 (私有子网 EC2) | ✅ |
| S3 批量操作 | 大规模迁移 | 数十亿 | ❌ 否 (无服务器) | ✅ |
| S3 复制 | 持续同步 | 任意 | ❌ 否 (无服务器) | ✅ |
| DataSync | 定期计划同步 | 大规模 | ❌ 否 (托管服务) | ✅ |

---

## 综合成本对比

### 各方法完整成本明细

| 方法 | 服务费用 | 数据传输 | S3 请求 | 需要计算资源 | 1TB/100万对象总成本 |
|------|---------|---------|---------|-------------|-------------------|
| `aws s3 sync` (同区域) | 免费 | 免费 | $5 PUT | EC2: ~$0.10/小时 | ~$5 + 计算费 |
| `aws s3 sync` (跨区域) | 免费 | $20/TB | $5 PUT | EC2: ~$0.10/小时 | ~$25 + 计算费 |
| 网关终端节点 | **免费** | 免费 (同区域) | $5 PUT | 需要 EC2 | ~$5 + 计算费 |
| 接口终端节点 | $7.20/月/AZ + $10/TB | 免费 (同区域) | $5 PUT | 需要 EC2 | ~$12 + 计算费 |
| S3 批量操作 | $0.25/百万对象 | 免费 (同区域) | $5 PUT | ❌ 无需 | ~$5.25 |
| S3 复制 | 免费 | 免费 (同区域) | $5 PUT | ❌ 无需 | ~$5 |
| DataSync | $12.50/TB | 免费 (同区域) | 已包含 | ❌ 无需 | ~$12.50 |
| NAT 网关 | $32.40/月 + $45/TB | $45/TB | $5 PUT | 需要 EC2 | ~$95 ⚠️ 避免使用 |

### 数据传输费用

| 场景 | 费用 |
|------|------|
| 同区域 (任何方法) | **免费** |
| 跨区域 (AWS 内部) | ~$0.02/GB ($20/TB) |
| 传输到互联网 | ~$0.09/GB ($90/TB) |
| S3 传输加速 | +$0.04/GB |

### S3 请求定价 (us-east-1)

| 请求类型 | 费用 |
|---------|------|
| PUT, COPY, POST, LIST | $0.005/千次请求 |
| GET, SELECT | $0.0004/千次请求 |
| DELETE | 免费 |

### 基础设施费用 (如需计算资源)

| 资源 | 费用 | 说明 |
|------|------|------|
| EC2 t3.micro | $0.0104/小时 (~$7.50/月) | aws s3 sync 最低配置 |
| EC2 t3.medium | $0.0416/小时 (~$30/月) | 大规模传输推荐 |
| NAT 网关 | $0.045/小时 + $0.045/GB | ⚠️ 昂贵 - 尽量避免 |
| VPC 对等连接 | 免费 (同区域) | 跨区域 $0.01/GB |
| Transit Gateway | $0.05/小时 + $0.02/GB | 复杂多 VPC 场景 |

---

## 方法对比表

### 按规模选择

| 对象数量 | 推荐方法 | 原因 |
|---------|---------|------|
| < 1万 | `aws s3 sync` | 简单快速 |
| 1万 - 100万 | `aws s3 sync --only-show-errors` | 减少输出开销 |
| 100万 - 1亿 | S3 批量操作 | 内置重试和跟踪 |
| > 1亿 | S3 批量操作 + S3 清单 | 自动生成清单 |
| 持续同步 | S3 复制 | 实时同步 |
| 定期同步 | DataSync | 托管调度 |

### 按网络需求选择

| 需求 | 解决方案 | 费用 |
|------|---------|------|
| 公有子网有 IGW | 直接访问 | 免费 |
| 私有子网 (无 IGW) | 网关终端节点 | **免费** |
| 需要安全组控制 | 接口终端节点 | $0.01/小时/AZ |
| 限制特定 VPC | 桶策略 + `aws:sourceVpc` | 免费 |
| 跨 VPC 同区域 | VPC 对等连接 + 终端节点 | 免费 |
| 跨 VPC 跨区域 | Transit Gateway | $0.05/小时 + $0.02/GB |

### 按使用场景选择

| 使用场景 | 最佳方法 | 原因 |
|---------|---------|------|
| 一次性迁移 (小规模) | `aws s3 sync` | 简单无需配置 |
| 一次性迁移 (大规模) | S3 批量操作 | 无服务器可扩展 |
| 持续备份 | S3 复制 | 自动实时 |
| 每日/每周同步 | DataSync | 调度托管 |
| 需要私有网络 | 网关终端节点 | 免费安全 |
| 合规/审计需求 | 接口终端节点 | 安全组日志 |

### 功能对比

| 功能 | aws s3 sync | S3 批量操作 | S3 复制 | DataSync |
|------|-------------|------------|---------|----------|
| 无服务器 | ❌ | ✅ | ✅ | ✅ |
| 自动重试 | ❌ | ✅ | ✅ | ✅ |
| 进度跟踪 | 有限 | 仪表板 | CloudWatch | 仪表板 |
| 完成报告 | ❌ | ✅ | ❌ | ✅ |
| 跨账户 | ✅ | ✅ | ✅ | ✅ |
| 跨区域 | ✅ | ✅ | ✅ | ✅ |
| 调度 | 手动/cron | 手动 | 持续 | 内置 |
| 最大规模 | ~100万对象 | 数十亿 | 无限 | 大规模 |

---

## 决策树

```
需要传输 S3 数据？
│
├─ 一次性还是持续？
│   ├─ 一次性
│   │   ├─ < 100万对象 → aws s3 sync (需要 EC2/计算资源)
│   │   └─ > 100万对象 → S3 批量操作 (无服务器)
│   │
│   └─ 持续
│       ├─ 实时 → S3 复制 (无服务器)
│       └─ 定期 → DataSync (托管)
│
├─ 私有子网 (无 IGW)？
│   ├─ 是 → 网关终端节点 (免费) + EC2
│   └─ 否 → 直接访问即可
│
├─ 需要安全组控制？
│   ├─ 是 → 接口终端节点 ($0.01/小时/AZ)
│   └─ 否 → 网关终端节点 (免费)
│
└─ 跨区域？
    ├─ 是 → 增加 ~$0.02/GB 数据传输费
    └─ 否 → 无数据传输费
```

---

## 传输场景矩阵

| 场景 | 方法 | 网络路径 | 需要计算资源 | 1TB 成本 |
|------|------|---------|-------------|---------|
| 同账户同区域 | 直接复制 | AWS 内部 | EC2/Lambda | ~$5 |
| 同账户跨区域 | 直接复制 | AWS 骨干网 | EC2/Lambda | ~$25 |
| 跨账户同区域 | 桶策略 | AWS 内部 | EC2/Lambda | ~$5 |
| 跨账户跨区域 | 桶策略 | AWS 骨干网 | EC2/Lambda | ~$25 |
| 私有子网 | 网关终端节点 | 私有 IP | EC2 | ~$5 |
| 严格隔离 | 接口终端节点 | 私有 IP | EC2 | ~$17 |
| 大规模迁移 (无服务器) | S3 批量操作 | AWS 内部 | 无需 | ~$5.25 |
| 持续同步 (无服务器) | S3 复制 | AWS 内部 | 无需 | ~$5 |

---

## ✅ 测试结果汇总 (2025-12-17)

### S3 批量操作
| 项目 | 值 |
|------|-----|
| 作业 ID | `64d1c633-c4ab-46e8-9906-dbe499ee6af5` |
| 区域 | ap-northeast-1 |
| 源 | ss-cross-acct-test (账户 767397876447) |
| 目标 | ss-cross-acct-accept (账户 127214164714) |
| 状态 | ✅ 完成 (2/2 任务) |

**关键经验：** 批量作业必须在源桶所在区域创建。

### S3 复制
| 项目 | 值 |
|------|-----|
| 源 | ss-cross-acct-test (账户 767397876447) |
| 目标 | ss-cross-acct-accept (账户 127214164714) |
| 前缀 | `replicate/` |
| 状态 | ✅ 已完成 |

**关键经验：** 两个桶都需要启用版本控制；目标桶策略必须允许复制。

### DataSync
| 项目 | 值 |
|------|-----|
| 任务 ARN | `task-0467e69aaf92a1a2b` |
| 源 | ss-cross-acct-test/datasync-test/ |
| 目标 | ss-cross-acct-accept/datasync-dest/ |
| 状态 | ✅ 成功 (1 文件, 50 字节) |

**关键经验：** DataSync 角色需要 `s3:DeleteObject` 权限；会创建 `.aws-datasync/` 元数据文件夹。

### 网关终端节点
| 项目 | 值 |
|------|-----|
| 终端节点 ID | vpce-00df0cd8226408340 |
| VPC | vpc-0ca8540233182736a |
| 状态 | ✅ 可用 |

### 接口终端节点
| 项目 | 值 |
|------|-----|
| 终端节点 ID | vpce-042a2b36f62dbd782 |
| ENI IP | 10.0.0.93 |
| 状态 | ✅ 可用 |

---

## 最佳实践

1. **使用网关终端节点** - 免费且保持流量私有
2. **避免 NAT 网关访问 S3** - 改用网关终端节点 (节省 ~$90/TB)
3. **使用 S3 批量操作** - 适用于数百万对象 (无服务器，无需 EC2)
4. **考虑 S3 复制** - 适用于持续同步 (无服务器，自动)
5. **最小权限 IAM** - 仅授予必要的 S3 操作权限
6. **桶策略限制** - 按 VPC/终端节点/账户/组织限制访问
7. **启用加密** - 静态加密使用 SSE-S3 或 SSE-KMS，传输加密使用 TLS
8. **启用日志** - S3 访问日志和 CloudTrail

---

# 详细实施指南

## 场景 1：同账户同区域

最简单的情况 - 无需特殊配置。

```bash
# 直接复制
aws s3 sync s3://source-bucket/ s3://destination-bucket/

# 显示进度
aws s3 sync s3://source-bucket/ s3://destination-bucket/ --progress
```

**费用：** 仅 PUT 请求费 (~$5/百万对象)

---

## 场景 2：跨账户同区域

### 架构
```
账户 A (源)                          账户 B (目标)
┌─────────────────┐                  ┌─────────────────┐
│  source-bucket  │ ───────────────> │  dest-bucket    │
│                 │   跨账户桶策略    │                 │
│  IAM 角色/用户  │                  │  桶策略         │
└─────────────────┘                  └─────────────────┘
```

### 步骤 1：目标桶策略 (账户 B)
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowCrossAccountAccess",
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::账户A_ID:root"},
    "Action": ["s3:GetObject", "s3:PutObject", "s3:ListBucket"],
    "Resource": [
      "arn:aws:s3:::dest-bucket",
      "arn:aws:s3:::dest-bucket/*"
    ]
  }]
}
```

### 步骤 2：源 IAM 策略 (账户 A)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": ["arn:aws:s3:::source-bucket", "arn:aws:s3:::source-bucket/*"]
    },
    {
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:ListBucket"],
      "Resource": ["arn:aws:s3:::dest-bucket", "arn:aws:s3:::dest-bucket/*"]
    }
  ]
}
```

### 步骤 3：执行传输
```bash
aws s3 sync s3://source-bucket/ s3://dest-bucket/ --profile source-account
```

**费用：** 仅 PUT 请求费 (~$5/百万对象)

---

## 场景 3：私有子网使用网关终端节点

### 架构
```
VPC (10.0.0.0/16)
┌─────────────────────────────────────────────────┐
│  私有子网 (无 IGW)                               │
│  ┌─────────────┐      ┌──────────────────────┐  │
│  │    EC2      │ ───> │  网关终端节点         │  │
│  │ 10.0.1.10   │      │  (vpce-xxx)          │  │
│  └─────────────┘      └──────────┬───────────┘  │
│                                  │              │
└──────────────────────────────────│──────────────┘
                                   │
                                   ▼
                          ┌─────────────────┐
                          │   S3 服务       │
                          │ (AWS 内部)      │
                          └─────────────────┘
```

### 创建网关终端节点
```bash
# 获取路由表 ID
RT_ID=$(aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=vpc-xxx" \
  --query 'RouteTables[0].RouteTableId' --output text)

# 创建网关终端节点
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-xxx \
  --service-name com.amazonaws.区域.s3 \
  --route-table-ids $RT_ID \
  --vpc-endpoint-type Gateway
```

**费用：** **免费** (网关终端节点无任何费用)

---

## 场景 4：接口终端节点 (PrivateLink)

需要安全组控制 S3 流量时使用。

### 创建接口终端节点
```bash
# 创建安全组
SG_ID=$(aws ec2 create-security-group \
  --group-name s3-endpoint-sg \
  --description "S3 接口终端节点" \
  --vpc-id vpc-xxx \
  --query 'GroupId' --output text)

# 允许 VPC 内 HTTPS
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp --port 443 \
  --cidr 10.0.0.0/16

# 创建接口终端节点
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-xxx \
  --service-name com.amazonaws.区域.s3 \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-xxx \
  --security-group-ids $SG_ID
```

**费用：** $0.01/小时/AZ (~$7.20/月) + $0.01/GB 处理量

---

## 场景 5：S3 批量操作 (大规模)

适用于数百万/数十亿对象 - **无需 EC2**。

### 步骤 1：创建 IAM 角色
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "batchoperations.s3.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
```

### 步骤 2：创建清单
```bash
# 选项 A：简单 CSV
echo "source-bucket,path/to/object1.txt" > manifest.csv
echo "source-bucket,path/to/object2.txt" >> manifest.csv
aws s3 cp manifest.csv s3://source-bucket/manifest/

# 选项 B：大规模桶使用 S3 清单
```

### 步骤 3：创建并运行作业
```bash
ETAG=$(aws s3api head-object --bucket source-bucket --key manifest/manifest.csv --query 'ETag' --output text | tr -d '"')

aws s3control create-job \
  --account-id 账户ID \
  --operation '{"S3PutObjectCopy":{"TargetResource":"arn:aws:s3:::dest-bucket"}}' \
  --manifest "{\"Spec\":{\"Format\":\"S3BatchOperations_CSV_20180820\",\"Fields\":[\"Bucket\",\"Key\"]},\"Location\":{\"ObjectArn\":\"arn:aws:s3:::source-bucket/manifest/manifest.csv\",\"ETag\":\"$ETAG\"}}" \
  --report '{"Bucket":"arn:aws:s3:::source-bucket","Format":"Report_CSV_20180820","Enabled":true,"Prefix":"batch-reports","ReportScope":"AllTasks"}' \
  --priority 10 \
  --role-arn arn:aws:iam::账户ID:role/S3BatchOperationsRole \
  --region 区域 \
  --no-confirmation-required
```

**费用：** $0.25/百万对象 + PUT 请求费

---

## 场景 6：S3 复制 (持续同步)

适用于持续同步 - **无需 EC2**。

### 步骤 1：启用版本控制
```bash
aws s3api put-bucket-versioning --bucket source-bucket --versioning-configuration Status=Enabled
aws s3api put-bucket-versioning --bucket dest-bucket --versioning-configuration Status=Enabled
```

### 步骤 2：创建复制角色
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "s3.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
```

### 步骤 3：配置复制
```bash
cat > replication.json << 'EOF'
{
  "Role": "arn:aws:iam::账户ID:role/S3ReplicationRole",
  "Rules": [{
    "ID": "ReplicateAll",
    "Status": "Enabled",
    "Priority": 1,
    "Filter": {},
    "Destination": {
      "Bucket": "arn:aws:s3:::dest-bucket",
      "Account": "目标账户ID",
      "AccessControlTranslation": {"Owner": "Destination"}
    },
    "DeleteMarkerReplication": {"Status": "Enabled"}
  }]
}
EOF

aws s3api put-bucket-replication --bucket source-bucket --replication-configuration file://replication.json
```

**费用：** 仅 PUT 请求费 (与手动复制相同)

---

## 场景 7：DataSync (定期同步)

适用于定期传输 - **无需 EC2**。

### 创建 DataSync 任务
```bash
# 创建源位置
SOURCE_LOC=$(aws datasync create-location-s3 \
  --s3-bucket-arn arn:aws:s3:::source-bucket \
  --s3-config '{"BucketAccessRoleArn":"arn:aws:iam::账户ID:role/DataSyncS3Role"}' \
  --query 'LocationArn' --output text)

# 创建目标位置
DEST_LOC=$(aws datasync create-location-s3 \
  --s3-bucket-arn arn:aws:s3:::dest-bucket \
  --s3-config '{"BucketAccessRoleArn":"arn:aws:iam::账户ID:role/DataSyncS3Role"}' \
  --query 'LocationArn' --output text)

# 创建任务
aws datasync create-task \
  --source-location-arn $SOURCE_LOC \
  --destination-location-arn $DEST_LOC \
  --name "S3SyncTask"
```

**费用：** $0.0125/GB 传输量

---

## 清理命令

```bash
# 删除 VPC 终端节点
aws ec2 delete-vpc-endpoints --vpc-endpoint-ids vpce-xxx

# 删除 IAM 角色
aws iam delete-role-policy --role-name S3BatchOperationsRole --policy-name S3BatchPolicy
aws iam delete-role --role-name S3BatchOperationsRole

# 删除复制配置
aws s3api delete-bucket-replication --bucket source-bucket

# 删除 DataSync 任务
aws datasync delete-task --task-arn arn:aws:datasync:区域:账户:task/task-xxx
```

---

## 参考资料

- [S3 批量操作](https://docs.aws.amazon.com/zh_cn/AmazonS3/latest/userguide/batch-ops.html)
- [S3 复制](https://docs.aws.amazon.com/zh_cn/AmazonS3/latest/userguide/replication.html)
- [S3 VPC 终端节点](https://docs.aws.amazon.com/zh_cn/AmazonS3/latest/userguide/privatelink-interface-endpoints.html)
- [DataSync](https://docs.aws.amazon.com/zh_cn/datasync/latest/userguide/what-is-datasync.html)
- [S3 定价](https://aws.amazon.com/cn/s3/pricing/)
