# S3 Cross-Account Data Transfer Guide

> 🌐 [中文版本 (Chinese Version)](./s3-cross-account-transfer-cn.md)

## Overview

This document describes how to securely transfer data between S3 buckets across different AWS accounts, regions, and VPCs. All methods keep traffic within AWS's internal network.

> **All methods tested and verified on 2025-12-17**

---

## Quick Summary: Which Method to Use?

| Method | Best For | Scale | Requires EC2/Compute? | Tested |
|--------|----------|-------|----------------------|--------|
| `aws s3 sync` | Simple ad-hoc transfers | < 1M objects | ✅ Yes (EC2/Lambda/Local) | ✅ |
| Gateway Endpoint | Private subnet S3 access | Any | ✅ Yes (EC2 in private subnet) | ✅ |
| Interface Endpoint | Strict network isolation | Any | ✅ Yes (EC2 in private subnet) | ✅ |
| S3 Batch Operations | Large migrations | Billions | ❌ No (Serverless) | ✅ |
| S3 Replication | Continuous sync | Any | ❌ No (Serverless) | ✅ |
| DataSync | Scheduled recurring sync | Large | ❌ No (Managed service) | ✅ |

---

## Comprehensive Cost Comparison

### Full Cost Breakdown by Method

| Method | Service Cost | Data Transfer | S3 Requests | Compute Required | Total for 1TB/1M objects |
|--------|-------------|---------------|-------------|------------------|--------------------------|
| `aws s3 sync` (same region) | FREE | FREE | $5 PUT | EC2: ~$0.10/hr | ~$5 + compute |
| `aws s3 sync` (cross region) | FREE | $20/TB | $5 PUT | EC2: ~$0.10/hr | ~$25 + compute |
| Gateway Endpoint | **FREE** | FREE (same region) | $5 PUT | EC2 required | ~$5 + compute |
| Interface Endpoint | $7.20/mo/AZ + $10/TB | FREE (same region) | $5 PUT | EC2 required | ~$12 + compute |
| S3 Batch Operations | $0.25/M objects | FREE (same region) | $5 PUT | ❌ None | ~$5.25 |
| S3 Replication | FREE | FREE (same region) | $5 PUT | ❌ None | ~$5 |
| DataSync | $12.50/TB | FREE (same region) | Included | ❌ None | ~$12.50 |
| NAT Gateway | $32.40/mo + $45/TB | $45/TB | $5 PUT | EC2 required | ~$95 ⚠️ Avoid |

### Data Transfer Costs

| Scenario | Cost |
|----------|------|
| Same Region (any method) | **FREE** |
| Cross Region (within AWS) | ~$0.02/GB ($20/TB) |
| To Internet | ~$0.09/GB ($90/TB) |
| S3 Transfer Acceleration | +$0.04/GB |

### S3 Request Pricing (us-east-1)

| Request Type | Cost |
|--------------|------|
| PUT, COPY, POST, LIST | $0.005 per 1,000 requests |
| GET, SELECT | $0.0004 per 1,000 requests |
| DELETE | FREE |

### Infrastructure Costs (if compute required)

| Resource | Cost | Notes |
|----------|------|-------|
| EC2 t3.micro | $0.0104/hr (~$7.50/mo) | Minimum for aws s3 sync |
| EC2 t3.medium | $0.0416/hr (~$30/mo) | Recommended for large transfers |
| NAT Gateway | $0.045/hr + $0.045/GB | ⚠️ Expensive - avoid if possible |
| VPC Peering | FREE (same region) | $0.01/GB cross-region |
| Transit Gateway | $0.05/hr + $0.02/GB | For complex multi-VPC |

---

## Method Comparison Tables

### By Scale

| Objects Count | Recommended Method | Reason |
|---------------|-------------------|--------|
| < 10,000 | `aws s3 sync` | Simple, fast |
| 10K - 1M | `aws s3 sync --only-show-errors` | Reduce output overhead |
| 1M - 100M | S3 Batch Operations | Built-in retry, tracking |
| > 100M | S3 Batch Operations + S3 Inventory | Automatic manifest generation |
| Continuous | S3 Replication | Real-time sync |
| Scheduled | DataSync | Managed scheduling |

### By Network Requirement

| Requirement | Solution | Cost |
|-------------|----------|------|
| Public subnet with IGW | Direct access | FREE |
| Private subnet (no IGW) | Gateway Endpoint | **FREE** |
| Security group control needed | Interface Endpoint | $0.01/hr/AZ |
| Restrict to specific VPC | Bucket policy + `aws:sourceVpc` | FREE |
| Cross-VPC same region | VPC Peering + Endpoint | FREE |
| Cross-VPC cross region | Transit Gateway | $0.05/hr + $0.02/GB |

