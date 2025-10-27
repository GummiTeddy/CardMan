##  About the Project: CardMan – AI-Powered Credit Card Cashback Optimizer


<img width="3798" height="1347" alt="image (2)" src="https://github.com/user-attachments/assets/059dd463-f9dc-40d0-b9cb-acda7a0ff038" />


##  Inspiration

Many people own multiple credit cards but rarely maximize their cashback potential because the reward structure is complex and dynamic: changing by category, time, or even specific merchants. I often found myself hesitating at checkout: “Which card should I use for dining?” or “Does this one give better travel points?”


That everyday frustration inspired CardMan, an intelligent AI agent that automatically identifies the best credit card for each purchase to help users earn more without thinking.

## What It Does

CardMan simplifies credit card management through an AI workflow:


Add Your Cards: Users upload either a photo of their cards or simply type the card name.


The image is parsed using AWS Bedrock’s foundation model via AgentCore to extract card identity and features.


If necessary, an external search (OpenAI gpt-4o-mini) cross-checks reward structures and verifies benefit details.


Detect Purchase Category: When a user uploads a receipt, product photo, or text like “I bought dinner at Chipotle”, CardMan uses natural language reasoning to detect the purchase type (e.g., dining, travel, gas).


Recommend the Best Card: The agent compares all available cards, their cashback multipliers, and category limits, then recommends the optimal card instantly.


\[
\text{BestCard} = \arg\max_{c \in \mathcal{C}} (r_c \times v_t)
\]


where \( r_c \) is the cashback rate for card \( c \), and \( v_t \) is the transaction value in category \( t \).

