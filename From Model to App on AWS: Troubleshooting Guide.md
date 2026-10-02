From Model to App on AWS: Troubleshooting Guide

LinkedIn Activity: 
https://www.linkedin.com/posts/krishnansmita_bla-03-part-1-managing-ai-models-with-amazon-activity-7511631410164805632-5TS4?utm_source=share&utm_medium=member_desktop&rcm=ACoAAAL-jvEBG3XA5cCh-Z3noyzpP1Fi4LZ61wk
https://www.linkedin.com/posts/krishnansmita

View Tutorial on YouTube: 
part1: https://youtu.be/tGUo0W8b27A
part2: https://youtu.be/J9fNY33b5Wc
part3:https://youtu.be/tsGB2fA7x5I


Three troubleshooting walkthroughs based on the Section 6 and Section 7 labs of the AWS Certified Generative AI Developer – Professional course (Managing Models with SageMaker AI and More Tools for Building AI Applications).

Scenario-based walkthroughs written from the course labs and AWS documentation. I haven't run these in a live AWS account yet.

1: JumpStart Model Won't Deploy

Related lab: Hands On: SageMaker JumpStart (Section 6, lecture 155) Concepts: SageMaker JumpStart, real-time endpoints, instance types, service quotas, EULAs

Scenario

An open-source foundation model is deployed from SageMaker JumpStart to a real-time endpoint.

python
from sagemaker.jumpstart.model import JumpStartModel

model = JumpStartModel(model_id="meta-textgeneration-llama-3-8b-instruct")
predictor = model.deploy()
Symptoms

First, a message asking you to accept the model's end-user licence agreement. After that's fixed:

ResourceLimitExceeded: The account-level service limit 'ml.g5.2xlarge for endpoint usage'
is 0 Instances, with current utilization of 0 Instances and a request delta of 1 Instances.
Please use AWS Service Quotas to request an increase to this quota.
Diagnosis

Two separate things:

Gated models (such as Llama) need their licence accepted in code before they deploy.
New accounts often start with a quota of 0 for GPU instance types on endpoints. The deploy asked for a GPU instance the account isn't allowed to run yet.

Check the quota:

bash
aws service-quotas list-service-quotas --service-code sagemaker \
  --query "Quotas[?contains(QuotaName, 'ml.g5.2xlarge for endpoint usage')].[QuotaName,Value]"
Fix
Accept the licence explicitly:
python
   predictor = model.deploy(accept_eula=True)
Either request a quota increase in Service Quotas → Amazon SageMaker, or deploy to an instance type the account already has quota for:
python
   model = JumpStartModel(model_id="meta-textgeneration-llama-3-8b-instruct",
                          instance_type="ml.g5.xlarge")
If you only need a quick test, a Bedrock model may avoid hosting an endpoint at all.
Verification

deploy() finishes, the endpoint shows InService, and this returns generated text:

python
predictor.predict({"inputs": "Explain RAG in one sentence."})
2: Step Functions Prompt Chain Fails at Step Two

Related lab: Lab: Prompt Chaining with Step Functions and Bedrock (Section 7, lecture 169) Concepts: Step Functions, Bedrock optimised integration, state machine IAM roles, JSONPath, ResultSelector / ResultPath

Scenario

A state machine chains two Bedrock calls: Summarise a customer email, then Classify the summary as billing, delivery or other.

Symptoms

First run fails at Summarise:

AccessDeniedException: ... is not authorized to perform: bedrock:InvokeModel
on resource: arn:aws:bedrock:us-east-1::foundation-model/amazon.nova-lite-v1:0

After that's fixed, it fails at Classify:

States.Runtime: The JSONPath '$.summary' specified for the field 'text.$'
could not be found in the input
Diagnosis

Open the failed execution in the Step Functions console and select each state to see its input and output.

