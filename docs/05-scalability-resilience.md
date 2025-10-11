# Scalability, Resilience & Observability

This section details how the proposed architecture handles growth, failures, and operational visibility.

---

## Scalability Strategy

### Current vs Projected Growth

| Metric | Current (150 clients) | 1 Year (450 clients, 3x) | 3 Years (1,500 clients, 10x) |
|--------|-----------------------|---------------------------|-------------------------------|
| **Users** | 10,000 | 30,000 | 100,000 |
| **Uploads/day** | 5,000 | 15,000 | 50,000 |
| **Storage** | 500 GB | 1.5 TB | 5 TB |
| **API Requests/min** | 500 | 1,500 | 5,000 |

**Goal**: Architecture handles 3x growth (1 year) without code changes, and 10x growth (3 years) with minimal changes.

---

## Component Scalability

### 1. API Layer (ECS Fargate)

**Current Configuration:**
- 2 tasks (minimum)
- 2 vCPU, 4 GB RAM per task
- Can handle ~1,000 req/min total

**Auto-Scaling Policy:**

```yaml
Scaling Metric: CPU Utilization
Target: 70%
Scale Out: +2 tasks when >70% for 2 minutes
Scale In: -1 task when <40% for 5 minutes
Min: 2 tasks
Max: 10 tasks
Cooldown: 3 minutes
```

**Alternative Scaling Metric** (for upload-heavy workload):
```yaml
Scaling Metric: SQS Queue Depth (ApproximateNumberOfMessagesVisible)
Target: 100 messages
Scale Out: +1 task per 100 messages in queue
```

**Capacity Projection:**

| Scenario | Tasks | Capacity (req/min) | Cost/month |
|----------|-------|-------------------|------------|
| Current (150 clients) | 2 | 1,000 | $130 |
| 3x growth (450 clients) | 4-6 | 3,000 | $260-390 |
| 10x growth (1,500 clients) | 10 | 5,000 | $650 |

**Why ECS over Lambda for API?**
- API needs persistent connections (WebSocket)
- Sub-100ms response time (Lambda cold start = 1-2s)
- Complex request routing (better fit for Express/NestJS)

**Scaling Beyond 10 Tasks:**
- Vertical scaling: Upgrade to 4 vCPU, 8 GB RAM per task
- Read replicas: Offload read-heavy queries (reports, analytics)
- Caching: Redis for frequent queries (project lists, user profiles)

---

### 2. Database Layer (RDS Aurora PostgreSQL)

**Current Configuration:**
- Instance: db.r6g.large (2 vCPU, 16 GB RAM)
- Multi-AZ: Automatic failover to standby
- Storage: 500 GB (auto-scaling to 128 TB)

**Scaling Strategy:**

**Vertical Scaling (Up to 10x):**
```
db.r6g.large  (2 vCPU, 16 GB)  → db.r6g.xlarge  (4 vCPU, 32 GB)  [2x]
db.r6g.xlarge (4 vCPU, 32 GB)  → db.r6g.2xlarge (8 vCPU, 64 GB)  [4x]
db.r6g.2xlarge (8 vCPU, 64 GB) → db.r6g.4xlarge (16 vCPU, 128 GB) [8x]
```

**Read Replicas (Horizontal Scaling for Reads):**
- Create 1-3 read replicas
- Route read-only queries (reports, analytics, dashboards) to replicas
- Write queries (upload status updates) to primary
- Reduces load on primary by 60-80%

**Connection Pooling:**
- NestJS connection pool: 10 connections per ECS task
- RDS max connections: ~600 for db.r6g.large
- With 10 ECS tasks: 100 connections total (well within limit)

**Query Optimization:**
- Index on `uploads(tenant_id, project_id, created_at)` for fast filtering
- Index on `uploads(status)` for processing queue queries
- Materialized views for complex reports (refresh hourly)

**Capacity Projection:**

| Scenario | Instance | Read Replicas | Connections | Cost/month |
|----------|----------|---------------|-------------|------------|
| Current | db.r6g.large | 0 | 100 | $350 |
| 3x growth | db.r6g.xlarge | 1 | 200 | $800 |
| 10x growth | db.r6g.2xlarge | 2 | 400 | $1,800 |

---

### 3. Cache Layer (ElastiCache Redis)

**Current Configuration:**
- Instance: cache.t3.medium (2 vCPU, 3.09 GB)
- Cache Hit Ratio Target: >80%

