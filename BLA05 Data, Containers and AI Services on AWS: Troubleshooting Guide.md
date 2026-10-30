# AWSGenAIdeveloper · BLA-05

## Data, Containers and AI Services on AWS: Troubleshooting Guide

**LinkedIn Activity:** [link 1] , [link 2] , [link 3]
> **View Tutorial on YouTube:** [link 1] , [link 2] , [link 3]

Three troubleshooting walkthroughs based on the labs of the AWS Certified Generative AI Developer – Professional course (*Other Services You Should Know*: Analytics; Compute, Container and Customer Engagement; Database; Developer Tools; Machine Learning).

> Scenario-based walkthroughs written from the course labs and AWS documentation. 

---

## 1: Kinesis Records Go In, but the Consumer Sees Nothing

**Related lab:** Amazon Kinesis Data Streams
**Concepts:** shards, partition keys, shard iterators, base64 encoding, AWS CLI v2

### Scenario
A stream called `chat-events` will carry customer chat messages to a Lambda function that scores sentiment with Bedrock. Before wiring up Lambda, records are sent and read back with the AWS CLI.

```bash
aws kinesis create-stream --stream-name chat-events --shard-count 1
aws kinesis put-record --stream-name chat-events \
  --partition-key user-42 --data '{"msg":"My parcel is late"}'
```

### Symptoms
First, `put-record` fails:
```
Invalid base64: "{"msg":"My parcel is late"}"
```
Once records are going in, reading them back returns nothing:
```json
{ "Records": [], "NextShardIterator": "AAAAAAAAAAH...", "MillisBehindLatest": 0 }
```
And when records do come back, the `Data` field looks like scrambled text: `eyJtc2ciOiJNeSBwYXJjZWwgaXMgbGF0ZSJ9`.

### Diagnosis
Three separate causes:

| Symptom | Cause |
|---|---|
| `Invalid base64` | AWS CLI **v2** expects `--data` to be base64 unless told otherwise |
| Empty `Records` | The shard iterator was `LATEST` — it only returns records added *after* the iterator was created |
| Scrambled `Data` | Kinesis returns the record body **base64-encoded** — it isn't corrupted |

### Fix
1. Tell the CLI the data is raw text:
   ```bash
   aws kinesis put-record --stream-name chat-events \
     --partition-key user-42 --data '{"msg":"My parcel is late"}' \
     --cli-binary-format raw-in-base64-out
   ```
2. Read from the **start** of the shard with `TRIM_HORIZON`:
   ```bash
   SHARD_IT=$(aws kinesis get-shard-iterator --stream-name chat-events \
     --shard-id shardId-000000000000 --shard-iterator-type TRIM_HORIZON \
     --query ShardIterator --output text)
   aws kinesis get-records --shard-iterator "$SHARD_IT"
   ```
3. Decode the data to read it:
   ```bash
   echo "eyJtc2ciOiJNeSBwYXJjZWwgaXMgbGF0ZSJ9" | base64 --decode
   ```
   In a Lambda consumer the same applies: `base64.b64decode(record["kinesis"]["data"])`.

> **Later at scale:** if `ProvisionedThroughputExceededException` appears, check the partition key. If every record uses the same key, one shard does all the work. Use a key with many values (user or session ID), add shards, or switch the stream to on-demand capacity.

### Verification
`get-records` returns the record, and decoding `Data` prints `{"msg":"My parcel is late"}`.

---

## 2: ECS Tasks Keep Stopping — Then the App Can't Call Bedrock

**Related labs:** Creating ECS Cluster · Creating ECS Service
**Concepts:** ECS services and tasks, Fargate, task execution role vs task role, ECR, CloudWatch Logs

### Scenario
A small summarisation API is packaged as a container, pushed to Amazon ECR, and run as an ECS **service** on Fargate. The app calls Amazon Bedrock.

### Symptoms
The service never becomes stable — tasks start, then stop, over and over. Once that's fixed, the app runs but every request fails:
```
AccessDeniedException: User: arn:aws:sts::111122223333:assumed-role/summariser-exec-role/...
is not authorized to perform: bedrock:InvokeModel
```

### Diagnosis
Find out why the last task stopped:

```bash
aws ecs list-tasks --cluster genai-cluster --desired-status STOPPED
aws ecs describe-tasks --cluster genai-cluster --tasks TASK_ARN \
  --query "tasks[0].{reason:stoppedReason,containers:containers[].reason}"
```

```
CannotPullContainerError: ... unable to retrieve ecr registry auth
```

Two different roles are involved, and each error points to one:

| Role | Used by | Needs |
|---|---|---|
| **Task execution role** | ECS itself | Pull from ECR, write logs → `AmazonECSTaskExecutionRolePolicy` |
| **Task role** | Your code inside the container | Call AWS services → `bedrock:InvokeModel` |

The first error means the execution role can't pull the image (or a task in a private subnet has no route to ECR). The second error names `summariser-exec-role` — the Bedrock permission was added to the **execution** role, but the app runs with the **task** role.

### Fix
1. Give the execution role the managed policy for pulling images and writing logs:
   ```bash
   aws iam attach-role-policy --role-name summariser-exec-role \
     --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
   ```
   If tasks run in a private subnet, add a NAT gateway or VPC endpoints for ECR and CloudWatch Logs.
