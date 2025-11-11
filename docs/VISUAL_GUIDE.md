# Visual Architecture Guide - Healthcare AI on AWS

**Quick Reference**: Diagrams, flows, and visual explanations for every component

---

## 1. System Architecture at a Glance

```
┌─────────────┐
│   Users     │
└──────┬──────┘
       │ HTTPS
       ▼
┌─────────────────────────┐
│   API Gateway (REST)    │
│  /summarize, / (health) │
└──────┬──────────────────┘
       │
       ├─────────────────────────┐
       │                         │
       ▼                         ▼
┌────────────────────┐  ┌──────────────────┐
│ Lambda Summarizer  │  │ Lambda Root      │
│ (Report Processing)│  │ (Health Checks)  │
└────────┬───────────┘  └──────────────────┘
         │
         ▼
┌─────────────────────────┐
│ Amazon Bedrock (Titan)  │
│ AI Model Inference      │
└────────┬────────────────┘
         │
         ▼
┌─────────────────────────┐
│ DynamoDB                │
│ PatientReports Table    │
└─────────────────────────┘
```

---

## 2. Request/Response Flow

```
START HERE: User sends medical report
             │
             ▼
    POST /summarize (JSON)
             │
             ├─ Report text
             ├─ Patient ID
             └─ Metadata
             │
             ▼
    API Gateway receives request
             │
             ├─ Validate format
             ├─ Check authorization
             └─ Apply throttling (10K req/sec)
             │
             ▼
    Invoke Lambda Function
             │
             ├─ Parse JSON payload
             ├─ Validate input (not empty, <50KB)
             └─ Prepare Bedrock prompt
             │
             ▼
    Call Amazon Bedrock
             │
             ├─ Send: "Summarize this medical report..."
             ├─ Model: Titan Text Express v1
             └─ Wait: 2-10 seconds (model inference)
             │
             ▼
    Receive AI Summary
             │
             ├─ Parse response
             ├─ Calculate tokens used (for logging)
             └─ Validate output
             │
             ▼
    Write to DynamoDB
             │
             ├─ PatientID: (partition key)
             ├─ ReportID: (sort key)
             ├─ OriginalText: (raw input)
             ├─ Summary: (AI output)
             ├─ ProcessingStatus: SUCCESS
             ├─ CreatedAt: (unix timestamp)
             └─ Metadata: {...}
             │
             ▼
    Log to CloudWatch
             │
             ├─ Execution time
             ├─ Bedrock tokens
             ├─ DynamoDB latency
             └─ Any errors
             │
             ▼
    Return HTTP 200
             │
             ├─ {
             │   "summary": "Patient presents with...",
             │   "status": "SUCCESS",
             │   "timestamp": "2025-11-11T...",
             │   "tokens": {"input": 150, "output": 100}
             │ }
             │
             ▼
    END: Response received by client
```

---

## 3. Architecture Layers

```
┌────────────────────────────────────────────────────┐
│              CLIENT LAYER                          │
│  ┌──────────────────────────────────────────────┐  │
│  │ Web App │ Mobile │ CLI │ 3rd-party systems  │  │
│  └──────────────────────────────────────────────┘  │
└────────────────────┬───────────────────────────────┘
                     │ HTTPS requests
                     ▼
┌────────────────────────────────────────────────────┐
│              API LAYER (Serverless)                │
│  ┌──────────────────────────────────────────────┐  │
│  │ API Gateway: REST endpoints, auth, throttle │  │
│  └──────────────────────────────────────────────┘  │
└────────────────────┬───────────────────────────────┘
                     │ Lambda invocation
                     ▼
┌────────────────────────────────────────────────────┐
│          COMPUTE LAYER (Serverless)                │
│  ┌──────────────────────────────────────────────┐  │
│  │ Lambda: Orchestration, validation, logging  │  │
│  └──────────────────────────────────────────────┘  │
└────────────────────┬───────────────────────────────┘
                     │ Two paths:
          ┌──────────┴──────────┐
          │                     │
          ▼                     ▼
      Call Bedrock        Write to DB
      (AI/ML)             (Storage)
          │                     │
          ▼                     ▼
┌──────────────────┐  ┌──────────────────┐
│ BEDROCK SERVICE  │  │ DYNAMODB TABLE   │
│ • Titan Model    │  │ • PatientReports │
│ • Inference      │  │ • On-demand      │
│ • Per-token cost │  │ • SSE encrypted  │
└──────────────────┘  └──────────────────┘

┌────────────────────────────────────────────────────┐
│          MONITORING LAYER (Observability)          │
│  ┌──────────────────────────────────────────────┐  │
│  │ CloudWatch: Logs, Metrics, Alarms           │  │
│  │ CloudTrail: Audit trail                      │  │
│  │ X-Ray: Distributed tracing (optional)        │  │
│  └──────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────┐
│          SECURITY LAYER (Access Control)           │
│  ┌──────────────────────────────────────────────┐  │
│  │ IAM: Role-based access control               │  │
│  │ TLS/HTTPS: Encryption in transit             │  │
│  │ SSE: Encryption at rest                      │  │
│  │ Secrets Manager: Credential storage          │  │
│  └──────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────┘
```

