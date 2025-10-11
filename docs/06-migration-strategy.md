# Migration Strategy

This section outlines the incremental migration approach to move from the current EC2 monolith to the proposed architecture with **minimal risk** and **immediate value delivery**.

---

## Migration Philosophy: Strangler Fig Pattern

The **Strangler Fig Pattern** (Martin Fowler) involves gradually replacing parts of a legacy system with new functionality, rather than a "big bang" rewrite.

**Key Principles:**
1. **Incremental**: Migrate one module at a time
2. **Coexistence**: Old and new systems run in parallel during transition
3. **Validation**: Each step is validated before proceeding
4. **Rollback**: Easy rollback if issues arise
5. **Zero Downtime**: Users experience no interruption

**Why This Approach for BuildPeer:**
- 150 active clients cannot tolerate downtime
- Team of 7 needs to deliver value while migrating
- Risk mitigation: Each phase independently validated
- Fast time-to-value: First improvement in Week 2 (not Month 6)

---

## Migration Phases Overview

| Phase | Timeline | Focus | Value Delivered | Risk |
|-------|----------|-------|-----------------|------|
| **Phase 0: Preparation** | Week 1 | Infrastructure setup | Foundation ready | Low |
| **Phase 1: Upload Module** | Week 2 | Extract upload handling | **10x upload throughput** | Low |
| **Phase 2: Containerize Legacy** | Week 3-4 | Migrate EC2 → ECS | Zero-downtime deploys | Low |
| **Phase 3: Modularize Remaining** | Week 5-12 | Extract other modules | Maintainability | Medium |

**Critical Success Factor**: Phase 1 delivers immediate, measurable value to users (faster uploads).

---

## Phase 0: Infrastructure Preparation (Week 1)

### Objective
Set up AWS infrastructure without affecting current production system.

### Tasks

#### Day 1-2: Networking & Security
```bash
# VPC and subnets
- Create VPC with public/private subnets in 2 AZs
- Configure route tables, Internet Gateway, NAT Gateway
- Set up security groups (ALB-SG, ECS-SG, RDS-SG, Redis-SG)
- Configure VPC endpoints for S3, ECR (cost optimization)
```

#### Day 3: Database & Cache
```bash
# RDS Aurora PostgreSQL
- Create Aurora cluster (db.r6g.large Multi-AZ)
- Enable automated backups, encryption
- Create read replica (for future use)
- Import schema from current PostgreSQL

# ElastiCache Redis
- Create cache.t3.medium cluster
- Configure security group (ECS access only)
```

#### Day 4: Compute & Storage
```bash
# ECS Cluster
- Create Fargate cluster (buildpeer-prod)
- Configure CloudWatch log groups
- Set up task execution role (IAM)

# S3 Buckets
- Create buildpeer-uploads bucket
- Enable S3 Transfer Acceleration
- Enable versioning, encryption (KMS)
- Configure lifecycle policy (Intelligent-Tiering)
```

#### Day 5: Event Processing & CDN
```bash
# EventBridge + SQS + Lambda
- Create EventBridge rule (S3 → SQS)
- Create SQS standard queue + DLQ
- Deploy skeleton Lambda functions (image-processor, document-processor)
- Configure reserved concurrency (20)

# CloudFront + ALB
- Create ALB with target groups
- Configure health checks (HTTP /health)
- Create CloudFront distribution (point to ALB)
- Configure WAF rules (rate limiting, geo-blocking if needed)
```

### Validation
- [ ] VPC resources accessible
- [ ] RDS accepting connections (test from local machine via bastion)
- [ ] S3 upload test (presigned URL generation)
- [ ] Lambda invocation test (manual trigger)
- [ ] ALB health check passing (simple Express app deployed)

### Rollback
✅ No rollback needed (production unaffected, running in parallel)

---

## Phase 1: Upload Module Extraction (Week 2)

### Objective
Extract upload handling to new ECS service, route `/api/uploads/*` to it, while legacy monolith handles all other routes.

### Architecture During Phase 1

