# Presentation Outline: 30-Minute Walkthrough

This document provides a structured outline for the 30-minute technical walkthrough, followed by Q&A.

---

## Presentation Structure

**Total Time**: 30 minutes
- Presentation: 25 minutes
- Buffer/Transition: 5 minutes (for questions during presentation)
- Q&A: 15-20 minutes (separate session)

---

## Slides Outline

### Slide 1: Title Slide (30 seconds)

```
┌────────────────────────────────────────────┐
│                                            │
│   Backend Architecture Proposal            │
│   BuildPeer Scale-Up                       │
│                                            │
│   Designing a Scalable Upload System       │
│                                            │
│   Presented by: [Your Name]                │
│   Date: [Presentation Date]                │
│                                            │
└────────────────────────────────────────────┘
```

**What to say:**
> "Good morning/afternoon. Today I'll present a proposal to re-architect BuildPeer's backend to handle rapid growth, specifically addressing the upload scalability challenge. We'll cover the architecture, key decisions, and migration approach."

---

### Slide 2: BuildPeer Context (2 minutes)

```
┌────────────────────────────────────────────┐
│  BuildPeer: #1 Proptech in LATAM          │
│                                            │
│  Current State:                            │
│  • 150 construction company clients        │
│  • 10,000+ users (3 roles)                 │
│  • 5,000 uploads/day                       │
│  • Growing 3x in next 12 months            │
│                                            │
│  Pain Points:                              │
│  ❌ Single EC2 monolith (fragile)          │
│  ❌ Slow uploads from field sites          │
│  ❌ No visibility into upload status       │
│  ❌ Deployment downtime                    │
│  ❌ Cannot scale to 3x growth              │
│                                            │
└────────────────────────────────────────────┘
```

**What to say:**
> "BuildPeer is the leading construction management platform in Latin America with 150 clients and 10,000 users. Currently running on a single EC2 instance with a monolithic Express app. The main pain point: handling 5,000 file uploads per day with spikes of 1,000-2,000 when projects kickoff. This is projected to triple in the next year, and the current infrastructure cannot handle it."

**Key points:**
- Emphasize LATAM context (unreliable networks at construction sites)
- Highlight growth trajectory (3x in 12 months)
- Focus on upload scalability as primary challenge

---

### Slide 3: Proposed Architecture Overview (3 minutes)

```
┌────────────────────────────────────────────┐
│      High-Level Architecture               │
│                                            │
│  [Show Diagram 1: System Architecture]    │
│                                            │
│  Key Components:                           │
│  • CloudFront → ALB → ECS (NestJS)         │
│  • S3 with Transfer Acceleration           │
│  • EventBridge → SQS → Lambda              │
│  • RDS Aurora + Redis Cache                │
│  • CloudWatch + X-Ray                      │
│                                            │
└────────────────────────────────────────────┘
```

**What to say:**
> "The proposed architecture is event-driven with three key innovations:
>
> 1. Clients upload directly to S3 using presigned URLs—bypassing the API entirely for file transfer
> 2. Event-driven processing pipeline with Lambda for thumbnails, virus scanning, metadata extraction
> 3. Multi-AZ deployment with auto-scaling for high availability
>
> The API is containerized on ECS Fargate, backed by RDS Aurora PostgreSQL and Redis for caching."

**Key points:**
- Walk through the diagram top to bottom
- Highlight the "direct to S3" upload path
- Mention LATAM optimizations (S3 Transfer Acceleration, CloudFront edge locations)

---

### Slide 4: Upload Flow - The Core Solution (8 minutes)

```
┌────────────────────────────────────────────┐
│       Upload Workflow Deep Dive            │
│                                            │
│  [Show Diagram 2: Upload Sequence]         │
│                                            │
│  Steps:                                    │
│  1. Client requests upload → API generates │
│     presigned URL (10ms)                   │
│  2. Client uploads directly to S3          │
│     (bypasses API)                         │
│  3. S3 event → EventBridge → SQS           │
│  4. Lambda processes asynchronously        │
│  5. Status updated → client notified       │
│                                            │
│  Result: 10x throughput improvement        │
│                                            │
└────────────────────────────────────────────┘
```

