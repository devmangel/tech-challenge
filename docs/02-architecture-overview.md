# Architecture Overview

## System Architecture

This section provides a comprehensive view of the proposed backend architecture, focusing on the upload handling system while showing how it integrates with the overall application.

---

## High-Level Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                              │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Constructoras (150)     Inversionistas      Empleados de Obra  │
│         │                     │                      │           │
│         └─────────────────────┼──────────────────────┘           │
│                               │                                  │
│                    ┌──────────▼───────────┐                     │
│                    │   Web + Mobile Apps  │                     │
│                    │   (React/React Native)│                    │
│                    └──────────┬───────────┘                     │
└───────────────────────────────┼──────────────────────────────────┘
                                │
                                │ HTTPS
                                │
┌───────────────────────────────▼──────────────────────────────────┐
│                        INGRESS LAYER                             │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────┐      ┌──────────────┐      ┌────────────┐     │
│  │  Route 53  │──────│  CloudFront  │──────│    WAF     │     │
│  │    (DNS)   │      │  (LATAM CDN) │      │ (Security) │     │
│  └────────────┘      └──────────────┘      └─────┬──────┘     │
│                                                    │            │
│                                          ┌─────────▼──────────┐ │
│                                          │  Application       │ │
│                                          │  Load Balancer     │ │
│                                          │  (ALB)             │ │
│                                          └─────────┬──────────┘ │
└────────────────────────────────────────────────────┼────────────┘
                                                     │
┌────────────────────────────────────────────────────▼────────────┐
│                    APPLICATION LAYER                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │         ECS Fargate Cluster (Auto-Scaling)               │  │
│  │                                                          │  │
│  │  ┌────────────────────────────────────────────────────┐ │  │
│  │  │        NestJS Monolithic Modular API              │ │  │
│  │  │                                                    │ │  │
│  │  │  Core Modules:                                    │ │  │
│  │  │  ├─ AuthModule (JWT, multi-tenant, roles)         │ │  │
│  │  │  ├─ UsersModule                                   │ │  │
│  │  │  ├─ ProjectsModule                                │ │  │
│  │  │  ├─ UploadsModule ★ CORE FOCUS                    │ │  │
│  │  │  ├─ DocumentsModule                               │ │  │
│  │  │  ├─ RFIsModule                                    │ │  │
│  │  │  └─ ReportsModule                                 │ │  │
│  │  │                                                    │ │  │
│  │  │  Shared Services:                                 │ │  │
│  │  │  ├─ Multi-tenant isolation                        │ │  │
│  │  │  ├─ AWS integrations (S3, SQS, SNS)               │ │  │
│  │  │  ├─ Redis caching                                 │ │  │
│  │  │  └─ Observability (logs, metrics, tracing)        │ │  │
│  │  └────────────────────────────────────────────────────┘ │  │
│  │                                                          │  │
│  │  Tasks: 2-10 (auto-scaling based on CPU + queue depth) │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└────────────────────────────────────┬────────────────────────────┘
                                     │