**Cached Data:**
- Upload status (TTL: 30 seconds) → Reduces DB load from polling
- User sessions (TTL: 1 hour)
- API responses (project lists, user profiles) (TTL: 5 minutes)
- Rate limiting counters (TTL: 1 minute)

**Scaling Strategy:**

**Vertical Scaling:**
```
cache.t3.medium (3 GB)  → cache.r6g.large  (13.5 GB)  [4x]
cache.r6g.large (13.5 GB) → cache.r6g.xlarge (27 GB)   [8x]
```

**Redis Cluster Mode:**
- At 10x scale, enable cluster mode (sharding)
- Partition keys by tenant_id
- 3-5 shards = 5x capacity

**Cache Invalidation Strategy:**
- Uploads: Invalidate on status change (Lambda → Redis DEL)
- Projects: Invalidate on update (API → Redis DEL)
- Users: Invalidate on profile change

---

### 4. Storage Layer (S3)

**Scalability**: ✅ **Infinite** (AWS managed)

**Performance Optimization:**
- S3 Transfer Acceleration: Uses CloudFront edge locations for faster uploads
- Multipart upload: Parallel upload of large files
- Throughput: 3,500 PUT/s per prefix (no bottleneck)

**Cost Optimization:**
- S3 Intelligent-Tiering: Automatically moves infrequently accessed files to cheaper storage
  - Hot tier (frequent access): $0.023/GB/month
  - Cool tier (infrequent access): $0.0125/GB/month
  - Archive (rarely accessed): $0.004/GB/month
- Lifecycle policy: Move files >90 days old to Glacier ($0.004/GB/month → $0.0036/GB/month)

**Capacity Projection:**

| Scenario | Uploads/month | Storage (cumulative) | Cost/month (storage only) |
|----------|---------------|----------------------|---------------------------|
| Current | 150,000 (5K/day) | 500 GB → 750 GB (+250 GB) | $17 → $25 |
| 3x growth | 450,000 (15K/day) | 1.5 TB → 2.25 TB (+750 GB) | $50 → $75 |
| 10x growth | 1,500,000 (50K/day) | 5 TB → 7.5 TB (+2.5 TB) | $167 → $250 |

*Note: With Intelligent-Tiering, costs reduce by 30-40% over time as old files move to cool/archive tiers.*

---

### 5. Processing Layer (Lambda)

**Scalability**: ✅ **Auto-scaling** (0 to 1,000 concurrent executions)

**Current Configuration:**
- Reserved concurrency: 20 (keeps functions warm)
- Burst concurrency: 100 (during spikes)
- Provisioned concurrency: 0 (not needed)

**Lambda Scaling Behavior:**

| Upload Rate | Lambda Invocations/min | Concurrent Executions | Auto-Scaling Action |
|-------------|------------------------|----------------------|---------------------|
| **Steady (200/hour)** | ~15 | 3-5 | Reuse warm containers |
| **Spike (2,000/hour)** | ~130 | 20-40 | Scale up in 1-2 min |
| **Extreme (5,000/hour)** | ~330 | 50-100 | Scale up to reserved limit |

**Cold Start Mitigation:**
- Reserved concurrency: 20 functions stay warm (sub-100ms invocation)
- Lambda SnapStart: Faster cold starts for Java/Node.js (reduces 1-2s → 200ms)
- CloudWatch Events: Ping functions every 5 minutes (keeps warm)

**Capacity Projection:**

| Scenario | Invocations/month | Compute (GB-seconds) | Cost/month |
|----------|-------------------|----------------------|------------|
| Current (5K uploads/day) | 600,000 (4 functions × 150K uploads) | 166,000 | $70 |
| 3x growth (15K/day) | 1,800,000 | 500,000 | $210 |
| 10x growth (50K/day) | 6,000,000 | 1,666,000 | $700 |

---

## Resilience Patterns

### 1. High Availability (Multi-AZ)

**Components with Multi-AZ:**

| Component | Multi-AZ Strategy | RPO (Data Loss) | RTO (Recovery Time) |
|-----------|-------------------|-----------------|---------------------|
| **ECS Fargate** | Tasks in 2+ AZs (us-east-1a, 1b) | 0 (stateless) | <1 min (health check + launch) |
| **RDS Aurora** | Primary + Standby in separate AZs | 0 (sync replication) | <60 seconds (automatic failover) |
| **ElastiCache** | Primary + Read replicas in separate AZs | <1 min (async replication) | <1 min (promote replica) |
| **S3** | Automatically replicated across AZs | 0 | 0 (no downtime) |
| **Lambda** | Automatically spans AZs | 0 | 0 (no downtime) |

