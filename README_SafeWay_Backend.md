# SafeWay Backend

## Table of Contents
- [Problem Statement](#problem-statement)
- [Target Users](#target-users)
- [Core Features (Implemented in Prototype)](#core-features-implemented-in-prototype)
- [Future Features (Planned for Real Deployment)](#future-features-planned-for-real-deployment)
- [Anticipated Impact](#anticipated-impact)
- [Generative AI Contribution](#generative-ai-contribution)
- [System Architecture](#system-architecture)
- [AWS Services Used](#aws-services-used)
- [Lambda Functions](#lambda-functions)
  - [6.1 uploadImage](#61-uploadimage)
  - [6.2 processImage](#62-processimage)
  - [6.3 Bedrock Integration](#63-bedrock-integration)
- [CloudWatch Logs](#cloudwatch-logs)
- [Setup Instructions](#setup-instructions)

---

## Problem Statement
In many Egyptian cities, citizens frequently face dangerous road conditions such as open or elevated manholes, broken streets, and unsafe streetlights (e.g., tilted poles or lights left on during the day). However, reporting these issues is complicated:

- People often don’t know which district authority or department is responsible for a given street, especially in large cities or highways.
- Accessing official complaint numbers or websites is difficult and time-consuming.
- As a result, many citizens choose the easier option of posting on Facebook, where the report’s reach is limited to friends or group members, and the issue rarely reaches the responsible authority.

This creates a gap between citizens and local authorities, delaying fixes and putting communities at risk.

## Target Users
- **Ordinary citizens**: Anyone encountering hazards in streets (drivers, pedestrians, parents with children) who need a simple and fast way to report issues without navigating government bureaucracy.  
- **Local authorities & municipalities**: Government departments (e.g., district offices, electricity authorities, sanitation departments) who need structured, location-based reports instead of random social media posts.  
- **Community members**: People who want to stay informed about dangers in their area (e.g., potholes on their daily route) or vote/confirm when issues are fixed.  

## Core Features (Implemented in Prototype)
- **Image-based Reporting** – Users upload a photo of the issue (e.g., broken road, open manhole, streetlight hazard).  
- **Automated Classification** – AWS Rekognition detects whether the image belongs to one of the supported issue types.  
- **Content Moderation** – Automatic rejection of inappropriate or irrelevant images using Rekognition moderation.  
- **Location Detection** – GPS coordinates from the photo are reverse-geocoded into a full address and mapped to the correct authority.  
- **Authority Routing** – Each issue type and region is mapped to the responsible authority (e.g., electricity department, district municipality).  
- **Citizen Verification** – Users can vote if an issue is resolved, helping to track status without waiting for government feedback.  
- **Cloud-native Architecture** – Built fully on AWS (S3, Lambda, DynamoDB, Rekognition, Location Service, API Gateway).  

## Future Features (Planned for Real Deployment)
- **Mobile Alerts** – Notify users when they approach unresolved hazards on their route.  
- **Authority Feedback Integration** – Direct communication with government authorities to update status officially.  
- **Generative AI Summaries** – Use Amazon Bedrock to auto-summarize complaints for each authority in natural language (Arabic + English).  
- **Automatic Face & License Plate Blurring (Privacy-by-Design)** – Detect faces and vehicle plates, then blur them before storage/display to protect citizen privacy and comply with local norms.  
- **Image Enhancement** – Apply Bedrock or custom ML models to improve low-quality citizen photos (blurred/night images).  
- **Expanded Issue Types** – Train ML models to cover more categories (flooding, garbage, illegal construction, etc.).  
- **Multi-language & Dialect Support** – Generative AI chatbot to allow reporting via voice or dialect text (Egyptian Arabic, etc.).  
- **Analytics Dashboard** – Authorities can view heatmaps of issues and trends for better planning.  

## Anticipated Impact
- **Citizen Safety**: Faster reporting → quicker government response → reduced accidents from open manholes, road cracks, and hazardous light poles.  
- **Government Efficiency**: Automatic classification and routing removes manual sorting, saving officials’ time.  
- **Transparency**: Citizens can track open issues, vote on whether they are resolved, and build public accountability.  
- **Community Engagement**: Encourages crowdsourced monitoring of urban infrastructure, building a sense of shared responsibility.  
- **Inclusivity**: Right-to-left Arabic support and simple UX lowers barriers for elderly and less tech-savvy citizens.  
- **Scalability**: Can expand from Cairo pilot → all Egyptian governorates → wider Arab region, addressing a regional need.  

## Generative AI Contribution
We leverage **Amazon Bedrock** to:  
- Generate summaries of citizen reports for government authorities.  
- Auto-translate complaints (Arabic ↔ English).  
- Enhance poor-quality images for better recognition.  
- Provide chatbot-based citizen interaction in Arabic dialects.  

## System Architecture
The system is built serverlessly with AWS managed services, ensuring scalability and cost efficiency.  

## AWS Services Used
- **Amazon S3** – Store uploaded citizen images.  
- **AWS Lambda** – Backend processing (image moderation, classification, routing).  
- **Amazon Rekognition** – Image classification & content moderation.  
- **Amazon DynamoDB** – Store issue metadata, status, and votes.  
- **Amazon Location Service** – Reverse geocoding GPS coordinates.  
- **Amazon API Gateway** – Provide API endpoints for mobile/web app.  
- **Amazon Bedrock** – Generative AI summaries, translations, and image enhancements.  

## Lambda Functions

### 6.1 uploadImage
- Triggered by user upload via API Gateway.  
- Stores image in S3 and metadata in DynamoDB.  

### 6.2 processImage
- Triggered by S3 upload event.  
- Runs content moderation → image classification → authority routing → location mapping.  
- Updates DynamoDB with issue details.  

### 6.3 Bedrock Integration
#### Bedrock Setup
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

This allows the Lambda to call Bedrock models.  

3. **Environment Variables**  
   Add the following environment variables in the Lambda configuration:  
   - `BEDROCK_MODEL_ID` → e.g., `anthropic.claude-3-5-haiku-20241022-v1:0`  
   - (Optional) `BEDROCK_REGION` if different from Lambda’s region.  

4. **Dependencies**  
   - Bedrock is accessed via AWS SDK v3.  
   - Ensure your Lambda deployment package includes:  
     ```bash
     npm install @aws-sdk/client-bedrock-runtime
     ```

5. **Code Integration**  
   Import the Bedrock client inside your Lambda:  

```javascript
const { BedrockRuntimeClient, InvokeModelCommand } = require("@aws-sdk/client-bedrock-runtime");
const bedrock = new BedrockRuntimeClient({ region: process.env.BEDROCK_REGION || process.env.AWS_REGION });

const command = new InvokeModelCommand({
  modelId: process.env.BEDROCK_MODEL_ID,
  contentType: "application/json",
  accept: "application/json",
  body: JSON.stringify({ input: "Summarize issue details..." })
});

const response = await bedrock.send(command);
```

6. **Testing**  
   - Deploy Lambda and trigger with a sample issue.  
   - Validate that Bedrock generates the correct natural-language summary.  

## CloudWatch Logs
- Each Lambda function writes logs to CloudWatch.  
- Check `/aws/lambda/uploadImage` and `/aws/lambda/processImage` for debugging.  

## Setup Instructions
1. Clone repo.  
2. Deploy infrastructure with AWS SAM/CloudFormation.  
3. Configure environment variables.  
4. Upload test images via API.  
5. Validate DynamoDB records and Bedrock summaries.  
