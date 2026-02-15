# Gramin AI Mitra - Design Document

## 1. System Overview

Gramin AI Mitra is a serverless, voice-first AI assistant built on AWS that helps rural Indians discover and understand government welfare schemes. The system uses Retrieval-Augmented Generation (RAG) to ensure accurate, grounded responses based on verified government documents.

### Design Principles
- **Simplicity**: Single-agent architecture, no complex orchestration
- **Serverless**: Fully managed services, minimal operational overhead
- **Voice-first**: Optimized for voice interaction with text fallback
- **Scalable**: Auto-scaling to handle variable load
- **Secure**: End-to-end encryption and data protection
- **Cost-effective**: Pay-per-use model suitable for non-profit deployment

## 2. Architecture

### 2.1 High-Level Architecture

```
┌─────────────┐
│   User      │
│  (Voice)    │
└──────┬──────┘
       │
       ▼
┌─────────────────────────────────────────────┐
│         AWS Amplify Hosting                 │
│  (Static Web App + Voice Interface)         │
└──────────────────┬──────────────────────────┘
                   │ HTTPS
                   ▼
┌─────────────────────────────────────────────┐
│         AWS API Gateway (REST)              │
│  - Request validation                       │
│  - Rate limiting                            │
│  - API key management                       │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│         AWS Lambda Functions                │
│                                             │
│  ┌─────────────────────────────┐           │
│  │  Voice Processing Lambda    │           │
│  │  - Speech-to-Text          │           │
│  │  - Text-to-Speech          │           │
│  └─────────────────────────────┘           │
│                                             │
│  ┌─────────────────────────────┐           │
│  │  Conversation Lambda        │           │
│  │  - Session management       │           │
│  │  - Context tracking         │           │
│  └─────────────────────────────┘           │
│                                             │
│  ┌─────────────────────────────┐           │
│  │  RAG Query Lambda           │           │
│  │  - Query processing         │           │
│  │  - Response generation      │           │
│  └─────────────────────────────┘           │
│                                             │
│  ┌─────────────────────────────┐           │
│  │  Profile Management Lambda  │           │
│  │  - User profile CRUD        │           │
│  └─────────────────────────────┘           │
└──────────┬──────────────┬───────────────────┘
           │              │
           ▼              ▼
┌──────────────────┐  ┌──────────────────────┐
│  AWS Bedrock     │  │   DynamoDB Tables    │
│                  │  │                      │
│  ┌────────────┐  │  │  - UserProfiles     │
│  │Claude 3    │  │  │  - Sessions         │
│  │Sonnet      │  │  │  - ConvHistory      │
│  └────────────┘  │  │  - QueryLogs        │
│                  │  └──────────────────────┘
│  ┌────────────┐  │
│  │Guardrails  │  │
│  └────────────┘  │
│                  │
│  ┌────────────┐  │
│  │Knowledge   │  │
│  │Base (RAG)  │  │
│  └─────┬──────┘  │
└────────┼─────────┘
         │
         ▼
┌──────────────────────┐
│    Amazon S3         │
│                      │
│  - Scheme Documents  │
│  - Embeddings        │
│  - Metadata          │
└──────────────────────┘
```

### 2.2 Component Descriptions

#### 2.2.1 Frontend (AWS Amplify)
- **Technology**: React/Next.js static web application
- **Hosting**: AWS Amplify for global CDN distribution
- **Features**:
  - Voice recording and playback interface
  - Text input as fallback
  - Language selection
  - Conversation history display
  - Responsive design for mobile devices

#### 2.2.2 API Gateway
- **Type**: REST API
- **Features**:
  - Request/response validation
  - API key authentication for public access
  - Cognito integration for registered users (optional)
  - Rate limiting (100 requests/minute per user)
  - CORS configuration
  - CloudWatch logging

#### 2.2.3 Lambda Functions

**Voice Processing Lambda**
- Runtime: Python 3.11
- Memory: 1024 MB
- Timeout: 30 seconds
- Responsibilities:
  - Convert speech to text using AWS Transcribe
  - Convert text to speech using AWS Polly
  - Language detection
  - Audio format conversion

**Conversation Lambda**
- Runtime: Python 3.11
- Memory: 512 MB
- Timeout: 15 seconds
- Responsibilities:
  - Session initialization and management
  - Conversation context tracking
  - User intent classification
  - Query routing