**Single Point of Failure Analysis:**
- ❌ ALB: In single region (us-east-1). Mitigated by Route 53 health checks (can failover to backup region if needed)
- ❌ CloudFront: Global service (AWS managed, 99.99% SLA)
- ✅ All compute and data layers: Multi-AZ

---

### 2. Retry Logic & Circuit Breakers

**Client → API:**
```javascript
// Exponential backoff for API failures
const retryPolicy = {
  maxRetries: 5,
  backoff: exponential, // 1s, 2s, 4s, 8s, 16s
  retryConditions: [
    (error) => error.status >= 500,  // Server errors
    (error) => error.code === 'ECONNREFUSED',  // Network errors
  ],
};
```

**API → S3 (Presigned URL generation):**
```typescript
// Circuit breaker pattern (prevent cascading failures)
const circuitBreaker = {
  errorThreshold: 50, // Open circuit if 50% of requests fail
  timeout: 10000,     // 10 second timeout
  resetTimeout: 30000, // Try again after 30 seconds
};

try {
  const url = await s3.getSignedUrl('putObject', params);
} catch (error) {
  if (circuitBreaker.isOpen()) {
    // Fail fast (don't even try S3)
    throw new ServiceUnavailableError('S3 temporarily unavailable');
  }
  // Otherwise, retry with exponential backoff
}
```

**SQS → Lambda:**
- Automatic retries: 3 attempts (SQS visibility timeout)
- Dead Letter Queue (DLQ): After 3 failures, message moves to DLQ
- Manual inspection: Ops team reviews DLQ messages, fixes issue, re-queues

**Lambda → RDS:**
```typescript
// Retry transient errors (connection timeouts, deadlocks)
const dbRetryPolicy = {
  maxRetries: 3,
  backoff: exponential,
  retryConditions: [
    (error) => error.code === 'ECONNREFUSED',
    (error) => error.code === 'ER_LOCK_DEADLOCK',
  ],
};
```

---

### 3. Graceful Degradation

**If S3 is unavailable:**
- API returns 503 Service Unavailable
- Client shows: "Uploads temporarily unavailable. Please try again in a few minutes."
- Don't fail entire request (user can still browse projects, view documents)

**If Lambda processing is failing:**
- Uploads complete successfully (files in S3)
- Status remains "UPLOADED" (not "COMPLETED")
- User sees: "Processing..." in UI
- Background job retries processing when Lambda recovers

**If RDS is slow:**
- Cache hit ratio increases (Redis serves stale data)
- Read replica handles read queries
- Write queries queue in ECS (connection pool waits)

**If Redis is down:**
- Fallback to database for upload status queries
- Slower (50ms → 200ms), but functional
- Session falls back to JWT (stateless auth)

---

### 4. Data Backup & Recovery

**RDS Aurora Backups:**
- Automated snapshots: Daily (retained 7 days)
- Point-in-time recovery: Restore to any second in last 7 days
- Manual snapshots: Before major migrations (retained 30 days)

**S3 Versioning:**
- Enabled on buildpeer-uploads bucket
- Every object has version history
- Accidental delete: Can restore previous version
- Malicious upload: Can revert to safe version

**Disaster Recovery Procedures:**

**Scenario 1: Corrupted Database**
```bash
# Restore RDS from automated snapshot (latest)
aws rds restore-db-cluster-to-point-in-time \
  --source-db-cluster-identifier buildpeer-prod \
  --target-db-cluster-identifier buildpeer-prod-restore \
  --restore-to-time 2024-10-10T12:00:00Z

# Update ECS task definition to point to restored cluster
# Downtime: ~15 minutes
```

**Scenario 2: Accidental S3 Object Deletion**
```bash
# List object versions
aws s3api list-object-versions \
  --bucket buildpeer-uploads \
  --prefix tenant-X/project-Y/

# Restore specific version
aws s3api copy-object \
  --bucket buildpeer-uploads \
  --copy-source buildpeer-uploads/path/to/file?versionId=VERSION_ID \
  --key path/to/file

# Downtime: 0 (no impact on service)
```

---

## Observability

### 1. Logging Strategy

**Structured JSON Logging:**
```json
{
  "timestamp": "2024-10-10T14:32:15Z",
  "level": "info",
  "service": "api",
  "tenantId": "tenant-uuid",
  "userId": "user-uuid",
  "requestId": "req-uuid",
  "message": "Upload initiated",
  "context": {
    "uploadId": "upload-uuid",
    "projectId": "project-uuid",
    "filename": "foto-obra-1.jpg",
    "size": 3145728
  }
}
```

