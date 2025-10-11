# Architectural Decisions & Trade-offs

This section explains the key architectural decisions made for BuildPeer's backend redesign, including alternatives considered and honest trade-off analysis.

---

## Decision Framework

Every architectural decision was evaluated against BuildPeer's specific context:

- **Team size**: 7 engineers (5 junior, 1 senior, 1 CTO)
- **Growth stage**: Post-seed startup, rapid growth
- **Current pain point**: Upload scalability (5,000/day → 15,000/day projected)
- **Operational constraints**: Small team = prefer managed services
- **Cost consciousness**: Budget matters, optimize for value

**Principle**: **Pragmatism over perfection**. Choose the solution that delivers the best ROI given constraints.

---

## Decision 1: Presigned URLs vs Proxy Upload

### The Problem
Users need to upload large files (up to 100 MB) to S3. How should the upload be handled?

### Options Considered

| Option | How It Works | Pros | Cons |
|--------|--------------|------|------|
| **A: Proxy Upload** | Client → API → S3 (API proxies file data) | Simple client implementation | API bottleneck, bandwidth costs, doesn't scale |
| **B: Presigned URLs** ✅ | Client → API (get URL) → S3 directly | API not a bottleneck, infinite scalability, lower costs | More complex client logic, presigned URL management |

### Decision: **Presigned URLs** ✅

**Rationale:**

**Scalability:**
- Proxy upload: API must handle 5,000 × 5MB avg = **25 GB/day** of file transfer
- At 3x growth: 75 GB/day
- ECS bandwidth costs + CPU overhead + memory usage
- **Bottleneck**: API becomes the limiting factor

- Presigned URLs: API generates URL (~10ms, lightweight operation)
- S3 handles all file transfer (infinitely scalable)
- API can handle 10,000+ URL generations/minute on 2 ECS tasks
- **No bottleneck**: S3 Transfer Acceleration optimized for LATAM

**Cost:**
- Proxy: ECS bandwidth (outbound to internet) = $0.09/GB = $6.75/day = **$200/month** just for bandwidth
- Presigned: S3 PUT requests = $0.005/1000 = **$0.75/month** for 5K uploads/day

**Trade-off accepted:**
- ❌ Client complexity increases (must handle presigned URLs, multipart logic)
- ✅ But we control client (web + mobile apps), so we implement once
- ✅ Massive performance improvement for end users
- ✅ Architecture can scale to 50K+ uploads/day without changes

### Implementation Notes
- Use S3 Transfer Acceleration (routes through CloudFront edge locations)
- Presigned URLs valid for 15 minutes (security)
- Multipart upload for files >5MB (resumable)
- Client SDK abstracts complexity from app developers

---

## Decision 2: Lambda vs ECS Workers for Processing

### The Problem
After files upload to S3, we need to process them (thumbnails, virus scan, metadata extraction). What compute model?

### Options Considered

| Option | How It Works | Pros | Cons |
|--------|--------------|------|------|
| **A: ECS Workers** | Long-running containers poll SQS | Predictable performance, no cold starts | Fixed cost (run 24/7), must manage scaling, higher complexity |
| **B: Lambda** ✅ | Event-triggered serverless functions | Auto-scaling, pay-per-use, zero ops | Cold starts, 15-min timeout, complex for long tasks |

### Decision: **Lambda** ✅

**Rationale:**

**Workload Pattern:**
- Uploads are **spiky**: 2,000 in 1 hour (project kickoff), then 200 over next 10 hours
- Average: 5,000/day = ~200/hour steady-state
- Peak: 2,000/hour = 33/minute

- ECS workers: Must provision for peak capacity
  - Need ~10 workers to handle 2,000/hour
  - Workers idle 90% of time (waste of money)
  - Cost: 10 workers × $30/month × 24/7 = **$300/month** even when idle

- Lambda: Scales from 0 to 100 concurrent executions automatically
  - Pay only for execution time
  - 5,000 uploads/day × 4 Lambda invocations (scan, process, thumbnail, metadata) × 10 sec avg = 55 hours execution time/month
  - Cost: 55 hours × $0.0000166667/GB-sec × 2GB = **$7/month** (compute only)
  - Reserved concurrency prevents throttling during spikes

**Simplicity:**
- Lambda: AWS manages servers, scaling, patching, monitoring
- ECS workers: We manage Dockerfile, task definitions, scaling policies, health checks