**RAG Query Lambda**
- Runtime: Python 3.11
- Memory: 2048 MB
- Timeout: 60 seconds
- Responsibilities:
  - Query preprocessing and enhancement
  - Invoke Bedrock Knowledge Base for retrieval
  - Invoke Claude 3 Sonnet through Bedrock for generation
  - Apply guardrails for safety
  - Format response for user

**Profile Management Lambda**
- Runtime: Python 3.11
- Memory: 256 MB
- Timeout: 10 seconds
- Responsibilities:
  - Create/update user profiles
  - Retrieve profile for personalization
  - Handle consent management
  - Anonymize data for analytics

#### 2.2.4 AWS Bedrock

**Foundation Model: Claude 3 Sonnet**
- Chosen for: Strong multilingual support, reasoning capabilities, and context length
- Configuration:
  - Max tokens: 2048
  - Temperature: 0.3 (low for factual accuracy)
  - Top P: 0.9

**Bedrock Guardrails**
- Content filters:
  - Hate speech detection
  - Violence and harm prevention
  - Sexual content filtering
  - Prompt injection detection
- Topic filters:
  - Block financial advice beyond scheme info
  - Block medical advice
  - Block legal advice
- Word filters:
  - Profanity filtering
- PII redaction:
  - Aadhaar numbers
  - Phone numbers (except for help desk)
  - Bank account details

**Bedrock Knowledge Base (RAG)**
- Vector database: OpenSearch Serverless
- Embedding model: Amazon Titan Embeddings G1 - Text
- Chunk size: 500 tokens with 50 token overlap
- Retrieval: Top 5 relevant documents
- Source attribution: Enabled

#### 2.2.5 Amazon S3

**Knowledge Base Bucket**
- Structure:
  ```
  schemes/
    central/
      agriculture/
      education/
      health/
      women-welfare/
    state/
      maharashtra/
      tamil-nadu/
      ...
  metadata/
    scheme-index.json
    last-updated.json
  ```
- Versioning: Enabled
- Encryption: SSE-S3
- Lifecycle: Archive older versions to Glacier after 90 days

#### 2.2.6 DynamoDB Tables

**UserProfiles Table**
- Partition Key: userId (String)
- Attributes:
  - language (String)
  - state (String)
  - district (String)
  - occupation (String)
  - age (Number)
  - gender (String)
  - createdAt (Number)
  - updatedAt (Number)
- Encryption: AWS managed key
- Billing: On-demand

**Sessions Table**
- Partition Key: sessionId (String)
- Attributes:
  - userId (String)
  - startTime (Number)
  - lastActivity (Number)
  - context (Map)
- TTL: 24 hours
- Billing: On-demand

**ConversationHistory Table**
- Partition Key: userId (String)
- Sort Key: timestamp (Number)
- Attributes:
  - sessionId (String)
  - query (String)
  - response (String)
  - schemes (List)
  - language (String)
- TTL: 90 days
- Billing: On-demand

**QueryLogs Table**
- Partition Key: date (String - YYYY-MM-DD)
- Sort Key: timestamp (Number)
- Attributes:
  - userId (String, anonymized)
  - query (String, PII redacted)
  - responseTime (Number)
  - tokensUsed (Number)
  - error (String, if any)
- TTL: 30 days
- Billing: On-demand
- Used for: Analytics and monitoring

## 3. Data Flow

### 3.1 Voice Query Flow

```
1. User speaks query
   ↓
2. Frontend captures audio
   ↓
3. Audio sent to API Gateway → Voice Processing Lambda
   ↓
4. Lambda uses AWS Transcribe for Speech-to-Text
   ↓
5. Text sent to Conversation Lambda
   ↓
6. Conversation Lambda:
   - Loads session context from DynamoDB
   - Classifies intent
   - Routes to RAG Query Lambda
   ↓
7. RAG Query Lambda:
   - Enhances query with user profile context
   - Queries Bedrock Knowledge Base for relevant schemes
   ↓
8. Knowledge Base:
   - Generates embedding for query
   - Searches S3-backed vector store
   - Returns top 5 relevant document chunks with citations
   ↓
9. RAG Query Lambda:
   - Constructs prompt with retrieved context
   - Calls Claude 3 Sonnet via Bedrock
   - Applies Guardrails to input and output
   - Generates response with citations
   ↓
10. Response sent back to Conversation Lambda
   ↓
11. Conversation Lambda:
    - Updates session context in DynamoDB
    - Saves conversation to history
    - Logs query metrics
   ↓
12. Response sent to Voice Processing Lambda
   ↓
13. Lambda uses AWS Polly for Text-to-Speech
   ↓
14. Audio response sent to Frontend
   ↓
15. Frontend plays audio to user
```