```
┌─────────────────────────────────────────────────┐
│              Application Load Balancer          │
│               (Path-Based Routing)              │
└──────────┬───────────────────────┬──────────────┘
           │                       │
    /api/uploads/*            /api/* (catch-all)
           │                       │
┌──────────▼─────────────┐   ┌─────▼────────────────┐
│  ECS Fargate (NEW)     │   │  EC2 Monolith (OLD)  │
│  ┌──────────────────┐  │   │                      │
│  │ Upload Module    │  │   │  - Projects          │
│  │ - NestJS API     │  │   │  - Users             │
│  │ - Presigned URLs │  │   │  - RFIs              │
│  │ - Status API     │  │   │  - Reports           │
│  └──────────────────┘  │   │  - (Legacy uploads)  │
└──────────┬──────────────┘   └──────┬───────────────┘
           │                         │
           └──────────┬──────────────┘
                      │
            ┌─────────▼──────────┐
            │  RDS PostgreSQL    │
            │  (SHARED DATABASE) │
            └────────────────────┘
```

**Key Decision**: Share database between old and new systems
- ✅ No data migration needed
- ✅ Atomic transactions possible (if needed)
- ✅ Simple rollback (no data sync issues)

### Tasks

#### Day 1: Code Preparation
```typescript
// Extract upload module from monolith
src/modules/uploads/
  ├── uploads.module.ts
  ├── uploads.controller.ts   // POST /initiate, GET /:id/status, POST /:id/complete
  ├── uploads.service.ts       // Business logic
  ├── presigned-url.service.ts // S3 URL generation
  └── multipart-upload.service.ts

// Shared modules (both systems access same DB)
src/shared/
  ├── database/ (TypeORM connection to RDS)
  ├── aws/ (S3, SQS, SNS clients)
  ├── multi-tenant/ (tenant isolation middleware)
```

#### Day 2: Build & Deploy (Canary)
```bash
# Build Docker image
docker build -t buildpeer-api:upload-v1 .
docker push <ECR>/buildpeer-api:upload-v1

# Deploy to ECS (2 tasks)
aws ecs create-service \
  --cluster buildpeer-prod \
  --service-name upload-service \
  --task-definition buildpeer-upload:1 \
  --desired-count 2 \
  --load-balancer targetGroupArn=<upload-tg-arn>,containerName=api,containerPort=3000

# Configure ALB listener rule
aws elbv2 create-rule \
  --listener-arn <alb-listener-arn> \
  --priority 10 \
  --conditions Field=path-pattern,Values='/api/uploads/*' \
  --actions Type=forward,TargetGroupArn=<upload-tg-arn>

# Route 10% of /api/uploads/* traffic to new service (canary)
aws elbv2 modify-rule \
  --rule-arn <upload-rule-arn> \
  --actions Type=forward,ForwardConfig='{
    "TargetGroups":[
      {"TargetGroupArn":"<upload-tg-arn>","Weight":10},
      {"TargetGroupArn":"<legacy-tg-arn>","Weight":90}
    ]
  }'
```

#### Day 3: Monitoring & Validation (10% Traffic)
```bash
# Monitor metrics
- Upload success rate (new service): Target >98%
- API latency P95: Target <200ms
- Error rate: Target <1%
- Lambda processing: Target <30s end-to-end

# Compare old vs new
Metric             | Old Service | New Service | Improvement
-------------------|-------------|-------------|------------
Success Rate       | 87%         | 99%         | +12%
P95 Latency        | 450ms       | 120ms       | -73%
Max Uploads/min    | 50          | 500+        | 10x+

# User feedback
- Monitor support tickets
- Check app reviews
- Field worker reports
```

#### Day 4: Gradual Rollout
```bash
# If metrics good, increase traffic
10% → 25% → 50% → 75% → 100%

# At each step, wait 2 hours, monitor

# Final state: 100% traffic to new service
aws elbv2 modify-rule \
  --rule-arn <upload-rule-arn> \
  --actions Type=forward,TargetGroupArn=<upload-tg-arn>
```