### By Use Case

| Use Case | Best Method | Why |
|----------|-------------|-----|
| One-time migration (small) | `aws s3 sync` | Simple, no setup |
| One-time migration (large) | S3 Batch Operations | Serverless, scalable |
| Continuous backup | S3 Replication | Automatic, real-time |
| Daily/weekly sync | DataSync | Scheduled, managed |
| Private network required | Gateway Endpoint | Free, secure |
| Compliance/audit needs | Interface Endpoint | Security group logs |

### Feature Comparison

| Feature | aws s3 sync | S3 Batch Ops | S3 Replication | DataSync |
|---------|-------------|--------------|----------------|----------|
| Serverless | ❌ | ✅ | ✅ | ✅ |
| Auto retry | ❌ | ✅ | ✅ | ✅ |
| Progress tracking | Limited | Dashboard | CloudWatch | Dashboard |
| Completion report | ❌ | ✅ | ❌ | ✅ |
| Cross-account | ✅ | ✅ | ✅ | ✅ |
| Cross-region | ✅ | ✅ | ✅ | ✅ |
| Scheduling | Manual/cron | Manual | Continuous | Built-in |
| Max scale | ~1M objects | Billions | Unlimited | Large |

---

## Decision Tree

```
Need to transfer S3 data?
│
├─ One-time or Continuous?
│   ├─ One-time
│   │   ├─ < 1M objects → aws s3 sync (need EC2/compute)
│   │   └─ > 1M objects → S3 Batch Operations (serverless)
│   │
│   └─ Continuous
│       ├─ Real-time → S3 Replication (serverless)
│       └─ Scheduled → DataSync (managed)
│
├─ Private subnet (no IGW)?
│   ├─ Yes → Gateway Endpoint (FREE) + EC2
│   └─ No → Direct access works
│
├─ Need security group control?
│   ├─ Yes → Interface Endpoint ($0.01/hr/AZ)
│   └─ No → Gateway Endpoint (FREE)
│
└─ Cross Region?
    ├─ Yes → Add ~$0.02/GB data transfer cost
    └─ No → No data transfer cost
```

---

## Transfer Scenarios Matrix

| Scenario | Method | Network Path | Compute Required | Cost (1TB) |
|----------|--------|--------------|------------------|------------|
| Same Account, Same Region | Direct copy | AWS internal | EC2/Lambda | ~$5 |
| Same Account, Cross Region | Direct copy | AWS backbone | EC2/Lambda | ~$25 |
| Cross Account, Same Region | Bucket policy | AWS internal | EC2/Lambda | ~$5 |
| Cross Account, Cross Region | Bucket policy | AWS backbone | EC2/Lambda | ~$25 |
| Private subnet | Gateway Endpoint | Private IPs | EC2 | ~$5 |
| Strict isolation | Interface Endpoint | Private IPs | EC2 | ~$17 |
| Large migration (serverless) | S3 Batch Operations | AWS internal | None | ~$5.25 |
| Continuous sync (serverless) | S3 Replication | AWS internal | None | ~$5 |

---

## ✅ Test Results Summary (2025-12-17)

### S3 Batch Operations
| Item | Value |
|------|-------|
| Job ID | `64d1c633-c4ab-46e8-9906-dbe499ee6af5` |
| Region | ap-northeast-1 |
| Source | ss-cross-acct-test (Account 767397876447) |
| Destination | ss-cross-acct-accept (Account 127214164714) |
| Status | ✅ Complete (2/2 tasks) |

**Key Lesson:** Batch job must be created in same region as source bucket.

### S3 Replication
| Item | Value |
|------|-------|
| Source | ss-cross-acct-test (Account 767397876447) |
| Destination | ss-cross-acct-accept (Account 127214164714) |
| Prefix | `replicate/` |
| Status | ✅ COMPLETED |

**Key Lesson:** Versioning required on both buckets; destination bucket policy must allow replication.

### DataSync
| Item | Value |
|------|-------|
| Task ARN | `task-0467e69aaf92a1a2b` |
| Source | ss-cross-acct-test/datasync-test/ |
| Destination | ss-cross-acct-accept/datasync-dest/ |
| Status | ✅ SUCCESS (1 file, 50 bytes) |

**Key Lesson:** DataSync role needs `s3:DeleteObject` permission; creates `.aws-datasync/` metadata folder.

### Gateway Endpoint
| Item | Value |
|------|-------|
| Endpoint ID | vpce-00df0cd8226408340 |
| VPC | vpc-0ca8540233182736a |
| Status | ✅ Available |