# CardMan - Installation & Setup Guide

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [AWS Setup](#aws-setup)
3. [Frontend Configuration](#frontend-configuration)
4. [Testing the Application](#testing-the-application)
5. [Troubleshooting](#troubleshooting)

---

## Prerequisites

Before you begin, ensure you have the following:

### Required Accounts & Tools
- **AWS Account** with appropriate permissions
- **OpenAI API Account** (for GPT-4o-mini integration)
- **AWS CLI** installed and configured ([Install Guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html))
- **Python 3.9+** (for local testing)
- **Git** (for cloning the repository)
- A modern web browser (Chrome, Firefox, Safari, or Edge)

### Required AWS Permissions
Your AWS IAM user/role needs permissions for:
- DynamoDB (CreateTable, PutItem, GetItem, Query, Scan, DeleteItem)
- Lambda (CreateFunction, UpdateFunctionCode, InvokeFunction)
- API Gateway (CreateApi, CreateRoute, CreateIntegration)
- IAM (CreateRole, AttachRolePolicy) - for Lambda execution role
- Amazon Bedrock (InvokeModel) - optional, for AI features

---

## AWS Setup

### Step 1: Configure AWS CLI

```bash
# Configure your AWS credentials
aws configure

# Enter your credentials when prompted:
# AWS Access Key ID: [Your Access Key]
# AWS Secret Access Key: [Your Secret Key]
# Default region name: us-east-1
# Default output format: json
```

### Step 2: Create DynamoDB Tables

Create two DynamoDB tables to store card and user data:

#### Table 1: Cards Table
```bash
aws dynamodb create-table \
    --table-name cards \
    --attribute-definitions \
        AttributeName=card_id,AttributeType=S \
    --key-schema \
        AttributeName=card_id,KeyType=HASH \
    --billing-mode PAY_PER_REQUEST \
    --region us-east-1
```

#### Table 2: User-Cards Table
```bash
aws dynamodb create-table \
    --table-name user-cards \
    --attribute-definitions \
        AttributeName=user_id,AttributeType=S \
        AttributeName=card_id,AttributeType=S \
    --key-schema \
        AttributeName=user_id,KeyType=HASH \
        AttributeName=card_id,KeyType=RANGE \
    --billing-mode PAY_PER_REQUEST \
    --region us-east-1
```

**Verify tables were created:**
```bash
aws dynamodb list-tables --region us-east-1
```

### Step 3: Create IAM Role for Lambda

Create an IAM role that allows Lambda to access DynamoDB:

```bash
# Create trust policy file
cat > lambda-trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the IAM role
aws iam create-role \
    --role-name CardManLambdaRole \
    --assume-role-policy-document file://lambda-trust-policy.json

# Attach policies to the role
aws iam attach-role-policy \
    --role-name CardManLambdaRole \
    --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

aws iam attach-role-policy \
    --role-name CardManLambdaRole \
    --policy-arn arn:aws:iam::aws:policy/AmazonDynamoDBFullAccess
```

**Note the Role ARN** (you'll need it in the next step):
```bash
aws iam get-role --role-name CardManLambdaRole --query 'Role.Arn' --output text
```

### Step 4: Create and Deploy Lambda Function

#### Package the Lambda function:
```bash
cd backend

# Create deployment package
zip -r lambda_deployment.zip lambda_function.py dynamodb_operations.py

# If you're on Windows, use PowerShell:
# Compress-Archive -Path lambda_function.py,dynamodb_operations.py -DestinationPath lambda_deployment.zip
```

#### Create the Lambda function:
```bash
# Replace YOUR_ROLE_ARN with the ARN from Step 3
aws lambda create-function \
    --function-name CardManAPI \
    --runtime python3.9 \
    --role YOUR_ROLE_ARN \
    --handler lambda_function.lambda_handler \
    --zip-file fileb://lambda_deployment.zip \
    --timeout 30 \
    --memory-size 256 \
    --environment Variables={CARDS_TABLE=cards,USER_CARDS_TABLE=user-cards} \
    --region us-east-1
```

**To update the function later:**
```bash
aws lambda update-function-code \
    --function-name CardManAPI \
    --zip-file fileb://lambda_deployment.zip \
    --region us-east-1
```

### Step 5: Create API Gateway

#### Create HTTP API:
```bash
aws apigatewayv2 create-api \
    --name CardManAPI \
    --protocol-type HTTP \
    --cors-configuration AllowOrigins="*",AllowMethods="GET,POST,PUT,DELETE,OPTIONS",AllowHeaders="*" \
    --region us-east-1
```

**Note the API ID** from the output, then create an integration:

```bash
# Get your Lambda function ARN
LAMBDA_ARN=$(aws lambda get-function --function-name CardManAPI --query 'Configuration.FunctionArn' --output text --region us-east-1)

# Replace YOUR_API_ID with the API ID from above
aws apigatewayv2 create-integration \
    --api-id YOUR_API_ID \
    --integration-type AWS_PROXY \
    --integration-uri $LAMBDA_ARN \
    --payload-format-version 2.0 \
    --region us-east-1
```

**Note the Integration ID** from the output.

#### Create routes:
```bash
# Replace YOUR_API_ID and YOUR_INTEGRATION_ID with values from above

# GET all cards
aws apigatewayv2 create-route \
    --api-id YOUR_API_ID \
    --route-key "GET /api/cards" \
    --target integrations/YOUR_INTEGRATION_ID \
    --region us-east-1

# POST new card
aws apigatewayv2 create-route \
    --api-id YOUR_API_ID \
    --route-key "POST /api/cards" \
    --target integrations/YOUR_INTEGRATION_ID \
    --region us-east-1

# GET specific card
aws apigatewayv2 create-route \
    --api-id YOUR_API_ID \
    --route-key "GET /api/cards/{card_id}" \
    --target integrations/YOUR_INTEGRATION_ID \
    --region us-east-1

# DELETE card
aws apigatewayv2 create-route \
    --api-id YOUR_API_ID \
    --route-key "DELETE /api/cards/{card_id}" \
    --target integrations/YOUR_INTEGRATION_ID \
    --region us-east-1

# GET user cards
aws apigatewayv2 create-route \
    --api-id YOUR_API_ID \
    --route-key "GET /api/users/{user_id}/cards" \
    --target integrations/YOUR_INTEGRATION_ID \
    --region us-east-1

# POST user card
aws apigatewayv2 create-route \
    --api-id YOUR_API_ID \
    --route-key "POST /api/users/{user_id}/cards" \
    --target integrations/YOUR_INTEGRATION_ID \
    --region us-east-1

# DELETE user card
aws apigatewayv2 create-route \
    --api-id YOUR_API_ID \
    --route-key "DELETE /api/users/{user_id}/cards/{card_id}" \
    --target integrations/YOUR_INTEGRATION_ID \
    --region us-east-1
```

#### Create a deployment stage:
```bash
aws apigatewayv2 create-stage \
    --api-id YOUR_API_ID \
    --stage-name prod \
    --auto-deploy \
    --region us-east-1
```

#### Grant API Gateway permission to invoke Lambda:
```bash
# Replace YOUR_API_ID and YOUR_ACCOUNT_ID
aws lambda add-permission \
    --function-name CardManAPI \
    --statement-id apigateway-invoke \
    --action lambda:InvokeFunction \
    --principal apigateway.amazonaws.com \
    --source-arn "arn:aws:execute-api:us-east-1:YOUR_ACCOUNT_ID:YOUR_API_ID/*/*" \
    --region us-east-1
```

**Get your API Gateway URL:**
```bash
echo "https://YOUR_API_ID.execute-api.us-east-1.amazonaws.com/prod"
```

### Step 6: (Optional) Enable Amazon Bedrock

If you want to use Amazon Bedrock for image analysis:

```bash
# Request model access in the AWS Console:
# 1. Go to Amazon Bedrock console
# 2. Click "Model access" in the left sidebar
# 3. Click "Manage model access"
# 4. Enable: Anthropic Claude models and Amazon Nova models
# 5. Click "Save changes"
```

Add Bedrock permissions to your Lambda role:
```bash
# Create Bedrock policy
cat > bedrock-policy.json << EOF
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
EOF

# Create and attach the policy
aws iam put-role-policy \
    --role-name CardManLambdaRole \
    --policy-name BedrockAccess \
    --policy-document file://bedrock-policy.json
```

---

## Frontend Configuration

### Step 1: Get OpenAI API Key

1. Go to [OpenAI Platform](https://platform.openai.com/)
2. Sign in or create an account
3. Navigate to API Keys: https://platform.openai.com/api-keys
4. Click "Create new secret key"
5. Copy the key (you won't be able to see it again!)

### Step 2: Configure Frontend

Navigate to the `frontend` directory and create your configuration file:

```bash
cd frontend
cp config.example.js config.js
```

Edit `config.js` with your actual values:

```javascript
const CONFIG = {
    // Your API Gateway URL from Step 5
    API_URL: 'https://YOUR_API_ID.execute-api.us-east-1.amazonaws.com/prod',
    
    // Your OpenAI API Key from Step 1
    OPENAI_API_KEY: 'sk-proj-...',
    
    // Optional: OpenAI Prompt configuration (if using custom prompts)
    OPENAI_PROMPT_ID: 'YOUR_PROMPT_ID_HERE',
    OPENAI_PROMPT_VERSION: '1',
    
    // Default user ID (for demo purposes)
    USER_ID: 'demo-user'
};

window.CONFIG = CONFIG;
```

### Step 3: Deploy Frontend

You have several options to host the frontend:

#### Option A: Local Testing
```bash
# Simple Python HTTP server
cd frontend
python3 -m http.server 8000

# Or using Node.js
npx http-server -p 8000

# Open browser to: http://localhost:8000
```

#### Option B: AWS S3 + CloudFront (Recommended for Production)
```bash
# Create S3 bucket
aws s3 mb s3://cardman-frontend-YOUR-UNIQUE-NAME --region us-east-1

# Enable static website hosting
aws s3 website s3://cardman-frontend-YOUR-UNIQUE-NAME \
    --index-document index.html

# Upload files
cd frontend
aws s3 sync . s3://cardman-frontend-YOUR-UNIQUE-NAME --acl public-read

# Get website URL
echo "http://cardman-frontend-YOUR-UNIQUE-NAME.s3-website-us-east-1.amazonaws.com"
```

#### Option C: GitHub Pages
```bash
# 1. Create a new repository on GitHub
# 2. Push your frontend folder
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/YOUR-USERNAME/cardman.git
git push -u origin main

# 3. Enable GitHub Pages in repository settings
# 4. Select main branch and /frontend folder
```

---

## Testing the Application

### Test DynamoDB Connection
```bash
# Test writing to cards table
aws dynamodb put-item \
    --table-name cards \
    --item '{
        "card_id": {"S": "test-card-001"},
        "card_name": {"S": "Test Card"},
        "bank": {"S": "Test Bank"},
        "cashback_categories": {"M": {
            "dining": {"M": {"rate": {"N": "3"}, "description": {"S": "3% on dining"}}}
        }}
    }' \
    --region us-east-1

# Verify data was written
aws dynamodb get-item \
    --table-name cards \
    --key '{"card_id": {"S": "test-card-001"}}' \
    --region us-east-1
```

### Test Lambda Function
```bash
# Test the Lambda function directly
aws lambda invoke \
    --function-name CardManAPI \
    --payload '{"httpMethod": "GET", "path": "/api/cards"}' \
    --region us-east-1 \
    response.json

# View the response
cat response.json
```

### Test API Gateway
```bash
# Replace with your actual API Gateway URL
curl https://YOUR_API_ID.execute-api.us-east-1.amazonaws.com/prod/api/cards
```

### Test Frontend
1. Open the frontend URL in your browser
2. Click the "💳 Cards" tab to see all available cards
3. Click "➕ Add Card" to add a new card
4. Test card recommendation features

---

## Troubleshooting

### Common Issues

#### Issue: "Access Denied" when creating resources
**Solution:** Ensure your IAM user has the necessary permissions listed in Prerequisites.

#### Issue: Lambda function times out
**Solution:** 
- Increase timeout: `aws lambda update-function-configuration --function-name CardManAPI --timeout 60`
- Check CloudWatch Logs for errors

#### Issue: CORS errors in browser
**Solution:**
- Verify API Gateway CORS configuration includes your domain
- Check that Lambda function returns proper CORS headers
- Clear browser cache and try again

#### Issue: "Table does not exist" error
**Solution:**
- Verify table names match environment variables
- Check that tables were created in the correct region
- Run: `aws dynamodb describe-table --table-name cards --region us-east-1`

#### Issue: OpenAI API errors
**Solution:**
- Verify API key is correct in `config.js`
- Check API key has sufficient credits
- Ensure you're not hitting rate limits

#### Issue: Can't access API Gateway URL
**Solution:**
- Verify deployment stage was created
- Check Lambda has permission for API Gateway to invoke it
- Test Lambda function directly first

### Viewing Logs

#### Lambda Logs (CloudWatch):
```bash
# View recent logs
aws logs tail /aws/lambda/CardManAPI --follow --region us-east-1

# View logs from last 10 minutes
aws logs tail /aws/lambda/CardManAPI --since 10m --region us-east-1
```

#### DynamoDB Table Info:
```bash
# Check table status
aws dynamodb describe-table --table-name cards --region us-east-1

# Scan table contents
aws dynamodb scan --table-name cards --region us-east-1
```

### Clean Up Resources (if needed)

To delete all AWS resources and avoid charges:

```bash
# Delete API Gateway
aws apigatewayv2 delete-api --api-id YOUR_API_ID --region us-east-1

# Delete Lambda function
aws lambda delete-function --function-name CardManAPI --region us-east-1

# Delete DynamoDB tables
aws dynamodb delete-table --table-name cards --region us-east-1
aws dynamodb delete-table --table-name user-cards --region us-east-1

# Delete IAM role (detach policies first)
aws iam detach-role-policy --role-name CardManLambdaRole \
    --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam detach-role-policy --role-name CardManLambdaRole \
    --policy-arn arn:aws:iam::aws:policy/AmazonDynamoDBFullAccess
aws iam delete-role-policy --role-name CardManLambdaRole --policy-name BedrockAccess
aws iam delete-role --role-name CardManLambdaRole
```

---

## Additional Resources

- [AWS Lambda Documentation](https://docs.aws.amazon.com/lambda/)
- [Amazon DynamoDB Documentation](https://docs.aws.amazon.com/dynamodb/)
- [API Gateway Documentation](https://docs.aws.amazon.com/apigateway/)
- [Amazon Bedrock Documentation](https://docs.aws.amazon.com/bedrock/)
- [OpenAI API Documentation](https://platform.openai.com/docs/)

## Support

For issues or questions:
1. Check the [Troubleshooting](#troubleshooting) section
2. Review AWS CloudWatch Logs
3. Create an issue in the project repository

---

## Security Best Practices

⚠️ **Important Security Notes:**

1. **Never commit sensitive credentials** to version control
2. **Use environment variables** for API keys in production
3. **Implement authentication** (AWS Cognito) for production deployments
4. **Enable API Gateway throttling** to prevent abuse
5. **Use AWS Secrets Manager** for storing sensitive information
6. **Restrict IAM permissions** to minimum required access
7. **Enable CloudTrail** for audit logging
8. **Use HTTPS** for all connections
9. **Regularly rotate credentials** and access keys
10. **Monitor usage and costs** in AWS Billing Dashboard

---

**Congratulations!** 🎉 You've successfully deployed CardMan. Start adding your credit cards and let AI help you maximize your cashback rewards!

## How It’s Built


**Architecture Overview**


Frontend: Simple web interface where users upload photos or text.


Backend Stack:


Amazon Bedrock (GPT-OSS-120B for text + Nova Pro for image): Core reasoning engine that interprets card and transaction data.


AWS Lambda: Executes business logic and handles API calls.


API Gateway: Connects front-end requests to Lambda functions.


DynamoDB: Stores user card profiles, categories, and usage history.


External Tools: Optional OpenAI GPT-4o-mini integration for supplemental card detail lookups.


All components are modular and stateless, allowing easy scaling and reproducibility.


## What We Learned


Building an autonomous AI agent requires balancing reasoning and retrieval — Bedrock’s AgentCore made orchestration between LLM and APIs seamless.


Prompt engineering and structured reasoning (chain-of-thought-style decomposition) dramatically improved result accuracy.


AWS services such as Lambda + DynamoDB are ideal for lightweight, cost-effective deployments.


##  Challenges


Parsing credit card text and benefit tiers from raw images required strong OCR reasoning — combining Bedrock and GPT-4o-mini improved accuracy.


Handling ambiguous consumption categories (e.g., “Starbucks in Target”) demanded context-sensitive classification logic.


Managing data privacy and security while processing card details was critical — CardMan uses only anonymized metadata and does not store sensitive card numbers.


## Future Work


Integrate real-time transaction data via Plaid or bank APIs.


Expand to multi-country support with regional card benefits.


Add personalized financial insights, helping users optimize spending patterns over time.
