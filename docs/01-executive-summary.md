# Backend Architecture Proposal: BuildPeer Scale-Up

## Executive Summary

### Context

**BuildPeer** is the leading construction management platform in Latin America, experiencing rapid growth:

- **+150 construction company clients** (constructoras)
- **+10,000+ users** across three roles:
  - Constructoras (project owners/managers)
  - Inversionistas (investors/stakeholders)
  - Empleados de obra (field workers)
- **Geographic presence**: Currently Mexico, expanding to LATAM (Colombia, Brazil, Chile, Peru)
- **Team size**: 7 engineers (5 junior developers, 1 senior full-stack, 1 CTO)

### Current Challenges

The existing backend infrastructure is experiencing critical pain points:

1. **Fragile Infrastructure**
   - Single ExpressJS API deployed on EC2 instance
   - Manual deployments with downtime affecting 150 active clients
   - No auto-scaling capability
   - Difficult to maintain and deploy safely

2. **Upload Scalability Issues**
   - **5,000 file uploads per day** with significant spikes
   - **Spike patterns**: 1,000-2,000 uploads when projects are created
   - Slow upload performance from construction sites (unreliable LATAM networks)
   - No visibility into upload status for users
   - File types: Construction plans (5-50 MB), photos (2-5 MB), reports (1-5 MB)

3. **Growth Constraints**
   - Current architecture cannot support 3x projected growth (450 clients, 15K uploads/day)
   - Zero tolerance for downtime (construction is 24/7 operation)
   - Small team needs maintainable, operationally simple solution

### Proposed Solution

**Event-Driven Architecture** with **NestJS Monolithic Modular Backend** deployed on AWS.

#### Core Architecture Principles

1. **Decouple upload handling from API**
   - Presigned URLs for direct client-to-S3 uploads
   - API generates URLs, doesn't proxy files (eliminates bottleneck)
   - S3 Transfer Acceleration optimized for LATAM geography

2. **Event-driven asynchronous processing**
   - S3 triggers → EventBridge → Lambda functions
   - Thumbnail generation, virus scanning, metadata extraction
   - SQS queuing for resilience and retry logic

3. **Incremental migration approach**
   - Strangler Fig Pattern: extract upload module first
   - Validate pattern with immediate value (2 weeks)
   - Low-risk rollback strategy

4. **Multi-tenant architecture**
   - Row-Level Security (RLS) in PostgreSQL
   - 150 tenants with complete data isolation
   - Scalable to 1,000+ tenants without architectural changes

### Key Benefits

| Benefit | Current State | Proposed State | Impact |
|---------|--------------|----------------|--------|
| **Upload Throughput** | ~500 files/hour | 5,000+ files/hour | **10x improvement** |
| **Deployment** | 30 min downtime | Zero downtime (blue/green) | **99.9% uptime** |
| **Scalability** | Manual vertical scaling | Auto-scaling (2-10 tasks) | **Handles 3x growth** |
| **LATAM Performance** | Slow from field sites | S3 Transfer Acceleration | **50% faster uploads** |
| **Operational Complexity** | Manual management | Managed services (ECS, Lambda) | **Reduced ops burden** |
| **Cost Efficiency** | $500/month (limited capacity) | $1,400/month (3x capacity) | **$9/client/month** |

### Technology Stack

- **Backend Framework**: NestJS 10 (TypeScript, Node.js 20 LTS)
- **API Hosting**: ECS Fargate (auto-scaling containers)
- **Database**: RDS Aurora PostgreSQL (Multi-AZ, read replicas)
- **File Storage**: S3 with Transfer Acceleration + Intelligent-Tiering
- **Processing**: Lambda (serverless, event-driven)
- **Caching**: ElastiCache Redis
- **Networking**: ALB, CloudFront (LATAM edge locations), WAF
- **Observability**: CloudWatch, X-Ray, custom metrics

### Success Metrics

**Performance Targets:**
- Upload success rate: **>98%** (currently ~85%)
- P95 upload latency: **<30 seconds** for 5MB file from construction site
- API P99 latency: **<500ms** for CRUD operations

**Business Targets:**
- Support **3x growth** (450 clients) without code changes
- Zero-downtime deployments
- **6-week timeline** from approval to production

### Implementation Timeline

- **Week 1**: Infrastructure setup (ECS, S3, Lambda, RDS)
- **Week 2**: Upload module extraction and deployment (canary rollout)
- **Week 3-4**: Containerize legacy monolith (EC2 → ECS)
- **Week 5-12**: Incremental modularization (Projects, Users, RFIs, Reports)

**First value delivery: Week 2** (upload performance improvement)

---

### Assumptions

This proposal is based on the following assumptions:

1. **Current State**
   - Non-modular ExpressJS monolith on EC2
   - PostgreSQL + S3 already in use
   - ~500 GB of existing documents
   - Manual deployment process

2. **Growth Projections**
   - 3x clients in 12 months (150 → 450 constructoras)
   - 3x upload volume (5K → 15K uploads/day)
   - Geographic expansion to 5 additional LATAM countries

3. **User Behavior**
   - 60% of uploads from mobile devices (field workers)
   - Unreliable internet at construction sites
   - Predictable spike patterns (project kickoffs)
   - Average file size: 3-5 MB

4. **Team & Operational Constraints**
   - 7 engineers (limited capacity for complex operations)
   - NestJS expertise in team (tech lead/senior dev)
   - Cost-conscious (post-seed startup budget)
   - Zero-downtime requirement (SLA 99.5%+)

5. **Business Requirements**
   - Multi-tenant data isolation (compliance, competitive projects)
   - Real-time upload status visibility
   - Audit trail for all operations (construction is regulated)
   - Data sovereignty considerations for LATAM expansion

---

**Next sections detail the technical architecture, upload workflow, key decisions, and migration strategy.**
