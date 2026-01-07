# AWS Serverless File Processing Pipeline - 2024

## The Challenge

A data analytics company was drowning in file processing costs. They needed to process 100,000 JSON files daily from multiple sources, but their direct S3-to-Lambda approach was burning through $2,500/month with 8-12% failure rates during peak hours.

**The problem:** Every file upload triggered a separate Lambda invocation. Database connection pools were exhausted. Processing lagged 6 hours behind during business hours. Mixed file types (JSON, CSV, images) all triggered processing, wasting money on irrelevant files.

**The goal:** Build a cost-effective, reliable pipeline that processes only relevant files without breaking the bank.

## The Solution

I redesigned the architecture around intelligent filtering and batch processing. Instead of processing files individually, the system filters at the source, buffers events in SQS, and processes 100 files per Lambda invocation.

### Results That Matter
- **Cost reduction:** $2,500 → $236/month (91% savings)
- **Processing speed:** 4.2s → 0.6s per file (86% faster)
- **Reliability:** 8-12% → <0.5% failure rate (95% improvement)
- **Lambda invocations:** 100,000 → 1,000/day (99% reduction)

## Architecture Overview

![Architecture](aws-files.png)

### How It Works
1. **S3 event filtering** → Only JSON files from `/input/` folder trigger events
2. **SQS buffering** → Events accumulate in queue with 4-day retention
3. **Batch processing** → Lambda processes 100 files per invocation
4. **Database efficiency** → Single connection handles 100 inserts
5. **Error handling** → Dead Letter Queue captures failures

### Core Components

**Event Processing**
- S3 event notifications with prefix/suffix filtering
- SQS queue with long polling and DLQ for reliability
- Lambda batch processing (100 messages per invocation)

**Data Storage**
- Amazon Aurora PostgreSQL for structured data
- S3 for raw file storage with lifecycle policies
- AWS Secrets Manager for database credentials

**CI/CD Pipeline**
- GitHub Actions for workflow orchestration
- AWS CodeBuild for VPC-based deployments
- Automated database migrations and dbt transformations

## Key Technical Decisions

### 1. S3 Event Filtering (Critical Cost Saver)
**Problem:** Bucket received 500,000 mixed file types daily, but only 100,000 JSON files needed processing.

**Solution:** S3 event notifications with filtering rules.
```yaml
Filter Rules:
  - Prefix: "input/"     # Only /input/ folder
  - Suffix: ".json"      # Only JSON files
```

**Impact:** 80% reduction in unnecessary events (500K → 100K/day)

### 2. SQS Batch Processing
**Problem:** 100,000 individual Lambda invocations were expensive and overwhelmed database connections.

**Solution:** SQS accumulates events, Lambda processes 100 at once.
- Batch size: 100 messages
- Batching window: 55 seconds
- Single database connection per batch
- 99% reduction in Lambda invocations

### 3. GitHub Actions + CodeBuild CI/CD
**Problem:** Database schema changes and ETL jobs needed automation with VPC access.

**Solution:** GitHub Actions orchestrates, CodeBuild executes in VPC.
- Database migrations run inside VPC with private RDS access
- Daily dbt transformations automated via scheduled workflows
- Infrastructure deployments version-controlled and tested

## Performance & Cost Analysis

**Before (Direct S3 → Lambda):**
```
Lambda Invocations: 100,000/day
Database Connections: 100,000/day (pool exhaustion)
Monthly Cost: $2,500
Failure Rate: 8-12%
Processing Lag: 6 hours
```

**After (Filtered S3 → SQS → Batch Lambda):**
```
Lambda Invocations: 1,000/day
Database Connections: 1,000/day
Monthly Cost: $236
Failure Rate: <0.5%
Processing Lag: <2 minutes
```

### Cost Breakdown
```
Aurora PostgreSQL: $225
Lambda: $0.50
SQS: $0.80
S3 + Secrets Manager: $10

Total: $236/month
Cost per file: $0.08 (100K files/month)
Savings: $2,264/month (91% reduction)
```

## Security & Reliability

**Network Security**
- Aurora in private subnets with security group restrictions
- CodeBuild runs in VPC for secure database access
- No public database endpoints

**Data Protection**
- AWS Secrets Manager for credential management
- Encryption at rest and in transit
- IAM least privilege access

**Error Handling**
- SQS Dead Letter Queue captures persistent failures
- 4-day message retention prevents data loss
- CloudWatch monitoring and alerting

## What I Learned

**Game Changers**
1. **Filtering at the source is non-negotiable** - 80% cost reduction from simple S3 event configuration
2. **Batch processing transforms economics** - At 100K+ events/day, individual processing is financially impossible
3. **SQS as intelligent buffer** - Naturally handles traffic spikes and provides retry mechanisms

**Architecture Insights**
- **Decouple with queues** - SQS absorbed 10K file bursts without breaking downstream systems
- **Automate everything** - GitHub Actions + CodeBuild eliminated manual database deployments
- **Monitor batch sizes** - Started with 10 (too small), settled on 100 (sweet spot for 50KB files)

**Future Improvements**
- Query result caching for repeated transformations
- Reserved Lambda concurrency for guaranteed capacity

## The Impact

This architecture proves that thoughtful design can transform an expensive, unreliable system into a cost-effective solution. The key insight: at high event volumes, individual processing isn't just inefficient—it's economically impossible.

By implementing two critical patterns—source filtering and batch processing—we achieved 91% cost savings while dramatically improving reliability and performance. The solution scales effortlessly to 1M+ files/day by adjusting batch size and concurrency.

Most importantly, it freed the team from constant firefighting around database connection exhaustion and processing delays, allowing them to focus on actual data analytics instead of infrastructure babysitting.

---

**Tech Stack:** Amazon S3, SQS, Lambda, Aurora PostgreSQL, Secrets Manager, CodeBuild, GitHub Actions, dbt
