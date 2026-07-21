# ATS Infrastructure — Deployment Guide (AWS Console)

Step-by-step guide to clean up manually created resources and deploy the
`resume-scan-pipeline` CloudFormation stack using the **AWS Management Console**.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Prerequisites](#prerequisites)
3. [Phase 1 — Delete Manually Created Resources](#phase-1--delete-manually-created-resources)
4. [Phase 2 — Upload Lambda Zip Files to S3](#phase-2--upload-lambda-zip-files-to-s3)
5. [Phase 3 — Deploy CloudFormation Stack](#phase-3--deploy-cloudformation-stack)
6. [Phase 4 — Verify Deployment](#phase-4--verify-deployment)
7. [Parameter Reference](#parameter-reference)
8. [Troubleshooting](#troubleshooting)

---

## Architecture Overview

The `resume-scan-pipeline` stack creates the following resources:

```
GuardDuty Malware Scan Result
        │
        ▼
┌─────────────────────────┐
│   EventBridge Rule      │  Filters scan events for quarantine bucket
│   (GuardDuty trigger)   │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│  Lambda: resume-scan-   │────▶│   SQS Queue      │────▶│ Lambda: resume- │
│  mover                  │     │  (parse queue)   │     │ parser-consumer │
│                         │     │                  │     │                 │
│  Moves file to clean/   │     │  DLQ: parse-dlq  │     │ Extracts text,  │
│  infected bucket        │     └──────────────────┘     │ calls Bedrock   │
└─────────────────────────┘                              └─────────────────┘
```

**Resources created by this template (12 total):**

| Category | Resource | Name Pattern |
|---|---|---|
| SQS | Main queue | `{env}-resume-parse-queue` |
| SQS | Dead letter queue | `{env}-resume-parse-dlq` |
| IAM | Mover Lambda role | `{env}-resume-scan-mover-role` |
| IAM | Parser Lambda role | `{env}-resume-parser-consumer-role` |
| Lambda | Mover function | `{env}-resume-scan-mover` |
| Lambda | Parser function | `{env}-resume-parser-consumer` |
| Lambda | Event source mapping | SQS → Parser Lambda |
| Lambda | Permission | EventBridge → Mover Lambda |
| EventBridge | Rule | `{env}-guardduty-malware-scan-result` |
| CloudWatch | Log group | `/aws/lambda/{env}-resume-scan-mover` |
| CloudWatch | Log group | `/aws/lambda/{env}-resume-parser-consumer` |

---

## Prerequisites

Before deploying, make sure you have:

- [x] AWS account access with IAM permissions (CloudFormation, Lambda, SQS, EventBridge, IAM, S3)
- [x] The 3 S3 buckets already exist:
  - `dev-ats-resumes-quarantine` (GuardDuty scans objects here)
  - `dev-ats-resume` (clean resumes go here)
  - `dev-stagging-ats-resume-infected` (infected resumes go here)
- [x] GuardDuty Malware Protection enabled on the quarantine bucket
- [x] Your `INTERNAL_API_KEY` value (must match `INTERNAL_API_KEY` in ATS-Backend `.env`)
- [x] Your API domain (e.g., `api.athena.analytiqs.io` for dev)

---

## Phase 1 — Delete Manually Created Resources

> **Order matters**: Delete Lambda functions first (they depend on SQS/IAM),
> then EventBridge rule, then SQS queues, then IAM roles.

### Step 1: Delete Lambda Functions

1. Open the **AWS Console** → search for **Lambda** → click **Lambda**
2. You will see a list of functions
3. Find the manually created functions (look for names containing `resume-scan`,
   `resume-parser`, or similar — anything you created manually, NOT the ones
   this template will create)
4. **For each function**:
   - Click the function name to open it
   - Click **Actions** button (top right corner)
   - Click **Delete**
   - Type the function name in the confirmation box
   - Click **Delete**
5. Confirm both Lambda functions are gone from the list

### Step 2: Delete EventBridge Rule

1. Open the **AWS Console** → search for **EventBridge** → click **Amazon EventBridge**
2. In the left sidebar, click **Rules**
3. Make sure the **Event bus** dropdown is set to **default**
4. Find the rule related to GuardDuty malware scan (name likely contains
   `guardduty`, `malware`, `scan-result`, or `resume-scan`)
5. **For each rule**:
   - Select the checkbox next to the rule
   - Click **Delete** (top right)
   - Type `delete` in the confirmation box
   - Click **Delete**

### Step 3: Delete SQS Queues

1. Open the **AWS Console** → search for **SQS** → click **Amazon SQS**
2. You will see a list of queues
3. Find the manually created queues (look for names containing `resume-parse`,
   `resume-queue`, or similar)
4. **For each queue**:
   - Select the checkbox next to the queue
   - Click **Delete** (top right)
   - Type the **queue name** in the confirmation box (this is required)
   - Click **Delete**
5. Delete both the main queue and the dead letter queue (DLQ)

### Step 4: Delete IAM Roles

1. Open the **AWS Console** → search for **IAM** → click **IAM**
2. In the left sidebar, click **Roles**
3. In the search box, type `resume` to filter roles
4. Find the manually created roles (look for names like `resume-scan-mover-role`,
   `resume-parser-consumer-role`, or similar)
5. **For each role**:
   - Click the role name to open it
   - Click **Delete** (right side, under "Summary")
   - Type the **role name** in the confirmation box
   - Click **Delete**
6. Confirm both roles are removed

### Step 5: Verify Cleanup

Go back to each service and confirm nothing resume-related remains:
- **Lambda** → no resume-scan/resume-parser functions
- **EventBridge** → no guardduty scan result rules
- **SQS** → no resume-parse queues
- **IAM** → no resume-scan/resume-parser roles

---

## Phase 2 — Upload Lambda Zip Files to S3

You need to upload 2 zip files from your local machine to the S3 bucket.

### Step 1: Open S3 Bucket

1. Open the **AWS Console** → search for **S3** → click **Amazon S3**
2. Click on the bucket named **`fission-athena-cf-templates`**
3. You should see existing files (cf-templates subfolders, etc.)

### Step 2: Upload the Mover Lambda Zip

1. Click the **Upload** button (top right)
2. Click **Add files**
3. Navigate to your local project folder:
   ```
   ATS-Infra/lambdas/resume_scan_mover/dist/resume_scan_mover.zip
   ```
4. Select the `resume_scan_mover.zip` file → click **Open**
5. Scroll down and click **Upload** (bottom of page)
6. Wait for the green banner: **"Upload succeeded"**
7. Click **Close**

### Step 3: Upload the Parser Lambda Zip

1. Click **Upload** again
2. Click **Add files**
3. Navigate to your local project folder:
   ```
   ATS-Infra/lambdas/resume_parser_consumer/dist/resume_parser_consumer.zip
   ```
4. Select the `resume_parser_consumer.zip` file → click **Open**
5. Scroll down and click **Upload**
6. Wait for **"Upload succeeded"**
7. Click **Close**

### Step 4: Upload the CloudFormation Template

1. Click **Upload** again
2. Click **Add files**
3. Navigate to:
   ```
   ATS-Infra/cloudformation/resume-scan-pipeline.yaml
   ```
4. Select the file → click **Open**
5. Click **Upload**
6. Wait for **"Upload succeeded"**
7. Click **Close**

### Step 5: Verify Uploads

You should now see these 3 files in the `fission-athena-cf-templates` bucket:

```
fission-athena-cf-templates/
├── resume_scan_mover.zip          (~1 KB)
├── resume_parser_consumer.zip     (~30 MB)
└── resume-scan-pipeline.yaml      (CloudFormation template)
```

---

## Phase 3 — Deploy CloudFormation Stack

### Step 1: Start Stack Creation

1. Open the **AWS Console** → search for **CloudFormation** → click **AWS CloudFormation**
2. You should see your existing stacks (master stack, nested stacks, etc.)
3. Click **Create stack** → click **With new resources (standard)**

### Step 2: Specify Template

1. Under "Prepare template", select: **Template is ready**
2. Under "Template source", select: **Amazon S3 URL**
3. Paste this URL in the text box:
   ```
   https://fission-athena-cf-templates.s3.ap-south-1.amazonaws.com/resume-scan-pipeline.yaml
   ```
4. Click **Next**

### Step 3: Stack Details

Fill in each parameter on the "Specify stack details" page:

| Parameter | Value to Enter | Notes |
|---|---|---|
| **Stack name** | `resume-scan-pipeline-dev` | Stack name in CloudFormation |
| **Environment** | `dev` | Short environment prefix |
| **QuarantineBucketName** | `dev-ats-resumes-quarantine` | Already exists |
| **CleanBucketName** | `dev-ats-resume` | Already exists |
| **InfectedBucketName** | `dev-stagging-ats-resume-infected` | Already exists |
| **ApiDomain** | `api.athena.analytiqs.io` | Your dev API host (no `https://`) |
| **ApiPath** | `/api/v1` | Keep default |
| **InternalApiKey** | *(your secret key)* | Must match backend `.env` INTERNAL_API_KEY |
| **BedrockModelId** | `apac.anthropic.claude-3-5-sonnet-20241022-v2:0` | Keep default |
| **LambdaCodeS3Bucket** | `fission-athena-cf-templates` | Bucket with zip files |
| **MoverLambdaS3Key** | `resume_scan_mover.zip` | Keep default |
| **ParserLambdaS3Key** | `resume_parser_consumer.zip` | Keep default |
| **MoverLambdaTimeoutSeconds** | `60` | Keep default |
| **ParserLambdaTimeoutSeconds** | `90` | Keep default |
| **ParserLambdaMemoryMb** | `512` | Keep default |
| **ResumeParseQueueVisibilityTimeoutSeconds** | `540` | Keep default (6x parser timeout) |

5. Click **Next**

### Step 4: Stack Options

1. **Tags** (optional but recommended):
   - Click **Add tag**
   - Key: `Project` | Value: `fission-ats`
2. **Permissions** — Leave as default (uses your current console role)
3. **Advanced options** — Leave all defaults
4. Click **Next**

### Step 5: Review and Deploy

1. Review all parameters — scroll down to verify they are correct
2. Under **Capabilities** section at the bottom:
   - Check the box: **"I acknowledge that AWS CloudFormation might create IAM resources with custom names"**
     (This is required because the template creates named IAM roles)
3. Click **Submit**

### Step 6: Wait for Deployment

1. You will be taken to the stack's **Events** tab
2. Watch the events — resources will be created in order:
   ```
   CREATE_IN_PROGRESS  resume-scan-pipeline-dev  CloudFormation stack
   CREATE_IN_PROGRESS  ResumeParseDLQ            SQS Queue
   CREATE_IN_PROGRESS  ResumeParseQueue          SQS Queue
   CREATE_IN_PROGRESS  MoverLambdaRole           IAM Role
   CREATE_IN_PROGRESS  ParserLambdaRole          IAM Role
   CREATE_IN_PROGRESS  MoverLambdaLogGroup       Log Group
   CREATE_IN_PROGRESS  ParserLambdaLogGroup      Log Group
   CREATE_IN_PROGRESS  ResumeScanMoverFunction   Lambda Function
   CREATE_IN_PROGRESS  ResumeParserConsumerFunc  Lambda Function
   CREATE_IN_PROGRESS  ParserSqsEventSourceMap   Event Source Mapping
   CREATE_IN_PROGRESS  GuardDutyScanResultRule   EventBridge Rule
   CREATE_IN_PROGRESS  EventBridgeInvokeMoverP   Lambda Permission
   ```
3. Wait for **"Stack CREATE_COMPLETE"** — this typically takes 2-5 minutes
4. If any resource fails, check the **Events** tab for the error message

---

## Phase 4 — Verify Deployment

### Verify Lambda Functions

1. Go to **Lambda → Functions**
2. Confirm these functions exist:
   - `dev-resume-scan-mover`
   - `dev-resume-parser-consumer`
3. Click each → **Configuration** → **Environment variables** → verify the
   bucket names and SQS queue URL are set correctly

### Verify SQS Queues

1. Go to **SQS → Queues**
2. Confirm these queues exist:
   - `dev-resume-parse-queue`
   - `dev-resume-parse-dlq`
3. Click the main queue → **Dead-letter queue** tab → confirm DLQ is attached
   with max receive count = 5

### Verify EventBridge Rule

1. Go to **EventBridge → Rules**
2. Confirm this rule exists:
   - `dev-guardduty-malware-scan-result`
3. Click the rule → verify:
   - **Event bus**: default
   - **State**: Enabled
   - **Target**: `dev-resume-scan-mover` Lambda function

### Verify IAM Roles

1. Go to **IAM → Roles**
2. Search for `resume`
3. Confirm these roles exist:
   - `dev-resume-scan-mover-role`
   - `dev-resume-parser-consumer-role`
4. Click each → **Permissions** tab → verify the inline policies are attached

### Verify CloudWatch Logs

1. Go to **CloudWatch → Log groups**
2. Search for `resume`
3. Confirm these log groups exist:
   - `/aws/lambda/dev-resume-scan-mover`
   - `/aws/lambda/dev-resume-parser-consumer`

---

## Parameter Reference

| Parameter | Default | Description |
|---|---|---|
| `Environment` | `dev` | Prefix for all resource names |
| `QuarantineBucketName` | `dev-ats-resumes-quarantine` | S3 bucket GuardDuty scans |
| `CleanBucketName` | `dev-ats-resume` | S3 bucket for clean resumes |
| `InfectedBucketName` | `dev-stagging-ats-resume-infected` | S3 bucket for infected files |
| `ApiDomain` | *(required)* | ATS backend API host |
| `ApiPath` | `/api/v1` | API path prefix |
| `InternalApiKey` | *(required)* | Shared secret for internal API calls |
| `BedrockModelId` | `apac.anthropic.claude-3-5-sonnet-20241022-v2:0` | Bedrock model for resume parsing |
| `LambdaCodeS3Bucket` | *(required)* | S3 bucket with Lambda zip files |
| `MoverLambdaS3Key` | `resume_scan_mover.zip` | S3 key for mover Lambda code |
| `ParserLambdaS3Key` | `resume_parser_consumer.zip` | S3 key for parser Lambda code |
| `MoverLambdaTimeoutSeconds` | `60` | Mover Lambda timeout |
| `ParserLambdaTimeoutSeconds` | `90` | Parser Lambda timeout |
| `ParserLambdaMemoryMb` | `512` | Parser Lambda memory (PyMuPDF needs more) |
| `ResumeParseQueueVisibilityTimeoutSeconds` | `540` | SQS visibility timeout (>= 6x parser timeout) |

---

## Troubleshooting

### Stack creation fails on IAM resources

**Error**: `CloudFormation can't create IAM resources with custom names`

**Fix**: Make sure you checked the IAM capabilities checkbox in Step 5.
Also ensure no manually created IAM roles with the same names exist —
delete them first (Phase 1, Step 4).

### Stack creation fails on Lambda resources

**Error**: `S3 object does not exist`

**Fix**: Verify the Lambda zip files were uploaded to the correct S3 bucket
(`fission-athena-cf-templates`) and the S3 keys match exactly:
- `resume_scan_mover.zip`
- `resume_parser_consumer.zip`

### Stack creation fails on SQS resources

**Error**: `Queue already exists`

**Fix**: The manually created SQS queues still exist. Go to SQS → delete
them, then try the stack creation again.

### EventBridge rule not triggering

**Fix**: Verify:
1. GuardDuty Malware Protection is enabled on the quarantine bucket
2. The EventBridge rule is in the **default** event bus
3. The rule is in **Enabled** state
4. The quarantine bucket name in the event pattern matches exactly

### Parser Lambda times out

**Fix**: Increase `ParserLambdaTimeoutSeconds` and
`ResumeParseQueueVisibilityTimeoutSeconds` (keep it >= 6x the parser timeout).
Re-deploy the stack with updated parameter values.

### Re-deploying after code changes

1. Rebuild the Lambda zip: `cd lambdas/resume_scan_mover && ./build.sh`
2. Upload the new zip to S3 (overwrite the existing file)
3. Go to **Lambda → Functions → {function name} → Code** tab
4. Click **Actions → Deploy from Amazon S3** → paste the S3 URL of the zip
5. OR re-deploy the CloudFormation stack (it will detect the change)

> **Note**: CloudFormation does NOT auto-detect S3 object changes. If you reuse
> the same S3 key, use `aws lambda update-function-code` or redeploy from the
> Lambda console directly.

---

## Quick Reference — Console URLs

| Service | URL |
|---|---|
| CloudFormation | https://console.aws.amazon.com/cloudformation/ |
| Lambda | https://console.aws.amazon.com/lambda/ |
| SQS | https://console.aws.amazon.com/sqs/ |
| EventBridge | https://console.aws.amazon.com/events/ |
| IAM | https://console.aws.amazon.com/iam/ |
| S3 | https://console.aws.amazon.com/s3/ |
| CloudWatch Logs | https://console.aws.amazon.com/cloudwatch/ 