### 3.2 Profile Creation Flow

```
1. User provides profile information (age, occupation, location, etc.)
   ↓
2. Frontend sends to API Gateway → Profile Management Lambda
   ↓
3. Lambda:
   - Validates data
   - Generates userId
   - Encrypts sensitive fields
   - Stores in UserProfiles DynamoDB table
   ↓
4. Returns success response with userId
   ↓
5. Frontend stores userId locally for future sessions
```

### 3.3 Knowledge Base Update Flow

```
1. New scheme documents uploaded to S3 bucket
   ↓
2. S3 triggers Lambda (via EventBridge, configured manually)
   ↓
3. Lambda initiates Knowledge Base sync
   ↓
4. Knowledge Base:
   - Reads new documents from S3
   - Chunks documents
   - Generates embeddings using Titan
   - Updates OpenSearch Serverless index
   ↓
5. New schemes available for querying
```

## 4. Security Design

### 4.1 Authentication and Authorization

**Public Users (Anonymous)**
- API Gateway API keys for basic access control
- Rate limiting: 100 requests/minute per IP
- No PII storage for anonymous users

**Registered Users (Optional)**
- Amazon Cognito User Pools for authentication
- JWT tokens for session management
- Fine-grained access based on user profile

### 4.2 Data Encryption

**In Transit**
- TLS 1.3 for all API communications
- HTTPS-only CloudFront distribution for Amplify

**At Rest**
- DynamoDB: AWS managed encryption keys
- S3: Server-side encryption (SSE-S3)
- Lambda environment variables: AWS KMS encryption

### 4.3 Network Security

- Lambda functions in VPC (optional, if accessing VPC resources)
- Security groups restrict access to necessary ports only
- API Gateway resource policies limit access to specific IP ranges (if needed)
- S3 bucket policies restrict access to Lambda execution roles only

### 4.4 IAM Roles and Policies

**Lambda Execution Roles** (Least Privilege)
```
VoiceProcessingRole:
  - transcribe:StartStreamTranscription
  - polly:SynthesizeSpeech
  - logs:CreateLogGroup, PutLogEvents
  
ConversationRole:
  - dynamodb:GetItem, PutItem (Sessions, ConvHistory)
  - logs:CreateLogGroup, PutLogEvents
  
RAGQueryRole:
  - bedrock:InvokeModel
  - bedrock:Retrieve (Knowledge Base)
  - bedrock:ApplyGuardrail
  - logs:CreateLogGroup, PutLogEvents
  
ProfileRole:
  - dynamodb:GetItem, PutItem, UpdateItem (UserProfiles)
  - logs:CreateLogGroup, PutLogEvents
```

### 4.5 Secrets Management

- API keys stored in AWS Secrets Manager
- Lambda environment variables encrypted with KMS
- No hardcoded credentials in code

### 4.6 Compliance

**Indian Regulations**
- Digital Personal Data Protection Act 2023
- IT Act 2000
- Data stored in AWS Mumbai region (ap-south-1)

**AWS Compliance Programs**
- SOC 2
- ISO 27001
- PCI DSS (if payment features added later)

### 4.7 Monitoring and Incident Response

- CloudWatch alarms for:
  - API Gateway 5xx errors
  - Lambda errors and throttling
  - DynamoDB throttling
  - Bedrock API errors
- AWS CloudTrail for audit logging
- GuardDuty for threat detection
- Incident response runbook for security events

## 5. Scalability Design

### 5.1 Horizontal Scaling

**Lambda Functions**
- Concurrent execution limit: 1000 (default, can increase)
- Auto-scaling based on request volume
- Reserved concurrency for critical functions

**DynamoDB**
- On-demand billing mode for automatic scaling
- DAX (DynamoDB Accelerator) for caching if needed
- Global tables for multi-region (future)