┌────────────────────────────────────▼────────────────────────────┐
│                        DATA LAYER                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────┐                  │
│  │  RDS Aurora PostgreSQL (Multi-AZ)        │                  │
│  │                                          │                  │
│  │  Primary: Write operations               │                  │
│  │  Replica: Read operations (reports)      │                  │
│  │                                          │                  │
│  │  Multi-tenant schema:                    │                  │
│  │  - Row-Level Security (RLS)              │                  │
│  │  - tenant_id in all tables               │                  │
│  │  - 150 constructoras isolated            │                  │
│  └──────────────────────────────────────────┘                  │
│                                                                 │
│  ┌──────────────────────────────────────────┐                  │
│  │  ElastiCache Redis                       │                  │
│  │                                          │                  │
│  │  Use cases:                              │                  │
│  │  - Session management                    │                  │
│  │  - Upload status cache (real-time)       │                  │
│  │  - Rate limiting                         │                  │
│  │  - API response caching                  │                  │
│  │                                          │                  │
│  │  TTL: 5 seconds - 1 hour                 │                  │
│  └──────────────────────────────────────────┘                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                  STORAGE & PROCESSING LAYER                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────┐                  │
│  │  Amazon S3 (Object Storage)              │                  │
│  │                                          │                  │
│  │  Bucket: buildpeer-uploads               │                  │
│  │  - Transfer Acceleration: ENABLED        │                  │
│  │  - Intelligent-Tiering: ENABLED          │                  │
│  │  - Versioning: ENABLED                   │                  │
│  │                                          │                  │
│  │  Structure:                              │                  │
│  │  /tenant-{id}/project-{id}/uploads/      │                  │
│  │                                          │                  │
│  │  Lifecycle:                              │                  │
│  │  - Standard: 0-90 days                   │                  │
│  │  - Glacier: 90+ days (cold storage)      │                  │
│  └──────────────┬───────────────────────────┘                  │
│                 │                                               │
│                 │ S3 Event Notification                         │
│                 │                                               │
│  ┌──────────────▼───────────────────────────┐                  │
│  │  Amazon EventBridge                      │                  │
│  │  - Event routing and filtering           │                  │
│  │  - Decouples S3 from processing          │                  │
│  └──────────────┬───────────────────────────┘                  │
│                 │                                               │
│                 │                                               │
│  ┌──────────────▼───────────────────────────┐                  │
│  │  Amazon SQS (Standard Queue)             │                  │
│  │                                          │                  │
│  │  - Buffering and backpressure handling   │                  │
│  │  - Retry logic (3 attempts)              │                  │
│  │  - Dead Letter Queue (DLQ)               │                  │
│  │  - Visibility timeout: 5 minutes         │                  │
│  └──────────────┬───────────────────────────┘                  │
│                 │                                               │
│                 │ Polling                                       │
│                 │                                               │
│  ┌──────────────▼────────────────────────────────────────────┐ │
│  │  AWS Lambda Functions (Serverless Processing)            │ │
│  │                                                           │ │
│  │  1. Image Processor Lambda:                              │ │
│  │     - Generate thumbnails (200x200, 800x800)             │ │
│  │     - Compress images (80% quality)                      │ │
│  │     - Extract EXIF data (GPS, timestamp)                 │ │
│  │     - Memory: 1024 MB, Timeout: 5 min                    │ │
│  │                                                           │ │
│  │  2. Document Processor Lambda:                           │ │
│  │     - Generate PDF preview (first page)                  │ │
│  │     - Extract text (OCR if needed)                       │ │
│  │     - Memory: 2048 MB, Timeout: 5 min                    │ │
│  │                                                           │ │
│  │  3. Security Scanner Lambda:                             │ │
│  │     - Virus/malware scanning (ClamAV)                    │ │
│  │     - File validation                                    │ │
│  │     - Memory: 3008 MB, Timeout: 5 min                    │ │
│  │                                                           │ │
│  │  4. Metadata Extractor Lambda:                           │ │
│  │     - File metadata (size, type, hash)                   │ │
│  │     - Update database status                             │ │
│  │     - Trigger notifications                              │ │
│  │     - Memory: 512 MB, Timeout: 1 min                     │ │
│  │                                                           │ │
│  │  Scaling: 0-100 concurrent executions                    │ │
│  └───────────────────────────┬───────────────────────────────┘ │
│                              │                                  │
│                              │ Update status                    │
│                              │                                  │
│  ┌───────────────────────────▼──────────────┐                  │
│  │  Amazon SNS (Notifications)              │                  │
│  │                                          │                  │
│  │  Topics:                                 │                  │
│  │  - upload-completed                      │                  │
│  │  - upload-failed                         │                  │
│  │  - processing-completed                  │                  │
│  │                                          │                  │
│  │  Subscribers:                            │                  │
│  │  - WebSocket API (real-time to clients)  │                  │
│  │  - Mobile push notifications (FCM)       │                  │
│  │  - Email (digest notifications)          │                  │
│  └──────────────────────────────────────────┘                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    OBSERVABILITY LAYER                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────┐                  │
│  │  Amazon CloudWatch                       │                  │
│  │                                          │                  │
│  │  - Logs: Structured JSON logging         │                  │
│  │  - Metrics: Custom + AWS service metrics │                  │
│  │  - Dashboards: Real-time monitoring      │                  │
│  │  - Alarms: Critical + warning alerts     │                  │
│  │                                          │                  │
│  │  Key Metrics:                            │                  │
│  │  - Upload success rate                   │                  │
│  │  - API latency (P50, P95, P99)           │                  │
│  │  - Lambda execution time                 │                  │
│  │  - SQS queue depth                       │                  │
│  │  - ECS CPU/Memory utilization            │                  │
│  └──────────────────────────────────────────┘                  │
│                                                                 │
│  ┌──────────────────────────────────────────┐                  │
│  │  AWS X-Ray (Distributed Tracing)         │                  │
│  │                                          │                  │
│  │  - End-to-end request tracking           │                  │
│  │  - Service map visualization             │                  │
│  │  - Latency bottleneck identification     │                  │
│  │  - Error root cause analysis             │                  │
│  └──────────────────────────────────────────┘                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                      SECURITY LAYER                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  VPC (Virtual Private Cloud):                                  │
│  - Public subnets: ALB, NAT Gateway                            │
│  - Private subnets: ECS, RDS, Lambda                           │
│  - Multi-AZ deployment (us-east-1a, us-east-1b)                │
│                                                                 │
│  Security Groups:                                              │
│  - ALB: Inbound 443 from CloudFront only                       │
│  - ECS: Inbound from ALB only                                  │
│  - RDS: Inbound from ECS only (port 5432)                      │
│  - Lambda: Outbound to S3, SQS, RDS                            │
│                                                                 │
│  IAM Roles & Policies:                                         │
│  - ECS Task Role: S3 read/write, SQS publish, SNS publish      │
│  - Lambda Execution Role: S3 read, RDS update, SNS publish     │
│  - Least privilege principle enforced                          │
│                                                                 │
│  Encryption:                                                   │
│  - At rest: KMS for S3, RDS, EBS                               │
│  - In transit: TLS 1.3 everywhere                              │
│  - Secrets: AWS Secrets Manager (DB credentials, API keys)     │
│                                                                 │
│  Multi-tenant Isolation:                                       │
│  - Row-Level Security (RLS) in PostgreSQL                      │
│  - S3 bucket policies per tenant prefix                        │
│  - Presigned URL validation includes tenant_id                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Component Details