#### Day 5: Cleanup & Documentation
```bash
# Remove legacy upload code from EC2 monolith
# (Keep for 1 week as backup, then delete)

# Update documentation
- API docs (Swagger)
- Runbook (ops team)
- Architecture diagrams

# Deploy Lambda functions (processing pipeline)
- image-processor
- document-processor
- virus-scanner
- metadata-extractor

# Connect S3 → EventBridge → SQS → Lambda
```

### Success Metrics (Must Pass Before Proceeding)

| Metric | Target | Actual (Expected) | Status |
|--------|--------|-------------------|--------|
| Upload success rate | >98% | 99.2% | ✅ |
| P95 upload latency | <30s | 18s | ✅ |
| API error rate | <1% | 0.3% | ✅ |
| Zero critical bugs | 0 | 0 | ✅ |
| User complaints | <5 | 2 | ✅ |

### Rollback Plan

**If metrics fail:**
```bash
# Rollback: Change ALB rule (30 seconds)
aws elbv2 modify-rule \
  --rule-arn <upload-rule-arn> \
  --actions Type=forward,TargetGroupArn=<legacy-tg-arn>

# All traffic back to EC2 monolith
# Investigate issue
# Fix and redeploy
```

**Rollback triggers:**
- Upload success rate < 95% for 10 minutes
- P95 latency > 1s for 10 minutes
- Error rate > 3% for 5 minutes
- Critical bug reported (data loss, security issue)

**Data safety:**
- Shared database = no data migration = no data loss risk
- Uploads in S3 are preserved (both systems access same bucket)

---

## Phase 2: Containerize Legacy Monolith (Week 3-4)

### Objective
Move EC2 monolith to ECS Fargate without changing code.

**Why this phase?**
- Enables auto-scaling (currently manual vertical scaling on EC2)
- Enables blue/green deployments (zero downtime)
- Consistent platform (everything on ECS)

### Tasks

#### Week 3: Containerization
```dockerfile
# Dockerfile (wrap existing Express app)
FROM node:20-alpine

WORKDIR /app

# Copy existing code (minimal changes)
COPY package*.json ./
RUN npm ci --production

COPY . .

EXPOSE 3000
CMD ["node", "server.js"]
```

```bash
# Build and test locally
docker build -t buildpeer-legacy:v1 .
docker run -p 3000:3000 buildpeer-legacy:v1

# Test all endpoints
npm run test:e2e

# Push to ECR
docker push <ECR>/buildpeer-legacy:v1
```

#### Week 4: Parallel Deployment
```bash
# Deploy legacy monolith to ECS (parallel with EC2)
aws ecs create-service \
  --cluster buildpeer-prod \
  --service-name legacy-service \
  --task-definition buildpeer-legacy:1 \
  --desired-count 2

# Add to ALB target group (weighted routing)
# EC2: 90%, ECS: 10% (canary)

# Gradually shift traffic (same pattern as Phase 1)
10% → 25% → 50% → 75% → 100%

# Final state: 100% traffic to ECS
# EC2 instance can be terminated
```

### Success Metrics

| Metric | Target | Status |
|--------|--------|--------|
| API success rate | >99% | ✅ |
| P95 latency | <500ms | ✅ |
| Zero regressions | 0 broken endpoints | ✅ |
| Auto-scaling working | Tasks scale 2→5 on load | ✅ |

### Rollback
✅ Change ALB rule back to EC2 (keep EC2 running for 1 week as backup)

---

## Phase 3: Modularize Remaining Modules (Week 5-12)

### Objective
Extract remaining modules (Projects, Users, RFIs, Reports) into clean NestJS modules within the monolith.

**Note**: This is internal refactoring, not separate microservices.

### Approach

**Week 5-6: Projects Module**
- Extract project CRUD logic
- Isolate dependencies
- Clear interfaces between modules

**Week 7-8: Users Module**
- Extract user management
- Authentication/authorization
- Role-based access control

**Week 9-10: RFIs & Reports Modules**
- Extract RFI workflow
- Extract reporting logic

**Week 11-12: Consolidation**
- Remove dead code
- Optimize module boundaries
- Documentation and training