**What to say:**
> "Let me walk through the upload flow step by step—this is the core of the solution.
>
> **Step 1**: A field worker opens the mobile app and selects 50 photos to upload. The app sends a request to the API: 'I want to upload these files.'
>
> **Step 2**: The API doesn't say 'send me the files.' Instead, it generates presigned URLs from S3—think of these as temporary, secure upload links—and returns them to the client. This operation takes about 10 milliseconds.
>
> **Step 3**: Now here's the key innovation: the client uploads directly to S3 using those URLs. The API is completely bypassed for file transfer. This means the API never sees the 150 megabytes of file data—only the lightweight metadata.
>
> **Step 4**: When the file lands in S3, S3 emits an event. EventBridge routes this event to an SQS queue, which triggers a Lambda function.
>
> **Step 5**: The Lambda function downloads the file, generates thumbnails, runs a virus scan, extracts metadata like GPS coordinates from EXIF data, and uploads the processed versions back to S3.
>
> **Step 6**: Lambda updates the database status from 'UPLOADED' to 'COMPLETED' and publishes a notification via SNS.
>
> **Step 7**: The client receives a real-time notification via WebSocket, or polls the status endpoint every 5 seconds.
>
> **The beauty of this approach**: The API handles 5,000 URL generations per day—a trivial load—while S3 handles 5,000 file uploads per day, which it's designed to do at massive scale. We've eliminated the bottleneck."