**Log Aggregation:**
- All services → CloudWatch Logs
- Log groups:
  - `/ecs/buildpeer-api` (API logs)
  - `/aws/lambda/image-processor` (Lambda logs)
  - `/aws/rds/cluster/buildpeer-prod` (Database logs)

**Log Retention:**
- Hot tier: 30 days (CloudWatch)
- Cold tier: 1 year (S3 archive)

**Query Examples (CloudWatch Insights):**

```sql
-- Find all failed uploads for a tenant
fields @timestamp, uploadId, filename, error
| filter tenantId = "tenant-uuid" and status = "FAILED"
| sort @timestamp desc
| limit 100

-- Upload success rate by hour
fields @timestamp, status
| stats count(*) as total,
        sum(status = "COMPLETED") as success
        by bin(@timestamp, 1h)
| sort @timestamp desc
```

---

### 2. Metrics & Dashboards

**Key Metrics (CloudWatch Custom Metrics):**

**Business Metrics:**
- `upload_success_rate`: % of uploads that complete successfully
- `upload_latency_p95`: 95th percentile upload time (initiate → complete)
- `upload_throughput`: Uploads per minute

**Technical Metrics:**
- `api_latency_p50/p95/p99`: API response times
- `lambda_duration_avg`: Average Lambda execution time
- `lambda_errors_count`: Number of Lambda errors
- `sqs_queue_depth`: Number of messages in SQS queue
- `ecs_cpu_utilization`: ECS task CPU usage
- `ecs_memory_utilization`: ECS task memory usage
- `rds_connections`: Number of active DB connections
- `redis_hit_rate`: Cache hit percentage

**CloudWatch Dashboard (Real-Time):**

```
┌─────────────────────────────────────────────────┐
│          BuildPeer Production Dashboard         │
├─────────────────────────────────────────────────┤
│                                                 │
│  Upload Success Rate (Last 1 Hour)             │
│  ████████████████████████░░ 98.5%               │
│                                                 │
│  API Latency (P95)                              │
│  ▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬ 120ms                         │
│                                                 │
│  ECS Tasks Running                              │
│  2 / 10 max                                     │
│                                                 │
│  Lambda Invocations (Last 5 Min)               │
│  ▁▂▃▅▇█▇▅▃▂▁ 45/min                             │
│                                                 │
│  SQS Queue Depth                                │
│  ▬▬▬▬▬ 12 messages                              │
│                                                 │
│  RDS Connections                                │
│  85 / 600 max (14%)                             │
│                                                 │
│  Redis Hit Rate                                 │
│  ████████████████████ 85%                       │
│                                                 │
└─────────────────────────────────────────────────┘
```

**Per-Tenant Dashboards:**
- Separate dashboard for each constructora client
- Shows upload activity, storage usage, error rates
- Useful for customer support ("Why are uploads slow for Client X?")

---

### 3. Distributed Tracing (X-Ray)

**End-to-End Request Tracing:**

```
Request ID: req-abc123
Total Duration: 45,230ms

API Gateway (initiate upload)      [  120ms ]
  ├─ Auth validation               [   15ms ]
  ├─ DB query (check project)      [   45ms ]
  ├─ Generate presigned URL (S3)   [   25ms ]
  └─ Response to client            [   10ms ]

Client uploads to S3               [30,000ms]  ← User's network

S3 → EventBridge → SQS             [  2,500ms]

Lambda (Image Processor)           [ 12,500ms]
  ├─ Download from S3              [  1,200ms]
  ├─ Generate thumbnails           [  8,500ms]
  ├─ Upload thumbnails to S3       [  2,300ms]
  └─ Update DB                     [    500ms]

SNS notification to client         [    110ms]
```

**Benefit**: Quickly identify bottlenecks (e.g., "thumbnail generation taking 8.5s, too slow")

---

### 4. Alerting

**Critical Alarms (PagerDuty):**

| Alarm | Threshold | Action |
|-------|-----------|--------|
| Upload success rate < 95% | 5 min | Page on-call engineer |
| API error rate > 5% | 5 min | Page on-call engineer |
| Lambda errors > 10/min | 5 min | Page on-call engineer |
| RDS connections > 500 | 1 min | Page on-call engineer |
| SQS DLQ messages > 10 | 1 min | Page on-call engineer |