---

## 4. Deployment Pipeline

```
GITHUB PUSH
    │
    ▼
┌──────────────────────────┐
│ GitHub Actions Workflow  │
│ (Triggered on push)      │
└────────────┬─────────────┘
             │
             ├─ Checkout code
             │
             ├─ Setup Python 3.11
             │
             ├─ Configure AWS credentials
             │
             ├─ Run: sam build
             │    ├─ Install dependencies
             │    ├─ Package Lambda code
             │    └─ Generate CloudFormation template
             │
             ├─ Run: sam deploy
             │    ├─ Upload SAM template to S3
             │    ├─ Create CloudFormation stack
             │    └─ Provision AWS resources:
             │        ├─ API Gateway
             │        ├─ Lambda functions
             │        ├─ DynamoDB table
             │        ├─ IAM role
             │        └─ CloudWatch logs
             │
             ▼
┌──────────────────────────────────────┐
│ AWS CLOUDFORMATION STACK (Deployed)  │
│                                       │
│  ✓ API Gateway live                 │
│  ✓ Lambda functions ready            │
│  ✓ DynamoDB table created            │
│  ✓ IAM role configured               │
│  ✓ Monitoring enabled                │
│                                       │
└──────────────────────────────────────┘
             │
             ▼
   PRODUCTION READY
   (API endpoint live, accepting requests)
```

---

## 5. Data Models

```
DynamoDB TABLE: PatientReports
════════════════════════════════

Partition Key: PatientID (String)
    Example: "PATIENT-001"

Sort Key: ReportID (String)
    Example: "REPORT-2025-11-001"

Attributes:
┌─────────────────────────────┐
│ PatientID (PK)              │ "PATIENT-001"
│ ReportID (SK)               │ "REPORT-2025-11-001"
├─────────────────────────────┤
│ OriginalText (String)       │ "Patient presents with fever..."
│ Summary (String)            │ "Patient has fever, needs antibiotics"
│ ProcessingStatus (String)   │ "SUCCESS" | "PENDING" | "FAILED"
│ CreatedAt (Number)          │ 1731326400 (unix timestamp)
│ UpdatedAt (Number)          │ 1731326410
│ SourceSystem (String)       │ "MedicalRecord" | "LabSystem"
│ Metadata (Map)              │ {
│                             │   "physician": "Dr. Smith",
│                             │   "department": "Cardiology",
│                             │   "urgency": "HIGH"
│                             │ }
└─────────────────────────────┘

Example Query:
  GET all reports for Patient-001:
  ├─ Query: PatientID = "PATIENT-001"
  ├─ Sort: ReportID (newest first)
  └─ Return: Last 10 reports

Example Write:
  Store new report:
  ├─ Put Item: {
  │   PatientID: "PATIENT-001",
  │   ReportID: "REPORT-2025-11-002",
  │   OriginalText: "...",
  │   Summary: "...",
  │   ProcessingStatus: "SUCCESS",
  │   CreatedAt: <current timestamp>,
  │   ...
  │ }
  └─ TTL: 90 days (auto-delete old records)
```

---

## 6. Security Model

```
USER REQUEST
    │
    ▼
┌─────────────────────────────────┐
│ AUTHENTICATION (Who are you?)   │
│                                 │
│ Method 1: AWS SigV4             │
│  ├─ AWS Access Key ID           │
│  ├─ AWS Secret Access Key       │
│  └─ Request signature           │
│                                 │
│ Method 2: API Key               │
│  └─ Simple key (not recommended)│
└────────────┬────────────────────┘
             │ (verified)
             ▼
┌─────────────────────────────────┐
│ AUTHORIZATION (What can you do?)│
│                                 │
│ API Gateway Authorization       │
│  ├─ Check: Is user authenticated│
│  ├─ Check: Rate limit reached?  │
│  └─ Check: IP whitelisted?      │
│                                 │
│ Lambda Execution (IAM Role)     │
│  ├─ Service: lambda.amazonaws   │
│  ├─ Can do:                      │
│  │  ├─ DynamoDB: Write to table │
│  │  ├─ Bedrock: Invoke model    │
│  │  └─ CloudWatch: Write logs   │
│  └─ Cannot do:                   │
│     ├─ S3 operations             │
│     ├─ Delete resources          │
│     └─ Cross-account access      │
└────────────┬────────────────────┘
             │ (authorized)
             ▼
┌─────────────────────────────────┐
│ EXECUTION (Perform action)      │
│                                 │
│ Lambda code runs with permissions
│  ├─ Access DynamoDB             │
│  ├─ Call Bedrock API            │
│  └─ Write CloudWatch logs       │
└────────────┬────────────────────┘
             │
             ▼
┌─────────────────────────────────┐
│ DATA PROTECTION                 │
│                                 │
│ In Transit:                      │
│  ├─ HTTPS/TLS 1.2 (encrypted)  │
│  └─ VPC endpoints (private)     │
│                                 │
│ At Rest:                         │
│  ├─ DynamoDB SSE (encrypted)    │
│  ├─ Lambda ephemeral (encrypted)│
│  └─ Optional: KMS keys          │
└─────────────────────────────────┘
```

