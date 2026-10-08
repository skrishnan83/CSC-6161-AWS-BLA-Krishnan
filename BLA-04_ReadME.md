# AWSGenAIdeveloper · BLA-04

## Monitoring and Securing GenAI on AWS: Troubleshooting Guide

**LinkedIn Activity:** [link 1] , [link 2] , [link 3]
**View Tutorial on YouTube:** [link 1] , [link 2] , [link 3]

Three troubleshooting walkthroughs based on the labs of the AWS Certified Generative AI Developer – Professional course (*Governance and QA* and *Security, Identity, and Compliance*).

> Scenario-based walkthroughs written from the course labs and AWS documentation.

---

## 1: CloudWatch Alarm for Bedrock Never Goes Off

**Related labs:** CloudWatch Alarms – Hands On · CloudWatch and GenAI Monitoring 
**Concepts:** CloudWatch metrics, namespaces and dimensions, alarm periods, missing data, SNS notifications

### Scenario
An alarm is created to email the team when Bedrock requests are throttled. During a busy afternoon, users report errors — but no email arrives.

### Symptoms
The alarm sits in **`INSUFFICIENT_DATA`** and never moves to `ALARM`, even while throttling is happening.

### Diagnosis
An alarm only works if it's watching a metric that **actually exists, with exactly the same name and dimensions**. List what Bedrock is really publishing:

```bash
aws cloudwatch list-metrics --namespace AWS/Bedrock --metric-name InvocationThrottles
```

Compare it with the alarm:

```bash
aws cloudwatch describe-alarms --alarm-names bedrock-throttles \
  --query "MetricAlarms[0].{ns:Namespace,metric:MetricName,dims:Dimensions,state:StateValue}"
```

Three common mismatches:

| What's wrong | Example |
|---|---|
| Wrong dimension value | Alarm uses the model *name*; metrics are published per `ModelId` |
| Inference profile used | The app calls `us.amazon.nova-lite-v1:0`, but the alarm watches `amazon.nova-lite-v1:0` |
| Missing data treated as "unknown" | No throttles for a while → no data points → `INSUFFICIENT_DATA` |

### Fix
Recreate the alarm on the exact `ModelId` the app calls, treat missing data as "not breaching", and send it to an SNS topic:

```bash
aws sns create-topic --name genai-alerts
aws sns subscribe --topic-arn arn:aws:sns:us-east-1:111122223333:genai-alerts \
  --protocol email --notification-endpoint team@example.com

aws cloudwatch put-metric-alarm \
  --alarm-name bedrock-throttles \
  --namespace AWS/Bedrock --metric-name InvocationThrottles \
  --dimensions Name=ModelId,Value=us.amazon.nova-lite-v1:0 \
  --statistic Sum --period 300 --evaluation-periods 1 \
  --threshold 10 --comparison-operator GreaterThanThreshold \
  --treat-missing-data notBreaching \
  --alarm-actions arn:aws:sns:us-east-1:111122223333:genai-alerts
```

Remember to **confirm the SNS subscription** from the email — until then, nothing is delivered.

### Verification
Force a test state change:

```bash
aws cloudwatch set-alarm-state --alarm-name bedrock-throttles \
  --state-value ALARM --state-reason "Testing notification"
```

The team receives the email, and the alarm then returns to `OK` on its own.

---

## 2: CloudTrail Shows the Model Was Called — but Not What Was Asked

**Related labs:** AWS CloudTrail – Hands On · CloudTrail and GenAI · Clarification on CloudTrail + Bedrock
**Concepts:** CloudTrail events, Bedrock model invocation logging, CloudWatch Logs, access to sensitive logs

### Scenario
A customer complains that the assistant gave them a rude reply yesterday. The team opens CloudTrail to find the conversation.

### Symptoms
CloudTrail shows the call was made — by which role, at what time, to which model — but **there's no prompt and no reply** anywhere in the event.

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventSource,AttributeValue=bedrock.amazonaws.com \
  --max-results 5