**API Gateway**
- 10,000 requests per second default limit
- Automatic scaling, no provisioning needed

**Bedrock**
- Managed service with built-in scaling
- Rate limits: 100 tokens/second per model (can increase)

### 5.2 Caching Strategy

**CloudFront (Amplify)**
- Cache static assets (JS, CSS, images) at edge locations
- TTL: 24 hours

**API Gateway**
- Response caching for common queries (optional)
- TTL: 5 minutes

**DynamoDB**
- Session data cached in Lambda memory (5 min TTL)
- DAX for sub-millisecond reads if needed

**Application-Level Cache**
- Frequently accessed scheme information cached in Lambda
- Warm Lambda containers reduce cold starts

### 5.3 Load Management

**Rate Limiting**
- API Gateway: 100 requests/minute per user
- Burst: 200 requests per user
- Throttling returns HTTP 429 with retry-after header

**Queue Management**
- For bulk operations (future): SQS for async processing
- DLQ (Dead Letter Queue) for failed requests

### 5.4 Performance Optimization

**Lambda**
- Larger memory allocation (2048 MB) for RAG Lambda to reduce latency
- Lambda SnapStart (for Java, if used)
- Provisioned concurrency for critical functions during peak hours

**DynamoDB**
- Projection expressions to fetch only needed attributes
- Batch operations for multiple reads/writes

**Bedrock Knowledge Base**
- Optimized chunk size (500 tokens) for retrieval quality
- Efficient embedding model (Titan)

**Frontend**
- Code splitting and lazy loading
- Service worker for offline capabilities (future)
- Audio compression for faster transmission

### 5.5 Capacity Planning

**Expected Load (Year 1)**
- Daily Active Users: 10,000
- Average queries per user: 3
- Total daily queries: 30,000
- Peak queries per second: ~10 (during peak hours)

**Resource Estimates**
- Lambda invocations: ~100,000/day
- DynamoDB read/write: ~200,000/day
- Bedrock API calls: ~30,000/day
- S3 storage: 50 GB (documents + embeddings)

## 6. Responsible AI Implementation

### 6.1 Bedrock Guardrails Configuration

**Content Filters**
```
Hate Speech: MEDIUM (block harmful content)
Insults: LOW (allow mild criticism)
Sexual: HIGH (block all sexual content)
Violence: MEDIUM (block graphic violence)
```

**Denied Topics**
- Financial investment advice
- Medical diagnosis or treatment
- Legal consultation
- Political endorsements
- Religious advice

**Word Filters**
- Custom profanity list for Indian languages
- Block offensive terms in Hindi, Tamil, Telugu, etc.

**PII Filters**
- Aadhaar number pattern: \d{4}\s\d{4}\s\d{4}
- Phone number pattern: [6-9]\d{9}
- Bank account pattern: \d{9,18}
- Action: Redact and log attempts

### 6.2 Prompt Engineering for Safety

**System Prompt Template**
```
You are Gramin AI Mitra, an AI assistant helping rural Indians find government welfare schemes.

Guidelines:
- Provide accurate information based ONLY on the retrieved documents
- Cite sources for all scheme information
- Use simple language avoiding jargon
- Be respectful and culturally sensitive
- If you don't know something, say "I don't have information about this"
- Never make up scheme names or details
- Never provide financial, medical, or legal advice
- Focus only on scheme discovery and information

User Profile: {profile}
Retrieved Context: {context}
User Query: {query}

Response:
```

### 6.3 Hallucination Prevention

- **Grounding**: All responses grounded in retrieved documents from Knowledge Base
- **Citations**: Every scheme mentioned includes source document
- **Confidence Scoring**: Bedrock Knowledge Base provides relevance scores
- **Fallback**: If confidence < 0.6, respond with "I don't have reliable information"
- **Human Review**: Random sampling of 1% responses for quality check

### 6.4 Bias Mitigation

- **Training Data**: Review scheme documents for discriminatory language
- **Testing**: Test with diverse user profiles (gender, age, location, occupation)
- **Monitoring**: Track scheme recommendation patterns across demographics
- **Feedback Loop**: Users can report bias or incorrect information

### 6.5 Transparency and Explainability

- **Disclosure**: Frontend clearly states "You are chatting with an AI assistant"
- **Limitations**: Explicitly mention "I can provide information but cannot submit applications"
- **Sources**: Show document name and page number for scheme information
- **Reasoning**: Explain eligibility with "You are eligible because..."