**Warning Alarms (Slack):**

| Alarm | Threshold | Action |
|-------|-----------|--------|
| Upload success rate < 98% | 10 min | Notify #ops channel |
| API P95 latency > 500ms | 10 min | Notify #ops channel |
| ECS CPU > 80% | 5 min | Notify #ops channel (auto-scale soon) |
| Redis hit rate < 70% | 10 min | Notify #ops channel (cache tuning) |

**Alarm Fatigue Prevention:**
- Only alert on **actionable** issues (not just "high CPU" if auto-scaling handles it)
- Escalation: Warning (Slack) → Critical (PagerDuty) if not resolved in 30 min
- Aggregate similar alarms (don't page for every failed upload, only if rate > threshold)

---

## Security

### 1. Network Security

**VPC Architecture:**
```
┌─────────────────────────────────────┐
│               VPC                    │
│                                     │
│  ┌──────────────────────────────┐  │
│  │    Public Subnet (AZ-1)      │  │
│  │  - ALB                       │  │
│  │  - NAT Gateway               │  │
│  └──────────────────────────────┘  │
│                                     │
│  ┌──────────────────────────────┐  │
│  │    Private Subnet (AZ-1)     │  │
│  │  - ECS Tasks                 │  │
│  │  - RDS Primary               │  │
│  │  - ElastiCache               │  │
│  │  - Lambda (VPC mode)         │  │
│  └──────────────────────────────┘  │
│                                     │
│  ┌──────────────────────────────┐  │
│  │    Private Subnet (AZ-2)     │  │
│  │  - ECS Tasks                 │  │
│  │  - RDS Standby               │  │
│  │  - ElastiCache Replica       │  │
│  └──────────────────────────────┘  │
└─────────────────────────────────────┘
```

**Security Groups (Least Privilege):**

| Security Group | Inbound Rules | Purpose |
|----------------|---------------|---------|
| **ALB-SG** | 443 from CloudFront only | Only accept HTTPS from CDN |
| **ECS-SG** | 3000 from ALB-SG only | Only accept from load balancer |
| **RDS-SG** | 5432 from ECS-SG, Lambda-SG | Only accept from API and Lambda |
| **Redis-SG** | 6379 from ECS-SG | Only accept from API |

**Egress Controls:**
- ECS → Internet (via NAT Gateway): Only for API calls to third-party services
- Lambda → Internet: Disabled (uses VPC endpoints for AWS services)

---

### 2. IAM Roles (Least Privilege)

**ECS Task Role:**
```json
{
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:GetSignedUrl"
      ],
      "Resource": "arn:aws:s3:::buildpeer-uploads/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "sqs:SendMessage",
        "sns:Publish"
      ],
      "Resource": [
        "arn:aws:sqs:us-east-1:123456:upload-queue",
        "arn:aws:sns:us-east-1:123456:upload-completed"
      ]
    }
  ]
}
```

**Lambda Execution Role:**
```json
{
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::buildpeer-uploads/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "rds:DescribeDBInstances",
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "*"
    }
  ]
}
```

---

### 3. Encryption

**At Rest:**
- S3: Server-Side Encryption (SSE-KMS)
- RDS: Encryption enabled (AES-256)
- EBS volumes: Encrypted (for ECS tasks)
- Secrets: AWS Secrets Manager (encrypted with KMS)

**In Transit:**
- Client ↔ CloudFront: TLS 1.3
- CloudFront ↔ ALB: TLS 1.3
- ALB ↔ ECS: TLS 1.2 (internal VPC)
- ECS ↔ RDS: TLS 1.2
- ECS ↔ S3: TLS 1.2 (HTTPS)

---

### 4. Audit Logging

**CloudTrail:**
- All AWS API calls logged
- S3 bucket policy changes
- IAM role changes
- RDS configuration changes

**Application Audit Log:**
```json
{
  "event": "upload_initiated",
  "actor": { "userId": "user-uuid", "role": "empleado", "ip": "192.168.1.1" },
  "target": { "tenantId": "tenant-uuid", "projectId": "project-uuid" },
  "action": "create_upload",
  "result": "success",
  "timestamp": "2024-10-10T14:32:15Z"
}
```

**Compliance:**
- GDPR: User data deletion (cascade delete uploads when user deleted)
- SOC 2: Audit logs retained 1 year
- Data sovereignty: Mexico clients' data stored in us-east-1 (AWS US region)

---

**Next section**: Migration strategy (incremental rollout plan)