**Trade-off accepted:**
- ❌ Cold starts: First invocation after idle = +1-2 seconds latency
- ✅ Mitigated by reserved concurrency (keeps functions warm)
- ❌ 15-minute timeout: Can't process very large files (>500 MB)
- ✅ BuildPeer's max file size is 100 MB (fits easily)

### Implementation Notes
- Use reserved concurrency: 20 for steady-state, burst to 100
- Lambda layers for shared dependencies (ClamAV, Sharp, PDFKit)
- SQS batching: Lambda processes 10 messages per invocation (efficiency)
- DLQ for failed invocations (manual retry)

---

## Decision 3: Monolithic Modular vs Microservices

### The Problem
Should the backend be split into multiple microservices, or remain a modular monolith?

### Options Considered

| Option | How It Works | Pros | Cons |
|--------|--------------|------|------|
| **A: Microservices** | Separate services: Users, Projects, Uploads, RFIs, Reports | Max scalability, independent deploys, tech diversity | Operational complexity, distributed transactions, more costly |
| **B: Modular Monolith** ✅ | Single codebase, modular structure (NestJS modules) | Simple ops, fast iteration, single deploy | Scales as one unit, eventual need to split |

### Decision: **Modular Monolith** (with extraction path) ✅

**Rationale:**

**Team Size:**
- 7 engineers (5 junior)
- Microservices typically require 2-3 engineers per service (ownership, on-call)
- With 5 microservices: Spread too thin, context switching overhead, slows velocity

- Monolith: Entire team works in one codebase
  - Easier onboarding for juniors
  - Faster iteration (no coordinating API changes across services)
  - Single deployment pipeline (simpler CI/CD)

**Operational Complexity:**
- Microservices:
  - Service discovery (e.g., AWS Cloud Map)
  - Inter-service communication (REST or gRPC)
  - Distributed tracing (X-Ray across services)
  - 5 separate deployments, monitoring dashboards, on-call rotations
  - Debugging distributed transactions (upload → project → notification)

- Monolith:
  - One ECS service
  - One CloudWatch log group
  - One deployment pipeline
  - Simple debugging (single stack trace)

**Current Scale:**
- 10,000 users, 150 clients
- **Not Google-scale**: Monolith can easily handle 100,000 users on ECS
- Microservices add complexity without proportional benefit at this scale

**Future-Proofing:**
- Modular structure: Upload Module is already isolated
- Clear interfaces between modules
- When scale demands it (e.g., Uploads = 60% of traffic), extract Upload Module to separate service
- Extraction timeline: 2 weeks (not 6 months rewrite)

**Trade-off accepted:**
- ❌ Cannot scale upload processing independently from API (coupled deployment)
- ✅ But Lambda already handles processing independently, so this is mitigated
- ❌ Single point of failure (if API crashes, everything is down)
- ✅ Mitigated by Multi-AZ ECS, health checks, auto-healing

### Implementation Notes
- NestJS modules as bounded contexts (AuthModule, UsersModule, UploadsModule, etc.)
- Shared services in `shared/` folder (AWS, cache, observability)
- Dependency injection: Modules only depend on interfaces, not implementations
- When extracting a module later: Lift-and-shift to new ECS service (minimal rewrite)

---

## Decision 4: Multi-Tenant Strategy (Shared DB vs DB-Per-Tenant)

### The Problem
BuildPeer has 150 constructora clients. How do we isolate their data?

### Options Considered

| Option | How It Works | Pros | Cons |
|--------|--------------|------|------|
| **A: DB per Tenant** | Each client gets own PostgreSQL database | Maximum isolation, easy to migrate client | Expensive (150 RDS instances), management overhead, connection limits |
| **B: Schema per Tenant** | Shared database, separate schema per tenant | Good isolation, cost-effective | Schema management complexity, hard to scale beyond 1000 tenants |
| **C: Shared DB + Row-Level Security** ✅ | All tenants in one database, `tenant_id` column + RLS | Cost-effective, simple, scales to 10,000+ tenants | Risk of data leakage (must be careful), query complexity |

### Decision: **Shared DB with Row-Level Security** ✅

**Rationale:**

**Cost:**
- DB per tenant: 150 RDS instances × $150/month = **$22,500/month** (untenable for startup)
- Schema per tenant: 1 RDS instance, but connection pooling issues at scale
- Shared DB + RLS: 1 RDS instance = **$350/month**

**Scalability:**
- DB per tenant: Linear cost growth (450 clients = $67,500/month)
- Shared DB + RLS: Same $350 instance can handle 1,000+ tenants