### Interface Endpoint
| Item | Value |
|------|-------|
| Endpoint ID | vpce-042a2b36f62dbd782 |
| ENI IP | 10.0.0.93 |
| Status | ✅ Available |

---

## Best Practices

1. **Use Gateway Endpoints** - Free and keeps traffic private
2. **Avoid NAT Gateway for S3** - Use Gateway Endpoint instead (saves ~$90/TB)
3. **Use S3 Batch Operations** - For millions of objects (serverless, no EC2 needed)
4. **Consider S3 Replication** - For ongoing sync (serverless, automatic)
5. **Least privilege IAM** - Only grant required S3 actions
6. **Bucket policies** - Restrict by VPC/endpoint/account/org
7. **Enable encryption** - SSE-S3 or SSE-KMS at rest, TLS in transit
8. **Enable logging** - S3 access logs and CloudTrail

---

# Detailed Implementation Guides

## Scenario 1: Same Account, Same Region

Simplest case - no special configuration needed.

```bash
# Direct copy
aws s3 sync s3://source-bucket/ s3://destination-bucket/

# With progress
aws s3 sync s3://source-bucket/ s3://destination-bucket/ --progress
```

**Cost:** PUT requests only (~$5 per million objects)

---

## Scenario 2: Cross Account, Same Region

### Architecture
```
Account A (Source)                    Account B (Destination)
┌─────────────────┐                  ┌─────────────────┐
│  source-bucket  │ ───────────────> │  dest-bucket    │
│                 │   Cross-account  │                 │
│  IAM Role/User  │   bucket policy  │  Bucket Policy  │
└─────────────────┘                  └─────────────────┘
```

### Step 1: Destination Bucket Policy (Account B)
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowCrossAccountAccess",
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::ACCOUNT_A_ID:root"},
    "Action": ["s3:GetObject", "s3:PutObject", "s3:ListBucket"],
    "Resource": [
      "arn:aws:s3:::dest-bucket",
      "arn:aws:s3:::dest-bucket/*"
    ]
  }]
}
```

### Step 2: Source IAM Policy (Account A)
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

### Step 3: Execute Transfer
```bash
aws s3 sync s3://source-bucket/ s3://dest-bucket/ --profile source-account
```

**Cost:** PUT requests only (~$5 per million objects)

---

## Scenario 3: Private Subnet with Gateway Endpoint

### Architecture
```
VPC (10.0.0.0/16)
┌─────────────────────────────────────────────────┐
│  Private Subnet (no IGW)                        │
│  ┌─────────────┐      ┌──────────────────────┐  │
│  │    EC2      │ ───> │  Gateway Endpoint    │  │
│  │ 10.0.1.10   │      │  (vpce-xxx)          │  │
│  └─────────────┘      └──────────┬───────────┘  │
│                                  │              │
└──────────────────────────────────│──────────────┘
                                   │
                                   ▼
                          ┌─────────────────┐
                          │   S3 Service    │
                          │ (AWS Internal)  │
                          └─────────────────┘