### 1. Ingress Layer

**Purpose**: Handle incoming traffic, provide CDN, security, and load balancing.

- **Route 53**: DNS with health checks and latency-based routing
- **CloudFront**:
  - LATAM edge locations (Mexico City, São Paulo, Buenos Aires)
  - Caches static assets and upload URLs
  - Reduces latency for field workers
- **WAF**:
  - Rate limiting (1,000 req/min per IP)
  - SQL injection / XSS protection
  - Geographic restrictions if needed
- **ALB**:
  - Path-based routing (/api/uploads/* → upload logic)
  - Health checks (HTTP 200 on /health endpoint)
  - SSL/TLS termination

### 2. Application Layer

**Purpose**: Core API logic, business rules, authentication.

- **ECS Fargate**:
  - Serverless containers (no EC2 management)
  - Auto-scaling: Target CPU 70%, scale 2-10 tasks
  - Each task: 2 vCPU, 4 GB RAM
  - Rolling updates (blue/green deployments)

- **NestJS Monolithic Modular API**:
  - TypeScript for type safety
  - Modular architecture (easy to extract modules later)
  - Upload Module is isolated but part of same deployment
  - Middleware: Multi-tenant isolation, authentication, logging

### 3. Data Layer

**Purpose**: Persistent storage for relational data and caching.

- **RDS Aurora PostgreSQL**:
  - Multi-AZ for high availability
  - Primary instance: All writes
  - Read replica: Reports, analytics (offload reads)
  - Automated backups (7-day retention)
  - Row-Level Security for multi-tenancy

- **ElastiCache Redis**:
  - Upload status (cache for 30 seconds to reduce DB load from polling)
  - Session storage (JWT refresh tokens)
  - Rate limiting counters
  - API response caching (project lists, user profiles)

### 4. Storage & Processing Layer

**Purpose**: Handle file uploads and asynchronous processing.

**Key Design Decision**: Direct client-to-S3 upload using presigned URLs
- API doesn't proxy files → eliminates bottleneck
- S3 handles all upload traffic → infinite scalability
- API only generates URLs → lightweight operation (<10ms)

**Processing Pipeline**:
1. S3 triggers event on object creation
2. EventBridge routes to SQS queue
3. Lambda functions poll queue and process files
4. Results stored in DB, notifications sent via SNS

**Why Lambda for processing?**
- Spiky workload (2,000 uploads in 1 hour, then 50/hour)
- Auto-scales from 0 to 100 concurrent executions
- Pay only for execution time (cost-effective)
- No server management

### 5. Observability Layer

**Purpose**: Monitor system health, debug issues, track performance.

- **CloudWatch**:
  - Centralized logging (all services)
  - Custom metrics (upload_success_rate, upload_latency_p95)
  - Dashboards for ops team
  - Alarms → PagerDuty/Slack

- **X-Ray**:
  - Trace requests across services (API → S3 → Lambda → DB)
  - Identify slow operations
  - Visualize service dependencies

### 6. Security Layer

**Purpose**: Defense in depth, multi-tenant isolation, compliance.

- **Network security**: VPC, security groups, NACLs
- **Application security**: JWT authentication, RBAC (3 user roles)
- **Data security**: Encryption at rest (KMS), in transit (TLS 1.3)
- **Multi-tenant isolation**: RLS prevents data leakage between constructoras

---

## LATAM-Specific Optimizations

### 1. S3 Transfer Acceleration
- Uses CloudFront edge locations for faster uploads
- Client uploads to nearest edge → AWS private network → S3
- **50% faster** from Mexico/LATAM construction sites

### 2. CloudFront LATAM Edge Locations
- Caches presigned URLs (valid for 15 minutes)
- Reduces API latency from 200ms → 50ms for field workers
- Serves static assets (thumbnails, PDFs) from edge

### 3. Multipart Upload Strategy
- Files >5MB split into 5MB chunks
- Each chunk uploaded independently
- **Resumable**: If network fails, only re-upload failed chunks
- Critical for unreliable construction site internet

### 4. Retry Logic Tuned for Poor Networks
- Exponential backoff: 1s, 2s, 4s, 8s, 16s (max 30s)
- 5 retry attempts before failing
- Lambda processing retries: 3 attempts + DLQ

---

## Scalability Characteristics

| Component | Current Capacity | Scaling Strategy | Maximum Capacity |
|-----------|-----------------|------------------|------------------|
| **ECS API** | 2 tasks | Auto-scale on CPU/queue depth | 10 tasks (5x) |
| **RDS** | db.r6g.large | Vertical scaling + read replicas | db.r6g.4xlarge (16x vCPU) |
| **Lambda** | 10 concurrent | Auto-scale | 1,000 concurrent (AWS limit) |
| **S3** | Unlimited | N/A | Unlimited |
| **Redis** | cache.t3.medium | Vertical scaling | cache.r6g.xlarge (8x) |

**Projected capacity**:
- **Current architecture**: 5,000 uploads/day
- **Proposed architecture**: 50,000 uploads/day (10x improvement)
- **At 3x growth (450 clients)**: Handles 15,000 uploads/day comfortably

---

## Cost Estimate

**Monthly costs (USD)**:

| Service | Cost | Notes |
|---------|------|-------|
| ECS Fargate (3 tasks avg) | $200 | 2 vCPU, 4GB RAM per task |
| RDS Aurora (r6g.large Multi-AZ) | $350 | Includes 500 GB storage |
| ElastiCache Redis | $80 | cache.t3.medium |
| S3 Standard + Transfer Accel | $250 | 150 GB new/month, 500GB total |
| Lambda | $100 | 5K uploads × 30 days × 4 functions |
| ALB | $20 | Fixed cost |
| CloudFront | $100 | LATAM traffic |
| Data Transfer | $80 | Outbound to internet |
| CloudWatch + X-Ray | $70 | Logs, metrics, traces |
| **TOTAL** | **~$1,400/month** | |

**Per-client cost**: $1,400 / 150 = **$9.33/client/month**

**At 3x growth (450 clients, 15K uploads/day)**: ~$3,000/month = **$6.67/client/month** (economies of scale)

---

**Next section**: Detailed upload flow (sequence diagram and step-by-step explanation)
