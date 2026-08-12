# Testing & Evaluation Guide

This guide walks you through automated testing and evaluating your Bedrock Flow customer support application using Amazon Bedrock Evaluations.

## Overview

Testing the customer support chatbot involves:
1. Deploying S3 & IAM testing infrastructure (`infrastructure/cloudformation-testing.yaml`).
2. Creating test prompts in `testing/flow-tests-template.json`.
3. Running the evaluation dataset generator script (`testing/generate-eval-dataset.py`).
4. Running an LLM-as-a-judge evaluation job in Amazon Bedrock.

---

## Step 1: Deploy Testing Infrastructure

Deploy the CloudFormation template to create an S3 bucket for evaluation dataset storage and an IAM role for Bedrock Evaluations:

```bash
aws cloudformation create-stack \
  --stack-name bug-report-testing-stack \
  --template-body file://infrastructure/cloudformation-testing.yaml \
  --capabilities CAPABILITY_IAM \
  --region us-east-1
```

Wait for the stack creation to finish:

```bash
aws cloudformation wait stack-create-complete \
  --stack-name bug-report-testing-stack \
  --region us-east-1
```

Retrieve the S3 bucket name and IAM role ARN:

```bash
aws cloudformation describe-stacks \
  --stack-name bug-report-testing-stack \
  --query "Stacks[0].Outputs" \
  --region us-east-1
```

---

## Step 2: Prepare Test Suite

Copy `testing/flow-tests-template.json` to create your test suite file (e.g., `testing/flow-tests.json`):

```json
{
    "flowInputNode": {
        "nodeName": "FlowNode"
    },
    "tests": [
        {
            "id": "test-faq-01",
            "prompt": "What payment methods do you accept?",
            "expected": "Mentions credit cards and debit cards as per online_shop_faq.md."
        },
        {
            "id": "test-bug-01",
            "prompt": "I found a bug where my cart empties when I refresh the page on Safari.",
            "expected": "Prompts for bug description, steps to reproduce, environment, and logs a ticket."
        }
    ]
}
```

---

## Step 3: Generate Evaluation Dataset

Run `testing/generate-eval-dataset.py` to invoke your Bedrock Flow against the test suite and produce a JSONL evaluation file:

```bash
python testing/generate-eval-dataset.py \
  --tests-json testing/flow-tests.json \
  --flow-id <YOUR_BEDROCK_FLOW_ID> \
  --flow-alias-id <YOUR_BEDROCK_FLOW_ALIAS_ID> \
  --out-jsonl testing/eval_dataset.jsonl \
  --region us-east-1
```

Upload the dataset to S3:

```bash
BUCKET_NAME=$(aws cloudformation describe-stacks --stack-name bug-report-testing-stack --query "Stacks[0].Outputs[?OutputKey=='EvalDatasetBucketName'].OutputValue" --output text)
aws s3 cp testing/eval_dataset.jsonl s3://$BUCKET_NAME/eval_dataset.jsonl
```

---

## Step 4: Run Bedrock Evaluation

1. Go to **Amazon Bedrock Console** -> **Model Evaluation** (or **Evaluations**).
2. Create an **Automated Evaluation** job.
3. Select **Bring Your Own Inferences (BYOI)** / **LLM-as-a-judge**.
4. Choose `amazon.nova-pro-v1:0` as the judge model.
5. Provide the S3 URI (`s3://<BUCKET_NAME>/eval_dataset.jsonl`) and execution role ARN (`bedrock-eval-role`).
6. Submit job and review evaluation metrics.
