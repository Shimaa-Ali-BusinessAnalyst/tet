# SafeWay – Backend & AWS Setup (Region: us-west-2)

This repository contains detailed instructions for setting up the **SafeWay** backend on AWS,
including S3, Lambda, DynamoDB, Amazon Location, Rekognition, IAM, API Gateway, and Bedrock integration.

---

## Table of Contents
- [0) Prerequisites](#0-prerequisites)
- [1) S3 (Image Storage)](#1-s3-image-storage)
- [2) DynamoDB](#2-dynamodb)
  - [2.1 Table: Reports](#21-table-reports)
  - [2.2 Table: RoutingRules](#22-table-routingrules)
  - [2.3 Table: IssueTypes](#23-table-issuetypes)
- [3) Amazon Location Service](#3-amazon-location-service)
- [4) Rekognition](#4-rekognition)
- [5) IAM Roles & Policies](#5-iam-roles--policies)
  - [5.1 Lambda role: generateUploadUrl-role](#51-lambda-role-generateuploadurl-role)
  - [5.2 Lambda role: processImage-role](#52-lambda-role-processimage-role)
- [6) Lambda Functions](#6-lambda-functions)
  - [6.1 generateUploadUrl](#61-generateuploadurl)
  - [6.2 processImage (S3-triggered)](#62-processimage-s3-triggered)
  - [6.3 Bedrock Integration](#63-bedrock-integration)
- [7) API Gateway (HTTP API)](#7-api-gateway-http-api)
- [8) CloudWatch Logs (Debugging)](#8-cloudwatch-logs-debugging)

---

## 0) Prerequisites
- AWS account with permissions to create: S3, Lambda, API Gateway, DynamoDB, Amazon Location, Rekognition, IAM
- Node.js 20.x locally (only if you plan to zip code yourself)
- Postman (or curl) to test APIs
- All resources are created in **us-west-2**

---

## 1) S3 (Image Storage)

**Bucket:** `safe-way-hackathon`

**Prefixes (folders):**
- `pending/` – raw uploads (temporary, auto-processed by Lambda)
- `approved/` – safe, approved images
- `quarantine/` – rejected/unsafe images

**Recommended Settings:**
- Block Public Access: **ON**
- (Optional) Lifecycle rules:
  - Delete `pending/*` older than 1–2 days
  - Transition/expire `quarantine/*` after N days

**Bucket Notifications:**
- Event: `ObjectCreated:*`
- Prefix filter: `pending/`
- Destination: Lambda: `processImage`

> **Why:** Uploading to `pending/` triggers automated moderation & classification.

---

## 2) DynamoDB

### 2.1 Table: Reports
- Partition key: `reportId` (String)
- No sort key
- On-demand capacity recommended

**Attributes (Document model):**
```json
{
  "reportId": "string",
  "createdAt": "ISO String",
  "updatedAt": "ISO String",
  "status": "PENDING_UPLOAD | REVIEWING | APPROVED | REJECTED",
  "image": { "pendingKey": "...", "approvedKey": "...", "quarantineKey": "..." },
  "location": { "lat": 0, "lng": 0, "address": "...", "regionKey": "..." },
  "issue": { "typeCode": "1a", "typeName": "...", "description": "...", "confidence": 0.9, "labels": [] },
  "issueVotes": { "optional": "Map of counters" },
  "ttl": "optional Number"
}
```

### 2.2 Table: RoutingRules
- PK: `regionKey` (String)
- SK: `issueTypeCode` (String)

**Example item:**
```json
{
  "regionKey": "Cairo/Maadi",
  "issueTypeCode": "3a",
  "authorityName": "معادي District",
  "authorityEmail": "complaints-maadi@example.gov.eg",
  "active": true
}
```

### 2.3 Table: IssueTypes
- PK: `code` (String e.g., 1a, 2a, 3a, 4a)
- Stores canonical names and descriptions

---

## 3) Amazon Location Service
- Place Index (reverse geocoding): `safe-way-index`
- Data provider: Esri or HERE
- Language support: Arabic (`"ar"`)
- Optional second index: `safe-way-index-alt`

**IAM Permissions for Lambda:**
- `geo:SearchPlaceIndexForPosition` on the place index ARN

---

## 4) Rekognition
- No resource creation needed (managed)
- Lambda permissions:
  - `rekognition:DetectModerationLabels`
  - `rekognition:DetectLabels`
  - (Future) `rekognition:DetectCustomLabels`

---

## 5) IAM Roles & Policies

### 5.1 Lambda role: `generateUploadUrl-role`
- Policies:
```json
// DynamoDB (Reports)
{
  "Effect": "Allow",
  "Action": ["dynamodb:PutItem"],
  "Resource": "arn:aws:dynamodb:us-west-2:<ACCOUNT_ID>:table/Reports"
}

// S3 (presign writes to pending)
{
  "Effect": "Allow",
  "Action": ["s3:PutObject"],
  "Resource": "arn:aws:s3:::safe-way-hackathon/pending/*"
}
```
- CloudWatch Logs: standard Lambda execution

### 5.2 Lambda role: `processImage-role`
- S3:
```json
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:DeleteObject"],
  "Resource": "arn:aws:s3:::safe-way-hackathon/pending/*"
},
{
  "Effect": "Allow",
  "Action": ["s3:PutObject"],
  "Resource": [
    "arn:aws:s3:::safe-way-hackathon/approved/*",
    "arn:aws:s3:::safe-way-hackathon/quarantine/*"
  ]
}
```
- DynamoDB (Reports):
```json
{
  "Effect": "Allow",
  "Action": ["dynamodb:GetItem","dynamodb:PutItem","dynamodb:UpdateItem"],
  "Resource": "arn:aws:dynamodb:us-west-2:<ACCOUNT_ID>:table/Reports"
}
```
- Rekognition:
```json
{
  "Effect": "Allow",
  "Action": ["rekognition:DetectModerationLabels", "rekognition:DetectLabels"],
  "Resource": "*"
}
```
- Amazon Location:
```json
{
  "Effect": "Allow",
  "Action": ["geo:SearchPlaceIndexForPosition"],
  "Resource": "arn:aws:geo:us-west-2:<ACCOUNT_ID>:place-index/safe-way-index"
}
```
- CloudWatch Logs: standard

---

## 6) Lambda Functions

### 6.1 generateUploadUrl
- Runtime: Node.js 20.x
- Handler: `index.handler`
- Returns: `{ reportId, objectKey, uploadUrl, requiredHeaders }`
- Env Variables:
  - AWS_REGION = us-west-2
  - BUCKET_NAME = safe-way-hackathon
  - TABLE_NAME = Reports
  - PENDING_PREFIX = pending/
  - PRESIGNED_EXPIRES = 300
  - ALLOWED_MIME = image/jpeg,image/png

### 6.2 processImage (S3-triggered)
- Runtime: Node.js 20.x
- Handler: `index.handler`
- Trigger: S3 ObjectCreated on `pending/`
- Steps:
  1. Load location from DynamoDB (fallback to S3 metadata if present)
  2. Reverse-geocode (Arabic) → address, regionKey
  3. Moderation (reject if unsafe)
  4. DetectLabels → map to issue (1a/2a/3a/4a)
  5. Copy to `approved/` or `quarantine/` with metadata
  6. Update Reports with enriched fields
- Env Variables:
  - AWS_REGION, BUCKET_NAME, TABLE_NAME
  - PENDING_PREFIX, APPROVED_PREFIX, QUARANTINE_PREFIX
  - MODERATION_THRESHOLD = 0.8
  - PLACE_INDEX_NAME = safe-way-index
  - (Optional) PLACE_INDEX_NAME_ALT
- Timeout: 30–60 sec
- Memory: 512–1024 MB

### 6.3 Bedrock Integration
**Setup Steps:**

1. **Enable Bedrock in your AWS account**
   - Go to AWS Console → Amazon Bedrock.
   - Request access to the desired models (e.g., Claude 3.5 Haiku).
   - Wait until the request status changes to Approved.

2. **IAM Role Permissions**
   - Edit the Lambda execution role used by `processImage`.
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

3. **Environment Variables**
   - `BEDROCK_MODEL_ID` → e.g., `anthropic.claude-3-5-haiku-20241022-v1:0`
   - (Optional) `BEDROCK_REGION` if different from Lambda’s region.

4. **Dependencies**
```bash
npm install @aws-sdk/client-bedrock-runtime
```

5. **Code Integration**
```javascript
const { BedrockRuntimeClient, InvokeModelCommand } = require("@aws-sdk/client-bedrock-runtime");
const bedrock = new BedrockRuntimeClient({ region: process.env.BEDROCK_REGION || process.env.AWS_REGION });

bedrock.send(new InvokeModelCommand({...}));
```

6. **Testing**
- Verify that Lambda can successfully call the Bedrock model and return expected outputs.

---

## 7) API Gateway (HTTP API)
- Type: HTTP API
- Stage: dev
- Routes:
  - POST `/generate-Upload-Url` → Lambda `generateUploadUrl`
- CORS:
  - Methods: POST, OPTIONS
  - Headers: Content-Type
  - Origins: * (for demo)

**Test Example (Postman):**
```json
POST https://{apiId}.execute-api.us-west-2.amazonaws.com/dev/generate-Upload-Url
Content-Type: application/json

{
  "mimeType": "image/jpeg",
  "lat": 30.0444,
  "lng": 31.2357
}
```
- Use `uploadUrl` + header `Content-Type: image/jpeg` to PUT image

---

## 8) CloudWatch Logs (Debugging)
- `generateUploadUrl`: inspect parsing errors, missing env vars, IAM denies.
- `processImage`: verify steps:
  - HeadObject / GetItem OK
  - ReverseGeocode OK
  - Moderation labels found?
  - DetectLabels → “Issue mapping: …”
  - “Moved to approved:” or “Moved to quarantine:”
  - Any AccessDenied → fix IAM