### 6.6 User Control and Privacy

- **Consent**: Explicit opt-in for profile creation
- **Data Access**: Users can view their conversation history
- **Data Deletion**: Users can request deletion of their data
- **Anonymity**: Allow usage without creating a profile

### 6.7 Monitoring and Auditing

**CloudWatch Metrics**
- Guardrail blocks per hour
- PII detection attempts
- Error rates by query type
- Response confidence distribution

**Audit Logs**
- All guardrail activations logged
- Failed queries with reasons
- Weekly review of blocked content
- Monthly bias audit reports

## 7. Cost Estimation

### 7.1 Monthly Cost Breakdown (10,000 DAU)

**AWS Bedrock**
- Claude 3 Sonnet: $0.003/1K input tokens, $0.015/1K output tokens
- Average tokens per query: 2,000 input, 1,000 output
- 30,000 queries/day = 900,000 queries/month
- Cost: ~$5,400/month

**Bedrock Knowledge Base**
- Storage: $0.024/GB/month for 50 GB = $1.20/month
- Queries: $0.0004 per query
- Cost: $360/month

**Lambda**
- 100,000 invocations/day = 3M/month
- Average duration: 3 seconds, 1024 MB memory
- Cost: ~$150/month

**DynamoDB**
- On-demand pricing
- 200,000 operations/day = 6M/month
- Cost: ~$75/month

**API Gateway**
- 100,000 requests/day = 3M/month
- Cost: ~$10/month

**AWS Transcribe**
- 30,000 minutes/month (average 1 min per query)
- Cost: ~$240/month

**AWS Polly**
- 30M characters/month
- Cost: ~$120/month

**S3**
- Storage: 50 GB = $1/month
- Requests: Minimal

**Amplify Hosting**
- 10,000 users, average 5 MB/user/month = 50 GB transfer
- Cost: ~$15/month

**CloudWatch**
- Logs and metrics
- Cost: ~$50/month

**Total Estimated Monthly Cost: ~$6,500**
**Cost per user per month: $0.65**

### 7.2 Cost Optimization Strategies

1. **Caching**: Cache common queries to reduce Bedrock calls
2. **Batch Processing**: Batch multiple user requests where possible
3. **Compression**: Compress API responses and audio files
4. **Reserved Capacity**: Use Savings Plans for predictable load
5. **Lifecycle Policies**: Archive old conversation logs to S3 Glacier
6. **Monitor and Alert**: Set CloudWatch alarms for cost anomalies

## 8. Deployment Strategy

### 8.1 Infrastructure as Code

**AWS CloudFormation / CDK**
- Separate stacks for:
  - Network and security (VPC, security groups)
  - Database (DynamoDB tables)
  - Compute (Lambda functions)
  - API (API Gateway)
  - Storage (S3, Knowledge Base)
  - Frontend (Amplify)

### 8.2 CI/CD Pipeline

**GitHub Actions / AWS CodePipeline**
```
1. Code commit to GitHub
   ↓
2. Run unit tests
   ↓
3. Build Lambda deployment packages
   ↓
4. Deploy to DEV environment
   ↓
5. Run integration tests
   ↓
6. Manual approval for PROD
   ↓
7. Deploy to PROD with blue-green deployment
   ↓
8. Run smoke tests
   ↓
9. Monitor CloudWatch metrics
```

### 8.3 Environment Setup

**Development**
- Smaller Bedrock model (Claude Haiku) for cost savings
- Reduced DynamoDB capacity
- Limited Knowledge Base (subset of schemes)

**Staging**
- Same configuration as production
- Test data mimicking production patterns
- Performance and load testing

**Production**
- Full scheme database
- Production-scale resources
- Enhanced monitoring and alerting

### 8.4 Rollback Strategy

- Blue-green deployment with API Gateway stages
- Lambda function versions and aliases
- CloudFormation rollback on failure
- Database backups before schema changes

## 9. Monitoring and Observability

### 9.1 Key Metrics Dashboard

**CloudWatch Dashboard**
- API Gateway request count, latency, errors
- Lambda invocation count, duration, errors, throttles
- DynamoDB read/write capacity, throttles
- Bedrock API calls, latency, errors
- Guardrail blocks and reasons
- Cost metrics from Cost Explorer

