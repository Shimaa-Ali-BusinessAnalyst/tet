# SafeWay Backend

## Table of Contents
- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [Lambda Functions](#lambda-functions)
- [S3 Buckets](#s3-buckets)
- [DynamoDB Tables](#dynamodb-tables)
- [API Gateway](#api-gateway)
- [Bedrock Setup](#bedrock-setup)
- [Deployment](#deployment)
- [Testing](#testing)

---


## Project Overview
This project provides ...

## Architecture
The architecture consists of ...

## Lambda Functions
1. processImage → Handles image upload and classification.
2. notifyAuthority → Sends alerts to the responsible authority.

## S3 Buckets
- safeway-uploads → Stores user-uploaded images.
- safeway-processed → Stores processed images.

## DynamoDB Tables
- ReportsTable → Stores report metadata and classification results.

## API Gateway
- Provides endpoints for mobile app → Lambda integration.

## Bedrock Setup
1. Enable Bedrock in your AWS account
   - Go to AWS Console → Amazon Bedrock.
   - Request access to the desired models (e.g., Claude 3.5 Haiku).
   - Wait until the request status changes to Approved.

2. IAM Role Permissions
   - Edit the Lambda execution role used by processImage.
   - Attach the following inline policy (or extend an existing one):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream"
      ],
      "Resource": "*"
    }
  ]
}
```
   - This allows the Lambda to call Bedrock models.

3. Environment Variables
   - BEDROCK_MODEL_ID → e.g., anthropic.claude-3-5-haiku-20241022-v1:0
   - (Optional) BEDROCK_REGION if different from Lambda’s region.

4. Dependencies
   - Bedrock is accessed via AWS SDK v3.
   - Ensure your Lambda deployment package includes:
```bash
npm install @aws-sdk/client-bedrock-runtime
```

5. Code Integration
   - Import the Bedrock client inside your Lambda:
```js
const { BedrockRuntimeClient, InvokeModelCommand } = require("@aws-sdk/client-bedrock-runtime");
const bedrock = new BedrockRuntimeClient({ region: process.env.BEDROCK_REGION || process.env.AWS_REGION });
```
   - Use bedrock.send(new InvokeModelCommand({...})) to generate natural-language text (e.g., risk severity explanation, summary message).

6. Testing
- Upload test images through the app.
- Validate reports are stored in DynamoDB and notifications are triggered.