---

## 7. Auto-Scaling in Action

```
NORMAL LOAD (10 requests/sec)
════════════════════════════

API Gateway:    ████ (40% of 10K limit)
Lambda:         ██ (2 concurrent containers)
DynamoDB:       ██ (on-demand, auto-scaling)
Cost:           ~$50/month
Status:         ✓ Green (healthy)


SPIKE LOAD (500 requests/sec)
════════════════════════════

API Gateway:    ████████████ (5% of 10K limit)
Lambda:         ████████████████ (20 concurrent)
DynamoDB:       ██████ (auto-scaled up)
Cost:           ~$100/month
Status:         ✓ Green (auto-scaled)
Response time:  200-500ms (slightly degraded)


EXTREME LOAD (5000 requests/sec)
════════════════════════════════

API Gateway:    ████████████████████████ (50% of 10K limit)
Lambda:         ████████████████████████ (100+ concurrent)
DynamoDB:       ████████████████████████ (fully scaled)
Cost:           ~$1000+/month
Status:         ⚠️ Yellow (at capacity)
Response time:  500ms-2s
Action needed:  Monitor Bedrock throttling, consider caching

CATASTROPHIC (10,000+ requests/sec)
════════════════════════════════════

API Gateway:    ████████████████████████ (100% limit reached)
  ├─ Returns: 429 Too Many Requests
  └─ Clients: Receive retry-after header
Lambda:         Scaling maxed out
DynamoDB:       Requesting unhandled surge pricing
Cost:           Potentially $5000+/month
Status:         ��� Red (overload)
Action needed:  Enable CloudFront caching, implement queue, vertical scaling
```

---

## 8. Cost Breakdown (Example)

```
SCENARIO: 1000 reports processed/day, 100 bytes average

┌──────────────────────────────────────┐
│ MONTHLY COST ESTIMATE                │
├──────────────────────────────────────┤
│ API Gateway:                          │
│  1000 requests/day = 30,000/month    │
│  Cost: 30,000 × $0.35/million        │
│  = $0.01 per month                   │
│                                       │
│ Lambda:                               │
│  30,000 invocations/month             │
│  Duration: 8 sec average × 128 MB    │
│  Cost: ($0.20 per million) + duration│
│  = $0.02 + $1.50 = $1.52             │
│                                       │
│ DynamoDB:                             │
│  30,000 writes/month                  │
│  On-demand: $0.25 per million WCU    │
│  + 5,000 reads: $0.10 per million RCU│
│  = $7.50 + $0.50 = $8.00             │
│                                       │
│ Bedrock:                              │
│  30,000 model calls                   │
│  Titan: ~$0.0004 per 1K tokens input │
│  + ~$0.0012 per 1K tokens output     │
│  Estimate: 200 input + 100 output    │
│  = $1.80 + $3.60 = $5.40             │
│                                       │
│ CloudWatch:                           │
│  Logs: 30,000 requests × 2KB average │
│  = 60GB/month                         │
│  First 5GB free, 55GB × $0.50        │
│  = $27.50                             │
│                                       │
├──────────────────────────────────────┤
│ TOTAL MONTHLY COST:        ~$42.43   │
│ ANNUAL COST:               ~$509     │
└──────────────────────────────────────┘

Scale to 100,000 reports/day:
├─ All components scale proportionally
├─ Lambda cost increases ~2x (more duration)
├─ DynamoDB cost increases ~100x (volume)
├─ Bedrock cost increases ~100x (volume)
└─ ESTIMATED MONTHLY: ~$4,000-5,000
```

---

## 9. Failure Scenarios & Recovery