**Operations:**
- DB per tenant: Backups, migrations, monitoring for 150 databases
- Shared DB: One backup, one migration, one monitoring dashboard

**Security:**
- Shared DB concerns: What if SQL injection bypasses RLS?
  - Mitigated by parameterized queries (ORM: TypeORM/Prisma)
  - Application-level checks: Every query includes `WHERE tenant_id = $1`
  - Database-level enforcement: RLS policies prevent cross-tenant access even if SQL injection occurs

**Trade-off accepted:**
- ❌ Single database = single point of failure for all clients
- ✅ Mitigated by Multi-AZ RDS (automatic failover in ~60 seconds)
- ✅ Automated backups (point-in-time recovery)
- ❌ "Noisy neighbor": One tenant's heavy query can impact others
- ✅ Mitigated by connection pooling, read replicas for heavy queries

### Implementation Notes

```sql
-- Row-Level Security policy
ALTER TABLE uploads ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON uploads
  USING (tenant_id = current_setting('app.current_tenant')::uuid);

-- Application sets tenant context per request
SET app.current_tenant = '<tenant-uuid>';
```

**Audit logging:**
- All queries logged with tenant_id
- CloudWatch insights to detect cross-tenant access attempts
- Alarm if any query returns rows from multiple tenants

---

## Decision 5: NestJS vs Other Frameworks

### The Problem
Choosing backend framework for new architecture.

### Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **Express.js** (current) | Simple, lightweight, large ecosystem | No structure, every project diverges, hard to scale team |
| **NestJS** ✅ | Opinionated structure, TypeScript, modular, built-in DI | Steeper learning curve, more boilerplate |
| **FastAPI** (Python) | Fast, modern, great docs | Team has JS/TS expertise, harder to find Python talent in Mexico |

### Decision: **NestJS** ✅

**Rationale:**

**Team Expertise:**
- Current codebase: Express.js (JavaScript)
- Team: Familiar with JavaScript/TypeScript
- NestJS: Built on Express, familiar concepts, easier migration

**Structure for Scale:**
- Express: No enforced structure, every developer implements differently
- NestJS: Opinionated structure (modules, controllers, services)
  - Easier to onboard junior developers (conventions to follow)
  - Codebase stays consistent as team grows

**TypeScript:**
- Type safety catches bugs at compile time
- Better IDE autocomplete (productivity boost)
- Refactoring confidence (rename variable = safe across codebase)

**Built-in Features:**
- Dependency injection (testability)
- Swagger/OpenAPI (auto-generated API docs)
- Validation pipes (class-validator)
- Interceptors (logging, caching, transformation)