```

### Create Gateway Endpoint
```bash
# Get route table ID
RT_ID=$(aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=vpc-xxx" \
  --query 'RouteTables[0].RouteTableId' --output text)

# Create Gateway Endpoint
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-xxx \
  --service-name com.amazonaws.REGION.s3 \
  --route-table-ids $RT_ID \
  --vpc-endpoint-type Gateway
```

**Cost:** **FREE** (Gateway Endpoints have no charge)

---

## Scenario 4: Interface Endpoint (PrivateLink)

Use when you need security group control over S3 traffic.

### Create Interface Endpoint
```bash
# Create security group
SG_ID=$(aws ec2 create-security-group \
  --group-name s3-endpoint-sg \
  --description "S3 Interface Endpoint" \
  --vpc-id vpc-xxx \
  --query 'GroupId' --output text)

# Allow HTTPS from VPC
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp --port 443 \
  --cidr 10.0.0.0/16

# Create Interface Endpoint
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-xxx \
  --service-name com.amazonaws.REGION.s3 \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-xxx \
  --security-group-ids $SG_ID
```

**Cost:** $0.01/hr/AZ (~$7.20/month) + $0.01/GB processed

---

## Scenario 5: S3 Batch Operations (Large Scale)

Best for millions/billions of objects - **no EC2 required**.

### Step 1: Create IAM Role
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

### Step 2: Create Manifest
```bash
# Option A: Simple CSV
echo "source-bucket,path/to/object1.txt" > manifest.csv
echo "source-bucket,path/to/object2.txt" >> manifest.csv
aws s3 cp manifest.csv s3://source-bucket/manifest/

# Option B: Use S3 Inventory for large buckets
```

### Step 3: Create and Run Job
```bash
ETAG=$(aws s3api head-object --bucket source-bucket --key manifest/manifest.csv --query 'ETag' --output text | tr -d '"')

aws s3control create-job \
  --account-id ACCOUNT_ID \
  --operation '{"S3PutObjectCopy":{"TargetResource":"arn:aws:s3:::dest-bucket"}}' \
  --manifest "{\"Spec\":{\"Format\":\"S3BatchOperations_CSV_20180820\",\"Fields\":[\"Bucket\",\"Key\"]},\"Location\":{\"ObjectArn\":\"arn:aws:s3:::source-bucket/manifest/manifest.csv\",\"ETag\":\"$ETAG\"}}" \
  --report '{"Bucket":"arn:aws:s3:::source-bucket","Format":"Report_CSV_20180820","Enabled":true,"Prefix":"batch-reports","ReportScope":"AllTasks"}' \
  --priority 10 \
  --role-arn arn:aws:iam::ACCOUNT_ID:role/S3BatchOperationsRole \
  --region REGION \
  --no-confirmation-required
```

**Cost:** $0.25 per million objects + PUT requests

---

## Scenario 6: S3 Replication (Continuous Sync)

Best for ongoing synchronization - **no EC2 required**.

### Step 1: Enable Versioning
```bash
aws s3api put-bucket-versioning --bucket source-bucket --versioning-configuration Status=Enabled
aws s3api put-bucket-versioning --bucket dest-bucket --versioning-configuration Status=Enabled
```

### Step 2: Create Replication Role
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

### Step 3: Configure Replication
```bash
cat > replication.json << 'EOF'
{
  "Role": "arn:aws:iam::ACCOUNT_ID:role/S3ReplicationRole",
  "Rules": [{
    "ID": "ReplicateAll",
    "Status": "Enabled",
    "Priority": 1,
    "Filter": {},
    "Destination": {
      "Bucket": "arn:aws:s3:::dest-bucket",
      "Account": "DEST_ACCOUNT_ID",
      "AccessControlTranslation": {"Owner": "Destination"}
    },
    "DeleteMarkerReplication": {"Status": "Enabled"}
  }]
}
EOF

aws s3api put-bucket-replication --bucket source-bucket --replication-configuration file://replication.json
```

**Cost:** PUT requests only (same as manual copy)

---

## Scenario 7: DataSync (Scheduled Sync)

Best for recurring transfers - **no EC2 required**.

### Create DataSync Task
```bash
# Create source location
SOURCE_LOC=$(aws datasync create-location-s3 \
  --s3-bucket-arn arn:aws:s3:::source-bucket \
  --s3-config '{"BucketAccessRoleArn":"arn:aws:iam::ACCOUNT_ID:role/DataSyncS3Role"}' \
  --query 'LocationArn' --output text)

# Create destination location
DEST_LOC=$(aws datasync create-location-s3 \
  --s3-bucket-arn arn:aws:s3:::dest-bucket \
  --s3-config '{"BucketAccessRoleArn":"arn:aws:iam::ACCOUNT_ID:role/DataSyncS3Role"}' \
  --query 'LocationArn' --output text)

# Create task
aws datasync create-task \
  --source-location-arn $SOURCE_LOC \
  --destination-location-arn $DEST_LOC \
  --name "S3SyncTask"
```

**Cost:** $0.0125/GB transferred

---

## Cleanup Commands

```bash
# Delete VPC Endpoints
aws ec2 delete-vpc-endpoints --vpc-endpoint-ids vpce-xxx

# Delete IAM Roles
aws iam delete-role-policy --role-name S3BatchOperationsRole --policy-name S3BatchPolicy
aws iam delete-role --role-name S3BatchOperationsRole

# Delete Replication Configuration
aws s3api delete-bucket-replication --bucket source-bucket

# Delete DataSync Task
aws datasync delete-task --task-arn arn:aws:datasync:REGION:ACCOUNT:task/task-xxx
```

---

## References

- [S3 Batch Operations](https://docs.aws.amazon.com/AmazonS3/latest/userguide/batch-ops.html)
- [S3 Replication](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication.html)
- [VPC Endpoints for S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/privatelink-interface-endpoints.html)
- [DataSync](https://docs.aws.amazon.com/datasync/latest/userguide/what-is-datasync.html)
- [S3 Pricing](https://aws.amazon.com/s3/pricing/)