```

### Diagnosis
This is CloudTrail working as designed. It records **who called which API, and when** — the audit trail. It does **not** record the content of prompts and responses.

To see the actual words, **Bedrock model invocation logging** has to be switched on. It's off by default, and it only captures calls made *after* it's enabled — so yesterday's conversation can't be recovered.

Check whether it's on:

```bash
aws bedrock get-model-invocation-logging-configuration
```

An empty result means nothing is being logged.

### Fix
1. Create a log group and a role that Bedrock can use to write to it.
2. Turn on invocation logging:

```bash
aws bedrock put-model-invocation-logging-configuration --logging-config '{
  "cloudWatchConfig": {
    "logGroupName": "/bedrock/model-invocations",
    "roleArn": "arn:aws:iam::111122223333:role/BedrockLoggingRole"
  },
  "textDataDeliveryEnabled": true,
  "imageDataDeliveryEnabled": false,
  "embeddingDataDeliveryEnabled": false
}'
```

3. **Restrict who can read the log group.** It now contains everything users typed, which may include personal information. Give read access only to the people who handle complaints and investigations.

### Verification
Make one test call, then search the new log group:

```
fields @timestamp, modelId, input.inputBodyJson.messages.0.content.0.text
| sort @timestamp desc
| limit 5
```

The prompt and the response both appear. CloudTrail still shows who made the call — the two together give the full picture.

---

## 3: Lambda Can't Read Its API Key from Secrets Manager

**Related labs:** AWS Secrets Manager – Hands On · AWS KMS – Hands On · IAM Roles – Hands On
**Concepts:** Secrets Manager, KMS customer managed keys, key policies, IAM roles, least privilege

### Scenario
A GenAI app calls a third-party translation API. Following best practice, the API key is stored in **Secrets Manager** — encrypted with a **customer managed KMS key** — instead of being written in the code. The Lambda function now fails every time.

```python
import boto3, json

secrets = boto3.client("secretsmanager")

def handler(event, context):
    secret = secrets.get_secret_value(SecretId="prod/translation-api-key")
    api_key = json.loads(secret["SecretString"])["api_key"]
    ...
```

### Symptoms
First:
```
AccessDeniedException: User: arn:aws:sts::111122223333:assumed-role/translate-fn-role/translate-fn
is not authorized to perform: secretsmanager:GetSecretValue
```
After that's fixed:
```
AccessDeniedException: Access to KMS is not allowed
```

### Diagnosis
Reading an encrypted secret needs **two separate permissions**, and these two errors match them:

| Error | Missing permission |
|---|---|
| `not authorized to perform: secretsmanager:GetSecretValue` | The Lambda role can't read the secret |
| `Access to KMS is not allowed` | The role can read the secret but can't **decrypt** it with the KMS key |

Because the key is *customer managed*, its **key policy** must also allow the role to use it — an IAM permission alone isn't enough if the key policy doesn't allow it.

```bash
aws kms get-key-policy --key-id KEY_ID --policy-name default
```

### Fix
1. Allow the Lambda role to read **only this one secret** (least privilege):
   ```json
   {
     "Effect": "Allow",
     "Action": "secretsmanager:GetSecretValue",
     "Resource": "arn:aws:secretsmanager:us-east-1:111122223333:secret:prod/translation-api-key-*"
   }
   ```
2. Allow it to decrypt with the key:
   ```json
   {
     "Effect": "Allow",
     "Action": "kms:Decrypt",
     "Resource": "arn:aws:kms:us-east-1:111122223333:key/KEY_ID"
   }
   ```
3. Add the role to the **key policy**:
   ```json
   {
     "Sid": "AllowTranslateFunctionToDecrypt",
     "Effect": "Allow",
     "Principal": { "AWS": "arn:aws:iam::111122223333:role/translate-fn-role" },
     "Action": "kms:Decrypt",
     "Resource": "*"
   }
   ```
4. Cache the secret in the function between invocations, instead of fetching it on every request.

### Verification
The function runs successfully. In CloudTrail, each run shows a `GetSecretValue` event **and** a KMS `Decrypt` event — a record of exactly who used the key, and when.

---

## Cleanup (avoid surprise charges)
- **Secrets Manager** secrets are charged per secret per month — delete test secrets:
  ```bash
  aws secretsmanager delete-secret --secret-id prod/translation-api-key --recovery-window-in-days 7
  ```
- **KMS customer managed keys** are charged per key per month — schedule deletion:
  ```bash
  aws kms schedule-key-deletion --key-id KEY_ID --pending-window-in-days 7
  ```
- CloudWatch alarms, log groups and SNS topics
- CloudTrail trails delivering to S3, and their S3 buckets
- X-Ray sampling rules and IAM test users created in the labs

## Common debugging checklist for monitoring and security
- **Match names exactly** — metric names, dimensions and model IDs must match what's really published.
- **Pick the right record** — CloudTrail for *who*, invocation logging for *what was said*, X-Ray for *where it's slow*.
- **Read "access denied" carefully** — it names the action that's missing.
- **Encrypted resources need two permissions** — the service action and `kms:Decrypt`, plus the key policy.
- **Grant the smallest scope that works** — one secret, one key, one bucket.
- **Protect your logs** — prompt logs can contain personal information.