**Key points:**
- Emphasize "API doesn't touch file data"
- Explain presigned URLs (security aspect)
- Mention multipart upload for large files (resumable for poor networks)
- Show metrics: 10ms URL generation vs 30 seconds file upload (clearly not the API's problem)

---

### Slide 5: Architectural Decisions & Trade-offs (5 minutes)

```
┌────────────────────────────────────────────┐
│       Key Decisions & Justifications       │
│                                            │
│  Decision              | Justification     │
│  ─────────────────────|──────────────────  │
│  Presigned URLs       | API not bottleneck │
│  vs Proxy Upload      | Infinite scale     │
│  ✅ Chosen: Presigned | ❌ Client complexity│
│                                            │
│  Lambda vs ECS        | Spiky workload     │
│  Workers              | Pay-per-use        │
│  ✅ Chosen: Lambda    | ❌ Cold starts     │
│                                            │
│  Monolith vs          | Team size (7)      │
│  Microservices        | Velocity critical  │
│  ✅ Chosen: Modular   | ❌ Coupled scaling │
│  Monolith             |                    │
│                                            │
│  Shared DB vs         | Cost: $350 vs      │
│  DB per Tenant        | $22,500/month      │
│  ✅ Chosen: Shared+RLS| ❌ Data leak risk  │
│                                            │
└────────────────────────────────────────────┘
```

**What to say:**
> "Every architectural decision involves trade-offs. Let me highlight four critical ones:
>
> **Decision 1: Presigned URLs vs Proxy Upload**
> - We chose presigned URLs because the API would be the bottleneck if it proxied files.
> - Trade-off: Client implementation is more complex (multipart upload logic).
> - But we control the client, so we implement this once, and gain infinite scalability.
>
> **Decision 2: Lambda vs ECS Workers for Processing**
> - We chose Lambda because uploads are spiky: 2,000 in one hour, then 200 over the next 10 hours.
> - ECS workers would need to be provisioned for peak and sit idle 90% of the time.
> - Lambda scales from 0 to 100 automatically and we pay only for execution time.
> - Trade-off: Cold starts add 1-2 seconds of latency on first invocation.
> - Mitigated with reserved concurrency (keeps 20 functions warm).
>
> **Decision 3: Monolith vs Microservices**
> - This was a critical decision. We chose a modular monolith, not microservices.
> - Why? Team size: 7 engineers, 5 of whom are juniors. Microservices require 2-3 engineers per service for ownership and on-call. We'd be spread too thin.
> - Operational complexity: With microservices, we'd need service discovery, inter-service communication, distributed tracing across 5+ services.
> - Velocity matters: Startup needs to iterate fast. Monolith = single deploy, simple debugging.
> - Trade-off: We can't scale upload processing independently from the API. But Lambda already handles processing independently, so this is mitigated.
> - Future-proofing: The upload module is already isolated with clear interfaces. If we need to extract it to a microservice later, it's a 2-week effort, not a 6-month rewrite.
>
> **Decision 4: Shared Database with Row-Level Security**
> - We chose one database for all 150 tenants, with row-level security policies.
> - Why? Cost: $350/month for one RDS instance vs $22,500/month for 150 separate databases.
> - Security: Application-level checks (every query includes WHERE tenant_id = X) plus database-level enforcement (RLS policies prevent cross-tenant access even if SQL injection occurs).
> - Trade-off: Single point of failure for all clients. Mitigated with Multi-AZ (automatic failover in 60 seconds) and automated backups.
>
> The pattern here is pragmatism: optimize for the current scale (150 clients, 7 engineers), not for Google-scale. The architecture can evolve as we grow."

**Key points:**
- Be honest about trade-offs (shows senior-level thinking)
- Explain the "why" behind each decision (context matters)
- Show you understand the team's constraints (velocity, budget, expertise)

---

### Slide 6: Scalability & Resilience (3 minutes)

```
┌────────────────────────────────────────────┐
│      How This Scales (3x Growth)           │
│                                            │
│  Component      | Current | 3x  | Strategy │
│  ───────────────|─────────|──────|───────── │
│  ECS Tasks      | 2       | 4-6  | Auto-scale│
│  RDS Instance   | r6g.L   | r6g.XL| Vertical │
│  Lambda Concur. | 20      | 60   | Auto     │
│  S3 Throughput  | ∞       | ∞    | N/A      │
│                                            │
│  Resilience Features:                      │
│  • Multi-AZ (ECS, RDS, Redis)              │
│  • Auto-healing (ECS health checks)        │
│  • Retry logic (SQS + DLQ)                 │
│  • Circuit breakers (prevent cascades)     │
│                                            │
│  Observability:                            │
│  • CloudWatch metrics + alarms             │
│  • X-Ray distributed tracing               │
│  • Per-tenant dashboards                   │
│                                            │
└────────────────────────────────────────────┘
```

**What to say:**
> "Let's talk about how this handles growth. At 3x scale—450 clients, 15,000 uploads per day—here's what happens:
>
> - **ECS auto-scales** from 2 to 6 tasks based on CPU and queue depth.
> - **RDS vertically scales** from r6g.large to r6g.xlarge (double the capacity), plus we add a read replica for reports.
> - **Lambda auto-scales** from 20 to 60 concurrent executions automatically.
> - **S3 has infinite throughput**—no bottleneck there.
>
> For resilience, we have:
> - Multi-AZ deployment: If one availability zone fails, traffic automatically routes to the other.
> - Auto-healing: ECS health checks detect failed tasks and launch replacements automatically.
> - SQS retry logic: If Lambda fails to process a file, SQS retries 3 times before sending to a Dead Letter Queue for manual investigation.
> - Circuit breakers: If S3 API is down, we fail fast with a user-friendly error instead of cascading failures.
>
> For observability:
> - CloudWatch tracks upload success rate, API latency P95, Lambda errors—all with alarms.
> - X-Ray provides distributed tracing: we can see exactly where latency is occurring in the request flow.
> - Per-tenant dashboards let us quickly diagnose 'Why are uploads slow for Client X?'
>
> This architecture handles 10x growth without major code changes—just scaling up resources."

**Key points:**
- Show specific numbers (builds confidence)
- Mention auto-scaling (not manual intervention)
- Emphasize observability (shows production-readiness)

---

### Slide 7: Migration Strategy (3 minutes)

```
┌────────────────────────────────────────────┐
│      Incremental Migration Approach        │
│          (Strangler Fig Pattern)           │
│                                            │
│  [Show Diagram 3: Migration Phases]        │
│                                            │
│  Phase 1 (Week 2): Upload Module           │
│  ✅ Extract upload logic to ECS            │
│  ✅ Route /api/uploads/* to new service    │
│  ✅ Legacy handles rest                    │
│  ✅ Shared database (no migration)         │
│  ✅ Rollback = ALB rule change (30 sec)    │
│                                            │
│  Value: 10x upload throughput              │
│                                            │
│  Phase 2 (Week 3-4): Containerize Legacy   │
│  Phase 3 (Week 5-12): Modularize Rest      │
│                                            │
└────────────────────────────────────────────┘
```

**What to say:**
> "I want to emphasize that this is not a big bang rewrite. We use the Strangler Fig Pattern for incremental migration.
>
> **Phase 1 - Week 2**: We extract just the upload module to a new ECS service. The ALB routes `/api/uploads/*` to the new service, and everything else goes to the legacy EC2 monolith. Critically, both share the same database—no data migration needed.
>
> The rollback plan is simple: change the ALB routing rule. Takes 30 seconds. Zero data loss risk because the database is shared.
>
> We deploy with a canary: 10% of traffic goes to the new service first. If metrics look good (upload success rate >98%, latency <200ms), we gradually increase: 10% → 25% → 50% → 100%.
>
> **Value delivered in Week 2**: Users immediately see 10x faster uploads and higher success rates. That's 2 weeks to value, not 6 months.
>
> **Phase 2 - Weeks 3-4**: We containerize the legacy monolith (same code, just in Docker on ECS). This enables auto-scaling and zero-downtime deployments.
>
> **Phase 3 - Weeks 5-12**: We refactor the remaining modules (Projects, Users, RFIs) into clean NestJS modules. This is internal code cleanup—no user impact.
>
> Total timeline: 12 weeks to fully modernized architecture, but the critical problem—upload scalability—is solved in Week 2."

**Key points:**
- Emphasize low risk (shared database, easy rollback)
- Highlight fast time-to-value (Week 2)
- Show you understand operational constraints (150 active clients, zero downtime tolerance)

---

### Slide 8: Success Metrics (1 minute)

```
┌────────────────────────────────────────────┐
│           Success Metrics                  │
│                                            │
│  Metric                | Target | Current │
│  ──────────────────────|────────|─────────│
│  Upload Success Rate   | >98%   | ~85%    │
│  P95 Upload Latency    | <30s   | ~60s    │
│  API P99 Latency       | <500ms | ~1.2s   │
│  Deployment Downtime   | 0 min  | 30 min  │
│  Auto-Scaling          | Yes    | No      │
│                                            │
│  Business Impact:                          │
│  • Support 3x growth (450 clients)         │
│  • Reduce field worker frustration         │
│  • Enable enterprise deals (5K+ users)     │
│  • Cost: $1,400/month (~$9/client)         │
│                                            │
└────────────────────────────────────────────┘
```

**What to say:**
> "How do we measure success? Five key metrics:
>
> 1. Upload success rate improves from 85% to 98%+
> 2. P95 upload latency drops from 60 seconds to under 30 seconds
> 3. API latency improves from 1.2 seconds to under 500ms
> 4. Deployments go from 30 minutes of downtime to zero downtime
> 5. Auto-scaling enabled—no more manual intervention
>
> Business impact: We can support 3x growth, reduce field worker frustration (critical for retention), enable enterprise deals with 5,000+ users per client, and all at a cost of about $9 per client per month."

---

### Slide 9: Summary (1 minute)

```
┌────────────────────────────────────────────┐
│               Summary                      │
│                                            │
│  Problem:                                  │
│  Upload scalability bottleneck            │
│                                            │
│  Solution:                                 │
│  Event-driven architecture with:           │
│  • Presigned URLs (direct S3 upload)       │
│  • Lambda processing (async, scalable)     │
│  • Multi-AZ, auto-scaling                  │
│                                            │
│  Key Decisions:                            │
│  • Pragmatic (monolith, not microservices) │
│  • Incremental migration (low risk)        │
│  • LATAM optimizations (Transfer Accel)    │
│                                            │
│  Value:                                    │
│  • 10x upload throughput                   │
│  • Week 2 delivery (fast ROI)              │
│  • Supports 10x growth                     │
│                                            │
└────────────────────────────────────────────┘
```

**What to say:**
> "To summarize:
>
> The problem is upload scalability—the current EC2 monolith cannot handle 5,000 uploads per day with spikes, and growth is projected at 3x.
>
> The solution is an event-driven architecture where clients upload directly to S3 using presigned URLs, bypassing the API. Lambda functions process files asynchronously. Everything is Multi-AZ with auto-scaling.
>
> Key decisions reflect pragmatism: we chose a modular monolith over microservices because of team size, incremental migration over big bang because of risk, and LATAM-specific optimizations like S3 Transfer Acceleration.
>
> The value: 10x upload throughput, delivered in Week 2, with an architecture that supports 10x growth without major changes.
>
> I'm happy to take questions."

---

## Q&A Preparation (15-20 minutes)

### Expected Questions & Answers

#### Q1: "Why not use Lambda for the API as well?"

**Answer:**
> "Great question. I considered that. Lambda is perfect for the processing pipeline because it's event-driven and spiky. But the API has different characteristics:
>
> 1. The API needs persistent WebSocket connections for real-time notifications. Lambda doesn't support that well.
> 2. The API has fairly steady traffic—not spiky—so ECS's fixed cost model is actually cheaper than Lambda's pay-per-request for this workload.
> 3. Cold starts: Lambda can have 1-2 second cold starts. For API requests, users expect sub-200ms latency.
>
> So we use the right tool for each job: ECS for the API, Lambda for processing."

---

#### Q2: "How do you handle files larger than 100 MB?"

**Answer:**
> "BuildPeer's current max file size is 100 MB, which Lambda can handle within the 15-minute timeout. For the multipart upload, we split files into 5MB chunks, so even a 100MB file is 20 parts, each uploaded in parallel.
>
> If BuildPeer needs to support larger files in the future—say, 500MB—we have two options:
> 1. Use Lambda with longer processing time (15 min is usually enough)
> 2. Use ECS workers for files above a certain threshold (e.g., 200MB+)
>
> But for the current use case, Lambda is sufficient."

---

#### Q3: "What if a user uploads a malicious file?"

**Answer:**
> "Security is built into multiple layers:
>
> 1. **Client-side validation**: We check file type and size before generating presigned URLs.
> 2. **Presigned URL constraints**: The presigned URL includes Content-Type validation—S3 rejects uploads that don't match.
> 3. **Virus scanning Lambda**: Every file is scanned with ClamAV before being marked as 'COMPLETED'. If a virus is detected, the file is quarantined (tagged in S3) and the upload record is marked as 'FAILED' with an error message.
> 4. **Access control**: Presigned URLs expire in 15 minutes and include the tenant_id, so users cannot upload files to another tenant's project.
>
> Additionally, BuildPeer's users are authenticated constructoras, inversionistas, and empleados—not the general public—so the risk is lower than a consumer app."

---

#### Q4: "How do you ensure data isolation between tenants?"

**Answer:**
> "Multi-tenancy is critical for BuildPeer. We use a defense-in-depth approach:
>
> **Application Layer:**
> - Every request includes an X-Tenant-ID header (extracted from JWT).
> - Middleware validates that the user belongs to that tenant.
> - Every database query includes WHERE tenant_id = X.
>
> **Database Layer:**
> - PostgreSQL Row-Level Security (RLS) policies enforce tenant isolation.
> - Even if SQL injection bypassed the application checks (unlikely with parameterized queries), RLS would prevent cross-tenant access.
>
> **Storage Layer:**
> - S3 bucket structure: /tenant-A/project-1/uploads/
> - Presigned URLs include the tenant prefix—users cannot generate URLs for other tenants' paths.
> - Bucket policies enforce that IAM roles can only access specific prefixes.
>
> **Audit:**
> - All queries are logged with tenant_id in CloudWatch.
> - We have alarms that trigger if any query returns rows from multiple tenants (which should never happen).
>
> I'd be happy to walk through the RLS policy syntax if you'd like."

---

#### Q5: "What's the estimated monthly cost?"

**Answer:**
> "For the current scale (150 clients, 5,000 uploads/day), approximately $1,400 per month:
>
> - ECS Fargate (3 tasks avg): $200
> - RDS Aurora (r6g.large Multi-AZ): $350
> - ElastiCache Redis: $80
> - S3 + Transfer Acceleration: $250
> - Lambda processing: $100
> - ALB, CloudFront, data transfer: $200
> - Observability (CloudWatch, X-Ray): $70
> - **Total: ~$1,400/month**
>
> That's about $9 per client per month, which is very cost-effective.
>
> At 3x scale (450 clients, 15K uploads/day), costs would increase to approximately $3,000/month—about $6.67 per client. Economies of scale kick in.
>
> Compared to the current EC2 setup at $500/month: yes, it's 3x more expensive, but we get 10x capacity and significantly better reliability. The ROI is clear."

---

#### Q6: "How do you handle the migration without downtime?"

**Answer:**
> "This is critical since BuildPeer has 150 active clients. The strategy is:
>
> **Phase 1:**
> - Deploy the new upload service to ECS alongside the existing EC2 monolith.
> - Both services share the same RDS database—no data migration needed.
> - Configure ALB with path-based routing: /api/uploads/* goes to the new service, everything else to the old service.
> - Start with a canary: 10% of upload traffic to the new service. If metrics are good, increase to 25%, 50%, 75%, 100% over the course of a day.
>
> **Rollback:**
> - If anything goes wrong, we change the ALB rule to route 100% back to the old service. This takes 30 seconds. Zero data loss because the database is shared.
>
> **Phase 2:**
> - Once uploads are stable on the new service, we containerize the legacy monolith and deploy it to ECS. Again, both run in parallel during the transition.
>
> The key is that we never turn off the old system until the new system is proven. This minimizes risk."

---

#### Q7: "Why NestJS instead of sticking with Express?"

**Answer:**
> "The current codebase is Express, but unstructured—it's become difficult to maintain as the team has grown. NestJS provides:
>
> 1. **Opinionated structure**: Modules, controllers, services. This is critical for onboarding junior developers—they have conventions to follow.
> 2. **TypeScript**: Type safety catches bugs at compile time. Refactoring is safer.
> 3. **Built-in features**: Dependency injection (testability), Swagger (auto-generated API docs), validation pipes (class-validator).
> 4. **Familiar foundation**: NestJS is built on Express, so the team's existing knowledge transfers.
>
> The learning curve exists, but it's a one-time investment that pays dividends in maintainability. Plus, I have experience with NestJS, so I can mentor the team effectively."

---

#### Q8: "What happens if EventBridge or SQS fails?"

**Answer:**
> "Great question—resilience is critical.
>
> **If EventBridge fails:**
> - S3 events would queue up and be delivered when EventBridge recovers. S3 event delivery is at-least-once, so no events are lost.
> - Worst case: Processing is delayed, but uploads themselves succeed (files are already in S3).
>
> **If SQS fails:**
> - This is extremely rare (SQS has 99.9% SLA), but if it happens:
> - Lambda cannot poll messages, so processing stops.
> - Uploads still succeed (S3 is independent).
> - When SQS recovers, messages are processed (SQS retains messages for up to 14 days by default).
>
> **Monitoring:**
> - CloudWatch alarms trigger if SQS queue depth exceeds a threshold or if Lambda invocations drop to zero unexpectedly.
> - On-call engineer is paged to investigate.
>
> The key is that uploads don't depend on EventBridge or SQS—those are for processing. Users see their files uploaded immediately; processing happens asynchronously."

---

#### Q9: "How do you handle GDPR / data deletion requests?"

**Answer:**
> "Good question. For GDPR compliance:
>
> 1. **User data deletion:**
>    - When a user is deleted, we cascade delete their uploads from the database (foreign key constraints).
>    - A Lambda function is triggered (via database trigger or API call) that deletes the user's files from S3 (using the s3_key stored in the database).
>    - Soft delete option: We can mark records as deleted but retain them for 30 days (audit trail), then hard delete.
>
> 2. **Tenant data deletion:**
>    - When a constructora client terminates, we delete all their data: database records, S3 files (entire /tenant-X/ prefix), Redis cache.
>    - This is a manual process (requires CTO approval) to prevent accidental deletion.
>
> 3. **Data export:**
>    - We provide an API endpoint: GET /api/tenants/{id}/export
>    - Returns a JSON file with all the tenant's data, plus links to S3 files (presigned URLs valid for 24 hours).
>
> 4. **Audit logging:**
>    - All deletion operations are logged in CloudTrail and application logs."

---

### Difficult/Edge Case Questions

#### Q10: "What if the database becomes the bottleneck?"

**Answer:**
> "If RDS becomes the bottleneck, we have several options in order of complexity:
>
> **Short-term (Week 1):**
> - Add read replicas and route read-heavy queries (reports, dashboards) to them.
> - Increase Redis caching (upload status, project lists).
> - Optimize slow queries (add indexes, use EXPLAIN ANALYZE).
>
> **Medium-term (Month 1):**
> - Vertical scaling: Upgrade from r6g.large to r6g.xlarge or r6g.2xlarge (8x capacity available).
> - Connection pooling: Tune PgBouncer settings to handle more concurrent connections.
>
> **Long-term (Month 3+):**
> - Partition hot tables (uploads table by tenant_id or created_at).
> - Extract upload status to DynamoDB (low-latency key-value lookups, infinite scale).
> - Consider Aurora Serverless v2 (auto-scales read/write capacity).
>
> The good news: At BuildPeer's scale (150 clients, 10K users), RDS can easily handle the load. We'd need to hit 1,000+ clients or 100K users before this becomes a real concern."

---

#### Q11: "How do you handle rate limiting / abuse?"

**Answer:**
> "We implement rate limiting at multiple layers:
>
> **Layer 1: WAF (CloudFront)**
> - Rate limit: 1,000 requests per minute per IP address (blocks DDoS).
>
> **Layer 2: API Gateway (if we add it) or Application-level**
> - Redis-based rate limiting: 100 API requests per minute per user.
> - Upload initiation: 50 uploads per minute per user.
>
> **Layer 3: S3 Presigned URL**
> - Presigned URLs expire in 15 minutes (cannot be reused indefinitely).
> - Each URL is for a specific S3 key (cannot upload to arbitrary paths).
>
> **Layer 4: Business Logic**
> - Upload file size: Max 100 MB per file.
> - Upload batch size: Max 1,000 files per batch.
> - Storage quota: Max 50 GB per tenant (alert when approaching limit).
>
> **Monitoring:**
> - CloudWatch metrics track unusual activity (e.g., single user uploading 10,000 files).
> - Alarms trigger if thresholds are exceeded.
>
> For a B2B SaaS product like BuildPeer, rate limiting is more about preventing accidental abuse (e.g., infinite loop in a script) than malicious attacks."

---

## Presentation Delivery Tips

### Before the Presentation

- [ ] Practice the full walkthrough 2-3 times (time yourself)
- [ ] Prepare diagrams in high resolution (readable on screen sharing)
- [ ] Test screen sharing setup (audio, video, screen)
- [ ] Have the proposal documents open (in case you need to reference details)
- [ ] Review the Q&A answers one more time

### During the Presentation

- **Pacing**: Speak clearly, don't rush (especially during technical explanations)
- **Engagement**: Pause after each major section and ask "Any questions so far?"
- **Visual aids**: Use the diagrams liberally (people remember visuals better than text)
- **Storytelling**: Frame as "Here's the problem → Here's how we solve it → Here's the value"
- **Confidence**: If you don't know an answer, be honest: "That's a great question. I'd need to research X, but my initial thought is Y."

### After the Presentation

- **Q&A**: Take notes during Q&A (shows you're listening)
- **Follow-up**: Offer to provide additional details via email if time runs out
- **Feedback**: Ask "What concerns do you have?" (invites constructive criticism)

---

## Key Talking Points (Memorize These)

1. **"The API never touches file data—only lightweight metadata."** (Core insight)
2. **"10x throughput improvement delivered in Week 2, not Month 6."** (Value proposition)
3. **"Pragmatic architecture: optimized for a team of 7, not a team of 70."** (Context awareness)
4. **"Incremental migration with easy rollback—low risk, high value."** (Risk mitigation)
5. **"Handles 10x growth without major code changes."** (Scalability)

---

**Good luck with your presentation!** 🚀