2. Create a **task role** with only the Bedrock permission the app needs:
   ```json
   {
     "Effect": "Allow",
     "Action": "bedrock:InvokeModel",
     "Resource": "arn:aws:bedrock:us-east-1::foundation-model/amazon.nova-lite-v1:0"
   }
   ```
3. Set both roles in the task definition, then update the service:
   ```json
   {
     "family": "summariser",
     "executionRoleArn": "arn:aws:iam::111122223333:role/summariser-exec-role",
     "taskRoleArn": "arn:aws:iam::111122223333:role/summariser-task-role",
     "requiresCompatibilities": ["FARGATE"],
     "networkMode": "awsvpc",
     "cpu": "512", "memory": "1024",
     "containerDefinitions": [{
       "name": "api",
       "image": "111122223333.dkr.ecr.us-east-1.amazonaws.com/summariser:1.0",
       "portMappings": [{ "containerPort": 8080 }],
       "logConfiguration": { "logDriver": "awslogs", "options": {
         "awslogs-group": "/ecs/summariser", "awslogs-region": "us-east-1",
         "awslogs-stream-prefix": "api" } }
     }]
   }
   ```
   ```bash
   aws ecs register-task-definition --cli-input-json file://taskdef.json
   aws ecs update-service --cluster genai-cluster --service summariser \
     --task-definition summariser --force-new-deployment
   ```

### Verification
`describe-services` shows `runningCount` equal to `desiredCount`, the CloudWatch log group `/ecs/summariser` shows requests, and a test call returns a summary instead of `AccessDeniedException`.

---

## 3: Textract Rejects a Multi-Page PDF

**Related lab:** Amazon Textract
**Concepts:** synchronous vs asynchronous Textract APIs, S3 input, job polling, pagination, preparing documents for a knowledge base

### Scenario
Scanned supplier invoices are in S3. Before loading them into a Bedrock knowledge base, Textract should extract their text so questions like *"How much did we pay this supplier in March?"* can be answered.

```python
textract = boto3.client("textract")
resp = textract.detect_document_text(
    Document={"S3Object": {"Bucket": "invoices-raw", "Name": "march-invoices.pdf"}})
```

### Symptoms
```
UnsupportedDocumentException: An error occurred (UnsupportedDocumentException)
when calling the DetectDocumentText operation: Request has unsupported document format
```
A single-page JPEG of one invoice works fine.

### Diagnosis
`DetectDocumentText` and `AnalyzeDocument` are **synchronous** — they're for single-page documents. `march-invoices.pdf` has 12 pages. Multi-page PDFs need the **asynchronous** APIs, which read the file from S3 and run as a job.

| Document | API |
|---|---|
| One page (JPEG, PNG, single-page PDF/TIFF) | `DetectDocumentText` / `AnalyzeDocument` — answer straight away |
| Multi-page PDF or TIFF in S3 | `StartDocumentTextDetection` / `StartDocumentAnalysis` → collect results later |

A second, quieter problem: when results do come back, an asynchronous job's output is **split into pages of results**. Code that reads only the first response silently loses most of the text.

### Fix
Start an asynchronous job, wait for it, and follow `NextToken` until every block is collected:

```python
import time, boto3

textract = boto3.client("textract")

job = textract.start_document_text_detection(
    DocumentLocation={"S3Object": {"Bucket": "invoices-raw", "Name": "march-invoices.pdf"}})
job_id = job["JobId"]

while True:
    result = textract.get_document_text_detection(JobId=job_id)
    if result["JobStatus"] in ("SUCCEEDED", "FAILED"):
        break
    time.sleep(5)

lines, token = [], None
while True:
    kwargs = {"JobId": job_id, **({"NextToken": token} if token else {})}
    page = textract.get_document_text_detection(**kwargs)
    lines += [b["Text"] for b in page["Blocks"] if b["BlockType"] == "LINE"]
    token = page.get("NextToken")
    if not token:
        break

print(len(lines), "lines extracted")
```

For tables and form fields (such as *Invoice total*), use `start_document_analysis` with `FeatureTypes=["TABLES", "FORMS"]`. For production, ask Textract to notify an SNS topic when the job finishes instead of polling in a loop.

### Verification
The job reaches `SUCCEEDED`, the line count covers all 12 pages, and the saved text file — uploaded to the knowledge base's S3 data source and synced — lets the knowledge base answer the March question with a citation.

---

## Cleanup (avoid surprise charges)
- **Kinesis Data Streams** bill per shard-hour while the stream exists:
  ```bash
  aws kinesis delete-stream --stream-name chat-events
  ```
- **ECS services on Fargate** bill while tasks run — scale to zero, then delete:
  ```bash
  aws ecs update-service --cluster genai-cluster --service summariser --desired-count 0
  aws ecs delete-service --cluster genai-cluster --service summariser
  aws ecs delete-cluster --cluster genai-cluster
  ```
- ECR images, NAT gateways, CloudWatch log groups
- EC2 instances, EKS clusters, ElastiCache clusters, and CloudFormation / CDK stacks from the other labs (`cdk destroy` or `aws cloudformation delete-stack`)

## Common debugging checklist for data, container and AI services
- **Read the exact error** — it usually names the API, role or format that's wrong.
- **Check the encoding** — Kinesis data is base64 both ways.
- **Check where you're reading from** — `LATEST` vs `TRIM_HORIZON`.
- **Know which role is which** — execution role for ECS, task role for your code.
- **Check the API matches the input** — single page → synchronous, multi-page → asynchronous.
- **Collect every page of results** — follow `NextToken`.