Access denied: the state machine's role can't call the model.
JSONPath error: Summarise replaced the whole state data with the raw Bedrock response, so there's no $.summary for Classify to read. The text is buried inside the response body, and the original email is gone too.
Fix
Allow the state machine role to call the models:
json
   {
     "Effect": "Allow",
     "Action": "bedrock:InvokeModel",
     "Resource": [
       "arn:aws:bedrock:us-east-1::foundation-model/amazon.nova-lite-v1:0",
       "arn:aws:bedrock:us-east-1::foundation-model/amazon.nova-micro-v1:0"
     ]
   }
On Summarise, pick out just the text and store it beside the original input:
json
   "Summarise": {
     "Type": "Task",
     "Resource": "arn:aws:states:::bedrock:invokeModel",
     "Parameters": {
       "ModelId": "arn:aws:bedrock:us-east-1::foundation-model/amazon.nova-lite-v1:0",
       "Body": {
         "messages": [{ "role": "user", "content": [{
           "text.$": "States.Format('Summarise this email in two sentences: {}', $.email)" }] }],
         "inferenceConfig": { "maxTokens": 200 }
       }
     },
     "ResultSelector": { "text.$": "$.Body.output.message.content[0].text" },
     "ResultPath": "$.summary",
     "Next": "Classify"
   }

Classify then reads $.summary.text. (The exact path inside Body depends on the model's response format.)

Verification

The execution graph shows both states green. Classify's input contains both email and summary.text.

3: CodeBuild Fails Before Running a Single Test

Related labs: AWS CodeBuild – Hands On, Parts 1 and 2 (Section 7, lectures 174–175) · AWS CodePipeline – Hands On (lecture 172) Concepts: CodePipeline, CodeBuild, buildspec.yml, build phases, testing prompt changes

Scenario

A pipeline is set up for a small GenAI app: Source → Build → Deploy. The Build stage should install dependencies and run tests, including checks on prompt and configuration files.

Symptoms

The pipeline stops at Build. The CodeBuild log shows:

Phase context status code: YAML_FILE_ERROR
Message: stat /codebuild/output/src123/src/buildspec.yml: no such file or directory

After that's fixed, the build fails again:

Phase context status code: COMMAND_EXECUTION_ERROR
Message: Error while executing command: pytest. Reason: exit status 127
Diagnosis
YAML_FILE_ERROR — CodeBuild looks for buildspec.yml in the root of the source. Here it was inside a config/ folder (or named buildspec.yaml).
Exit status 127 means "command not found" — pytest was never installed.
Fix
Move the file to the repository root and name it exactly buildspec.yml — or point the project to its real location in Buildspec name.
Install dependencies before running tests:
yaml
   version: 0.2
   phases:
     install:
       runtime-versions:
         python: 3.12
       commands:
         - pip install -r requirements.txt pytest
     build:
       commands:
         - python -m pytest tests -q
         - python -c "import json; json.load(open('config/model-settings.json'))"
   artifacts:
     files:
       - '**/*'

The last command checks the model settings file is valid — a prompt or config change is tested like code.

Verification

The CodeBuild log shows SUCCEEDED for every phase, and the pipeline moves on to Deploy.

Cleanup (avoid surprise charges)
SageMaker endpoints bill every hour they exist, even with no traffic:
python
  predictor.delete_model()
  predictor.delete_endpoint()
Step Functions state machines, Lambda functions and API Gateway APIs
CodePipeline pipelines, CodeBuild projects, the pipeline's S3 artifact bucket, and any EC2 instances from the CodeDeploy lab
SQS queues, SNS topics and EventBridge rules from the messaging labs
Common debugging checklist for models and AI apps
Read the error's exact wording — quota, licence, permission and path errors each point somewhere different.
Check every role — SageMaker execution role, state machine role, CodeBuild service role.
Inspect state input and output in Step Functions before changing the definition.
Check the build log's phase and status code — they tell you which stage failed.
Delete endpoints as soon as you're done.