### Benefits
- ✅ Maintainable codebase (easier for juniors to understand)
- ✅ Faster iteration (clear module boundaries)
- ✅ Future-ready (easy to extract to microservices if needed)

---

## Risk Mitigation

### Risk: Database Migration Errors
**Mitigation:**
- Use shared database (no migration needed in Phase 1)
- Automated schema migrations (TypeORM migrations)
- Test migrations on staging environment first

### Risk: User Experience Disruption
**Mitigation:**
- Gradual rollout (10% → 100%)
- Real-time monitoring
- Fast rollback (<30 seconds via ALB rule)
- Feature flags (can disable new upload flow per tenant)

### Risk: Team Capacity
**Mitigation:**
- Senior developer leads Phase 1 (critical)
- Junior developers support (testing, documentation)
- Phase 2-3 can proceed at slower pace (not urgent)

### Risk: Bugs in New System
**Mitigation:**
- Extensive testing (unit, integration, E2E)
- Canary deployment (catch bugs with 10% traffic)
- CloudWatch alarms (auto-detect issues)
- Rollback plan tested in advance

---

## Timeline & Milestones

```
Week 1: ████████████████████ Infrastructure Setup
        - VPC, RDS, ECS, S3, Lambda
        Milestone: Infrastructure ready ✅

Week 2: ████████████████████ Upload Module Migration
        - Extract, deploy, validate
        Milestone: 10x upload throughput ✅

Week 3: ████████████████████ Containerize Legacy (Part 1)
        - Dockerfile, local testing

Week 4: ████████████████████ Containerize Legacy (Part 2)
        - Deploy to ECS, traffic shift
        Milestone: Zero-downtime deploys ✅

Week 5-12: ██████████████████ Modularization
        - Projects, Users, RFIs, Reports
        Milestone: Maintainable codebase ✅

TOTAL: 12 weeks to fully modernized architecture
FIRST VALUE: Week 2 (Upload performance)
```

---

## Communication Plan

### Week 1 (Preparation)
**Stakeholders**: CTO, Engineering Team
**Message**: "We're setting up infrastructure for new upload system. No impact on users."

### Week 2 (Upload Launch)
**Stakeholders**: All Users, Customer Success
**Message**: "We've launched a new upload system! You should see faster, more reliable uploads, especially from construction sites."

**Metrics to share:**
- "Uploads are now 10x faster"
- "Success rate improved from 87% to 99%"
- "You can now upload 1,000 files at once without issues"

### Week 4 (Full Migration)
**Stakeholders**: CTO, Engineering Team
**Message**: "All services now running on modern infrastructure. Zero-downtime deployments enabled."

### Week 12 (Complete)
**Stakeholders**: Entire Company
**Message**: "Backend modernization complete. Architecture now supports 10x growth without major changes."

---

## Post-Migration

### Decommissioning Legacy
- Week 13: Terminate EC2 instance (after 1 week buffer)
- Week 13: Remove legacy code from repository
- Week 13: Archive old deployment scripts

### Optimization
- Tune ECS auto-scaling thresholds based on real traffic
- Optimize Lambda memory allocations (cost vs performance)
- Fine-tune Redis cache TTLs (hit rate optimization)
- Review RDS query performance (add indexes if needed)

### Future Considerations

**When to extract Upload Module to separate microservice?**

Criteria:
- Upload traffic > 60% of total API traffic
- Need to scale upload processing independently (different scaling profile)
- Team size > 15 engineers (can maintain separate service)

**Extraction effort**: 2 weeks (module already isolated, just move to separate ECS service)

---

## Conclusion

The proposed migration strategy balances **risk** and **value delivery**:

✅ **Low Risk**: Incremental, canary deployments, fast rollback
✅ **Fast Value**: Upload improvements in Week 2 (not Month 6)
✅ **Pragmatic**: Matches team capacity (7 engineers)
✅ **Future-Ready**: Architecture can scale 10x without major changes

**Total timeline**: 12 weeks from start to fully modernized architecture
**First value delivery**: Week 2 (immediate user benefit)

---

**End of Technical Proposal**

For questions or clarifications, please reach out to the engineering team.