**Trade-off accepted:**
- ❌ More boilerplate than Express (decorators, modules)
- ✅ But boilerplate enforces best practices (worth it for team)
- ❌ Slightly slower than raw Express (negligible for BuildPeer's scale)

---

## Decision 6: SQS vs Kinesis vs EventBridge for Event Routing

### The Problem
S3 emits events when files are uploaded. How do we route these events to Lambda?

### Options Considered

| Option | Use Case | Pros | Cons |
|--------|----------|------|------|
| **Direct S3 → Lambda** | Simple trigger | Zero latency, no queue cost | No retry logic, no backpressure, can overwhelm Lambda |
| **S3 → SQS → Lambda** ✅ | Reliable queuing | Retry logic, DLQ, backpressure | Extra hop (~1 sec latency) |
| **S3 → EventBridge → Lambda** | Event routing | Flexible filtering, multiple targets | More complex, higher cost |
| **S3 → Kinesis → Lambda** | Streaming | Real-time processing, replay | Overkill for BuildPeer, expensive |

### Decision: **S3 → EventBridge → SQS → Lambda** ✅

**Rationale:**

**Why EventBridge before SQS?**
- Filtering: EventBridge can filter events (e.g., only process `*.jpg`, ignore thumbnails)
- Multiple targets: Can route to multiple destinations (SQS for Lambda + SNS for notifications)
- Future-proofing: Easy to add new event consumers without changing S3 config

**Why SQS before Lambda?**
- Retry logic: If Lambda fails, SQS retries (up to 3 times)
- DLQ: After 3 failures, message goes to Dead Letter Queue for investigation
- Backpressure: If Lambda hits concurrency limit, messages wait in queue (don't get dropped)
- Batching: Lambda can process 10 messages per invocation (efficiency)

**Trade-off accepted:**
- ❌ Extra latency: S3 → EventBridge → SQS → Lambda = ~2-3 seconds
- ✅ Acceptable: Upload processing doesn't need to be instant
- ❌ Extra cost: SQS requests = $0.0000004/request = **$0.06/month** for 5K uploads
- ✅ Negligible cost for resilience gained

---

## Decision 7: CloudFront vs Direct ALB Access

### The Problem
Should clients connect directly to ALB, or through CloudFront CDN?

### Options Considered

| Option | Latency | Cost | Pros | Cons |
|--------|---------|------|------|------|
| **Direct ALB** | Higher (no edge) | Lower | Simple | Slow for LATAM users, no caching, no DDoS protection |
| **CloudFront → ALB** ✅ | Lower (edge) | Higher | Fast for LATAM, caching, WAF integration | Extra $100/month |

### Decision: **CloudFront** ✅

**Rationale:**

**User Experience:**
- Direct ALB: API request from Monterrey, Mexico → us-east-1 (Virginia) = ~100-150ms latency
- CloudFront: API request → Mexico City edge → us-east-1 = ~50-70ms latency
- **50% latency reduction** for LATAM users (critical for mobile app responsiveness)

**Caching:**
- Static assets (thumbnails, PDFs) cached at edge
- Presigned URLs cached for 5 minutes (reduces API load)

**Security:**
- WAF at CloudFront edge (blocks attacks before hitting ALB)
- DDoS protection (AWS Shield Standard included)

**Cost vs Benefit:**
- CloudFront: $100/month for LATAM traffic
- Value: Better UX = higher user retention = worth the cost

---

## Decision 8: Incremental Migration vs Big Bang Rewrite

### The Problem
Current monolith on EC2 needs to be modernized. How to migrate?

### Options Considered

| Option | Timeline | Risk | Pros | Cons |
|--------|----------|------|------|------|
| **Big Bang Rewrite** | 6+ months | High | Clean slate, perfect architecture | No value until complete, high risk, expensive |
| **Lift-and-Shift** | 1 month | Low | Fast, low risk | Doesn't solve upload problem |
| **Incremental (Strangler Fig)** ✅ | 2 weeks (first value) | Low | Immediate value, validate pattern, rollback-friendly | Hybrid state (temporary complexity) |

### Decision: **Incremental Migration (Strangler Fig Pattern)** ✅

**Rationale:**

**Time to Value:**
- Big Bang: 6 months → users wait 6 months for improvement
- Incremental: 2 weeks → users get 10x upload performance immediately

**Risk:**
- Big Bang: If rewrite fails after 4 months, wasted effort
- Incremental: After 2 weeks, if pattern doesn't work, rollback in 30 seconds (ALB rule change)

**Validation:**
- Big Bang: All decisions validated at once (risky)
- Incremental: Validate Upload Module first, learn, apply to remaining modules

**Team Capacity:**
- Big Bang: All 7 engineers working on rewrite for 6 months (no new features)
- Incremental: 3 engineers on Upload Module (2 weeks), 4 engineers continue feature work

**Trade-off accepted:**
- ❌ Temporary hybrid state (EC2 monolith + ECS upload service)
- ✅ Only for 2-4 weeks, then fully migrated
- ❌ Slightly more complex routing (ALB path-based)
- ✅ But gives us safe rollback capability (critical for 150 active clients)

**Implementation**: See Migration Strategy section (next page)

---

## Summary of Key Trade-offs

| Decision | Chosen Path | Trade-off Accepted | Why It's Worth It |
|----------|-------------|-------------------|-------------------|
| **Upload** | Presigned URLs | Client complexity | API not a bottleneck, scales infinitely |
| **Processing** | Lambda | Cold starts | Pay-per-use, auto-scaling, zero ops |
| **Architecture** | Modular Monolith | Coupled scaling | Team velocity, operational simplicity |
| **Multi-tenancy** | Shared DB + RLS | Data leakage risk | Cost-effective ($350 vs $22,500/month) |
| **Framework** | NestJS | More boilerplate | Structure for team, type safety |
| **Event Routing** | EventBridge + SQS | Extra latency (~2s) | Retry logic, DLQ, resilience |
| **CDN** | CloudFront | $100/month cost | 50% latency reduction for LATAM |
| **Migration** | Incremental | Temporary hybrid | Value in 2 weeks, low risk |

---

**Key Principle**: **Optimize for team velocity and operational simplicity at current scale (150 clients, 7 engineers)**. Architecture can evolve as scale demands it.

---

**Next section**: Scalability, resilience, and observability deep dive