```
SCENARIO 1: Lambda Function Crashes
═════════════════════════════════════

What happens:
├─ Lambda: Throws unhandled exception
├─ CloudWatch: Logs error
├─ API Gateway: Returns 500 error
├─ Client: Receives error response
└─ DynamoDB: No record written

Recovery:
├─ Automatic: Lambda is retry-able (3x)
├─ Manual: Fix code, push to GitHub
├─ Monitoring: CloudWatch alarm triggers
└─ Time: ~5-10 minutes for fix + redeploy

Prevention:
├─ Error handling: Try/except in code
├─ Input validation: Check before Bedrock call
├─ Logging: Log errors to CloudWatch
└─ Testing: Unit tests + load tests


SCENARIO 2: DynamoDB Table Throttled
═════════════════════════════════════

What happens:
├─ High write volume exceeds capacity
├─ DynamoDB: Returns ThrottlingException
├─ Lambda: Retries (exponential backoff)
├─ Client: May timeout (>30 sec)
└─ Data: Not persisted (transient)

Recovery:
├─ Automatic: On-demand billing auto-scales
├─ Manual: Increase provisioned capacity (if not on-demand)
├─ Monitoring: CloudWatch shows throttle events
└─ Time: <1 minute (automatic)

Prevention:
├─ Use on-demand billing (current setup ✓)
├─ Partition data (different patient IDs)
├─ Archive old data (TTL cleanup ✓)
└─ Monitor metrics: Track consumed WCU


SCENARIO 3: Bedrock Model Unavailable
══════════════════════════════════════

What happens:
├─ Region: Model not available in that region
├─ Bedrock: Returns ServiceUnavailableException
├─ Lambda: Retries (3x, then fails)
├─ Client: Receives error after ~15 seconds
└─ Impact: All summarization stops

Recovery:
├─ Automatic: AWS brings service back (usually <5 min)
├─ Manual: Fallback to secondary region
├─ Monitoring: CloudWatch detects unavailability
└─ Time: 5-15 minutes typically

Prevention:
├─ Multi-region setup (not current)
├─ Fallback logic: Simple summary if Bedrock fails
├─ Health checks: Verify Bedrock endpoint
└─ Capacity: Request limit increase with AWS TAM


SCENARIO 4: Complete Region Outage
═══════════════════════════════════

What happens:
├─ AWS region: All services down (rare)
├─ API Gateway: Not responding
├─ All components: Completely unavailable
├─ Data: Safe in DynamoDB
└─ Impact: Complete service outage

Recovery:
├─ Automated (current): None (single region)
├─ Manual: Deploy to secondary region
├─ Data: Restore from backup (PITR)
└─ Time: 30-60 minutes (manual redeploy)

Prevention:
├─ Multi-region failover (future enhancement)
├─ Automated backups to different region
├─ Route53 health checks
└─ Cost-benefit analysis needed ($$ increase)

Estimated downtime:
├─ Current setup (single region): RTO 30-60 min
├─ Multi-region (future): RTO 5-10 min
└─ Acceptable? Depends on business requirements
```

---

## 10. Interview Whiteboard Sketch

```
How to explain in an interview (60 seconds):

"Here's how the system works:

1. CLIENT sends medical report to API Gateway
   
2. API GATEWAY routes request to Lambda
   (Provides security, throttling, CORS)
   
3. LAMBDA processes:
   ├─ Validates input (size, format)
   ├─ Calls Bedrock AI model for summarization
   └─ Writes result to DynamoDB
   
4. BEDROCK MODELS provides AI summarization
   (Managed by AWS, no infrastructure needed)
   
5. DYNAMODB stores:
   ├─ Original text
   ├─ AI summary
   ├─ Processing status
   └─ Metadata
   
6. RESPONSE returned to client:
   {status, summary, timestamp}

Why Serverless?
├─ No server management needed
├─ Auto-scaling (handles traffic spikes)
├─ Pay-per-use (cost efficient)
└─ Easy CI/CD (GitHub Actions)

Scaling?
├─ API Gateway: 10K req/sec default
├─ Lambda: Auto-spawns containers
├─ DynamoDB: On-demand scaling
└─ Bedrock: Managed service

Security?
├─ IAM: Least privilege role
├─ TLS: All connections encrypted
├─ SSE: Data encrypted at rest
└─ Audit: CloudTrail logs everything

Questions?"
```

---

## Quick Reference: Who Does What?

| Component | Responsibility | If it fails |
|-----------|-----------------|-----------|
| **API Gateway** | HTTP endpoint, auth, throttling | Returns 500, clients can't reach service |
| **Lambda** | Orchestration, Bedrock calls, DB writes | Returns error, report not summarized |
| **Bedrock** | AI summarization | Lambda retries, can implement fallback |
| **DynamoDB** | Store reports and summaries | Data not persisted, retry on auto-scaling |
| **CloudWatch** | Logging and monitoring | Logs disappear, harder to debug |
| **IAM** | Access control | Lambda can't execute, permission denied |

---

**Use this guide to:**
- ✓ Understand the system at a glance
- ✓ Prepare for technical interviews
- ✓ Debug issues quickly
- ✓ Explain to non-technical stakeholders
- ✓ Plan scaling and improvements

**Last Updated**: November 11, 2025
