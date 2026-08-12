# Tools Setup Guide

This guide walks you through deploying and configuring the `create_bug_report` tool required by the Customer Support Chatbot.

## Overview

When customers report a bug on the website, the chatbot collects details about the issue and logs a ticket in DynamoDB using an AWS Lambda function.

The tool consists of:
- **DynamoDB Table**: `BugReports-{Suffix}` - stores ticket records (`ticketId`, `description`, `stepsToReproduce`, `environment`, `status`, `createdAt`).
- **AWS Lambda Function**: `create-bug-report-{Suffix}` - implements the tool handler (`src/create_bug_report.py`).
- **IAM Roles & Permissions**: Gives Lambda access to DynamoDB and grants Amazon Bedrock permission to invoke Lambda.

---

## Step 1: Deploy the Tool Infrastructure

Deploy the CloudFormation template located in `infrastructure/cloudformation-tool.yaml`:

```bash
aws cloudformation create-stack \
  --stack-name bug-report-tool-stack \
  --template-body file://infrastructure/cloudformation-tool.yaml \
  --capabilities CAPABILITY_IAM \
  --region us-east-1
```

Wait for the stack creation to complete:

```bash
aws cloudformation wait stack-create-complete \
  --stack-name bug-report-tool-stack \
  --region us-east-1
```

Retrieve the stack outputs (Lambda Function ARN and DynamoDB Table Name):

```bash
aws cloudformation describe-stacks \
  --stack-name bug-report-tool-stack \
  --query "Stacks[0].Outputs" \
  --region us-east-1
```

---

## Step 2: Tool Implementation Details

The Lambda code (located at `src/create_bug_report.py`) handles `messageVersion: 1.0` Bedrock Agent function calls.

### Expected Parameters

1. `description` (Required): Detailed description of the bug.
2. `stepsToReproduce` (Optional/Recommended): Steps taken by the user leading to the bug.
3. `environment` (Optional/Recommended): Operating system, browser, or platform details.

### Sample Payload

```json
{
  "messageVersion": "1.0",
  "function": "create_bug_report",
  "actionGroup": "BugReportGroup",
  "parameters": [
    { "name": "description", "value": "Checkout button freezes on step 2" },
    { "name": "stepsToReproduce", "value": "Add item to cart, proceed to step 2, click pay" },
    { "name": "environment", "value": "Chrome 120 on Windows 11" }
  ]
}
```

---

## Step 3: Integrate with Amazon Bedrock

1. Open the **Amazon Bedrock Console** -> **Agents** (or **Flows**).
2. Add an Action Group named `BugReportGroup`.
3. Select **Define with function details** and set the function name to `create_bug_report`.
4. Attach the deployed Lambda function (`create-bug-report-*`).
5. Save and prepare the Agent / Flow.
