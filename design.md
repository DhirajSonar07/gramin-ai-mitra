# Gramin AI Mitra – System Design Document

## 1. System Overview
Gramin AI Mitra is built as a serverless AI-powered guidance assistant using AWS services.

## 2. High-Level Architecture
Frontend (AWS Amplify)
→ API Gateway
→ AWS Lambda
→ Amazon Bedrock
→ Knowledge Base (RAG over S3 documents)
→ Amazon DynamoDB (logs)

## 3. Component Breakdown
- Amplify: Hosts frontend UI.
- API Gateway: Routes secure requests.
- Lambda: Handles backend logic.
- Bedrock: Generates AI responses.
- Knowledge Base: Retrieves verified scheme documents.
- DynamoDB: Stores logs and session data.

## 4. Data Flow
1. User submits query.
2. API Gateway forwards request to Lambda.
3. Lambda retrieves relevant documents via Knowledge Base.
4. Bedrock generates grounded response.
5. Response returned to user.
6. Logs stored in DynamoDB.

## 5. Responsible AI
- Guardrails enabled at Bedrock level.
- Responses grounded in verified documents.
- Monitoring via logs.

## 6. Scalability & Security
- Serverless architecture supports scaling.
- IAM roles follow least privilege principle.
- Data encrypted in transit.