### 9.2 Alerting

**Critical Alerts (PagerDuty/SNS)**
- API Gateway 5xx error rate > 5%
- Lambda error rate > 2%
- DynamoDB throttling
- Bedrock API failures
- Cost anomaly > 150% of daily average

**Warning Alerts (Email/Slack)**
- API latency P95 > 5 seconds
- Lambda cold starts > 20%
- Low knowledge base accuracy feedback
- Guardrail block rate spike

### 9.3 Logging Strategy

**Application Logs**
- Structured JSON logging
- Correlation IDs across services
- Log levels: ERROR, WARN, INFO, DEBUG
- Retention: 30 days in CloudWatch, archive to S3

**Audit Logs**
- All API calls logged with CloudTrail
- User actions logged (profile create/update)
- Scheme queries logged (anonymized)
- Retention: 1 year

### 9.4 Tracing

- AWS X-Ray for distributed tracing
- Trace requests across Lambda, API Gateway, Bedrock
- Identify bottlenecks and optimize

## 10. Testing Strategy

### 10.1 Unit Tests
- Lambda function logic
- Input validation
- Error handling
- Mock AWS service calls

### 10.2 Integration Tests
- End-to-end API flow
- DynamoDB read/write operations
- Bedrock API calls with test prompts
- Knowledge Base retrieval accuracy

### 10.3 Load Tests
- Apache JMeter or Locust
- Simulate 1000 concurrent users
- Measure latency under load
- Identify throttling limits

### 10.4 Accuracy Tests
- 100 test queries with known answers
- Measure scheme retrieval accuracy
- Measure response relevance
- Test multilingual capabilities

### 10.5 Security Tests
- OWASP API Security Top 10
- Penetration testing
- Prompt injection attempts
- PII leakage tests

## 11. Disaster Recovery

### 11.1 Backup Strategy

**DynamoDB**
- Point-in-time recovery enabled
- On-demand backups before major changes
- Retention: 35 days

**S3**
- Versioning enabled for all documents
- Cross-region replication (future)
- Lifecycle policy for version cleanup

**Lambda**
- Code stored in version control (GitHub)
- CloudFormation templates backed up

### 11.2 Recovery Procedures

**RTO (Recovery Time Objective): 4 hours**
**RPO (Recovery Point Objective): 1 hour**

**Failure Scenarios**
1. Lambda function failure: Automatic retry, fallback to previous version
2. DynamoDB outage: Serve cached responses, restore from backup
3. Bedrock API unavailable: Queue requests, retry with exponential backoff
4. Region failure: Failover to secondary region (future implementation)

## 12. Known Limitations

1. **Language Support**: Initial release supports Hindi + 3 regional languages
2. **Offline Access**: Requires internet connectivity
3. **Voice Quality**: Accuracy depends on audio quality and background noise
4. **Scheme Coverage**: Limited to schemes with digital documentation
5. **Real-time Updates**: Scheme information updated periodically, not real-time
6. **No Application Submission**: Provides guidance only, cannot submit applications
7. **Context Window**: Limited conversation history (last 5 exchanges)
8. **Token Limits**: Long scheme documents may be truncated

## 13. Future Architecture Enhancements

1. **Multi-region Deployment**: For better latency and disaster recovery
2. **Real-time Updates**: EventBridge scheduled rules for daily scheme updates
3. **Advanced Analytics**: QuickSight dashboards for usage patterns
4. **Feedback Loop**: Collect user feedback to improve responses
5. **A/B Testing**: Test different prompts and models for optimization
6. **Personalized Recommendations**: ML model for proactive scheme suggestions
7. **WhatsApp Integration**: Amazon Pinpoint for messaging
8. **Offline Capabilities**: Progressive Web App with service workers
9. **Voice Biometrics**: For secure user authentication
10. **Multi-modal Input**: Support for image/document upload (future)

## 14. Conclusion

This design provides a simple, scalable, and secure architecture for Gramin AI Mitra using AWS managed services. The serverless approach minimizes operational overhead while the RAG-based system ensures accurate, grounded responses. Bedrock Guardrails and comprehensive monitoring ensure responsible AI deployment. The system is designed to scale from pilot to production while maintaining cost-effectiveness suitable for social impact applications.
