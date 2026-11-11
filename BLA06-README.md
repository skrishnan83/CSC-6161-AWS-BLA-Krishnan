# AWSGenAIdeveloper · BLA-06

## Operations, Networking and Storage for GenAI Apps: Troubleshooting Guide

**LinkedIn Activity:** [https://lnkd.in/p/gZkhJ7M8]
**View on YouTube:** [https://www.youtube.com/watch?v=OLY8puXvD9A&t=9s] , [link 2] , [link 3]

Three troubleshooting walkthroughs based on the last *Other Services You Should Know* labs of the AWS Certified Generative AI Developer – Professional course (Management and Governance; Migration and Transfer; Networking and Content Delivery; Storage), plus a short note on the GenAI architecture trade-offs that close the section.

> Scenario-based walkthroughs written from the course labs and AWS documentation.

---

## 1: Session Manager Can't Connect to the Instance

**Related labs:** AWS Systems Manager – Session Manager · Parameter Store
**Concepts:** SSM Agent, instance profiles, `AmazonSSMManagedInstanceCore`, VPC interface endpoints, SecureString parameters

### Scenario
An EC2 instance in a **private subnet** runs a small RAG worker that reads its model ID from Parameter Store. To check its logs, the team wants a shell through Session Manager instead of opening port 22.

### Symptoms
In the console, **Connect → Session Manager** is greyed out, and the instance is missing from Fleet Manager. From the CLI:
```
An error occurred (TargetNotConnected) when calling the StartSession operation:
i-0abc1234def567890 is not connected.
```

### Diagnosis
Session Manager only works when the **SSM Agent** on the instance can register with Systems Manager. Check whether it has:

```bash
aws ssm describe-instance-information \
  --filters "Key=InstanceIds,Values=i-0abc1234def567890" \
  --query "InstanceInformationList[].{id:InstanceId,ping:PingStatus,agent:AgentVersion}"
```
An empty list means the agent has never checked in. There are three usual causes:

| Check | What's wrong |
|---|---|
| Instance profile | No role attached, or the role is missing `AmazonSSMManagedInstanceCore` |
| Network path | Private subnet with no NAT gateway and no VPC endpoints, so the agent can't reach Systems Manager over HTTPS |
| Agent | Not installed or not running (it comes preinstalled on Amazon Linux 2023) |

This instance has no role attached, and its subnet has no route out.

### Fix
1. Create a role for EC2 with the managed policy, and attach it:
   ```bash
   aws iam create-role --role-name rag-worker-ssm-role \
     --assume-role-policy-document file://ec2-trust.json
   aws iam attach-role-policy --role-name rag-worker-ssm-role \
     --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
   aws iam create-instance-profile --instance-profile-name rag-worker-ssm-profile
   aws iam add-role-to-instance-profile --instance-profile-name rag-worker-ssm-profile \
     --role-name rag-worker-ssm-role
   aws ec2 associate-iam-instance-profile --instance-id i-0abc1234def567890 \
     --iam-instance-profile Name=rag-worker-ssm-profile
   ```
   `ec2-trust.json`:
   ```json
   { "Version": "2012-10-17",
     "Statement": [{ "Effect": "Allow",
       "Principal": { "Service": "ec2.amazonaws.com" },
       "Action": "sts:AssumeRole" }] }
   ```
2. Give the private subnet a private path to Systems Manager — three **interface endpoints**. Their security group must allow inbound HTTPS (443) from the instance:
   ```bash
   for svc in ssm ssmmessages ec2messages; do
     aws ec2 create-vpc-endpoint --vpc-id vpc-0123abcd --vpc-endpoint-type Interface \
       --service-name com.amazonaws.us-east-1.$svc \
       --subnet-ids subnet-0priv1234 --security-group-ids sg-0endpoints \
       --private-dns-enabled
   done
   ```
3. If the agent was stopped, restart it (or the instance): `sudo systemctl restart amazon-ssm-agent`
4. To start sessions from your laptop, install the **Session Manager plugin** for the AWS CLI. Without it, the CLI reports `SessionManagerPlugin is not found`.

**Then: the parameter.** The worker's own role also needs to read its settings. A `SecureString` needs both SSM and KMS permissions. Without `kms:Decrypt`, the call fails with `AccessDeniedException`:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow",
      "Action": ["ssm:GetParametersByPath", "ssm:GetParameter"],
      "Resource": "arn:aws:ssm:us-east-1:111122223333:parameter/genai/prod*" },
    { "Effect": "Allow",
      "Action": "kms:Decrypt",
      "Resource": "arn:aws:kms:us-east-1:111122223333:key/EXAMPLE-KEY-ID" }
  ]
}
```
```bash
aws ssm get-parameters-by-path --path /genai/prod --with-decryption
```

### Verification
`describe-instance-information` shows `PingStatus: Online`, and this opens a shell with no inbound ports and no SSH key:
```bash
aws ssm start-session --target i-0abc1234def567890
```
Inside the session, `get-parameters-by-path` returns `/genai/prod/model-id` decrypted.

---

## 2: The Load Balancer Returns 503 — Every Target Is Unhealthy

**Related lab:** Amazon Elastic Load Balancing
**Concepts:** listeners, target groups, health checks, security groups between ALB and targets, idle timeout

### Scenario
Two EC2 instances run a chat API on **port 8080**. An Application Load Balancer listens on 443 and forwards `/chat/*` to a target group called `chat-tg`.

### Symptoms
Every request through the load balancer returns:
```
503 Service Temporarily Unavailable
```
Calling an instance directly on port 8080 (from inside the VPC) works fine.

### Diagnosis
A 503 from an ALB usually means **no healthy targets**. Ask the target group why:

```bash
aws elbv2 describe-target-health --target-group-arn arn:aws:elasticloadbalancing:us-east-1:111122223333:targetgroup/chat-tg/abc123 \
  --query "TargetHealthDescriptions[].{id:Target.Id,state:TargetHealth.State,reason:TargetHealth.Reason,detail:TargetHealth.Description}"
```
```
i-0aaa...  unhealthy  Target.ResponseCodeMismatch  Health checks failed with these codes: [404]
i-0bbb...  unhealthy  Target.Timeout               Request timed out
```

Two different problems:

| Reason | Cause |
|---|---|
| `ResponseCodeMismatch [404]` | The health check uses the default path `/`, but the app only serves `/chat/*` and `/health` |
| `Target.Timeout` | The second instance's security group doesn't allow port 8080 **from the load balancer** |

### Fix
1. Point the health check at a path the app really serves:
   ```bash
   aws elbv2 modify-target-group --target-group-arn <chat-tg-arn> \
     --health-check-path /health --matcher HttpCode=200 \
     --healthy-threshold-count 2 --health-check-interval-seconds 15
   ```
2. Allow traffic from the **ALB's security group** — not from the whole internet:
   ```bash
   aws ec2 authorize-security-group-ingress --group-id sg-0targets \
     --protocol tcp --port 8080 --source-group sg-0alb
   ```
3. **Later, the GenAI-specific problem:** long answers fail with `504 Gateway Timeout` after about 60 seconds. That's the ALB's default idle timeout. Raise it, or better, stream the response so bytes keep flowing:
   ```bash
   aws elbv2 modify-load-balancer-attributes --load-balancer-arn <alb-arn> \
     --attributes Key=idle_timeout.timeout_seconds,Value=180
   ```

| Error | Usually means |
|---|---|
| 503 | No healthy targets in the target group |
| 502 | Target reached, but it closed the connection or sent a bad response |
| 504 | Target took too long — check the idle timeout and app timeouts |

### Verification
`describe-target-health` shows both targets `healthy`, and `curl https://chat.example.com/chat/ping` returns 200 through the load balancer.

---

## 3: EFS Mount Hangs, Then Times Out

**Related lab:** Amazon EFS
**Concepts:** mount targets per Availability Zone, NFS port 2049, security groups, `amazon-efs-utils`, VPC DNS settings

### Scenario
Two GPU instances in different Availability Zones should share one EFS file system that holds downloaded model files and documents. Then each file only has to be downloaded once.

```bash
sudo mkdir -p /mnt/shared
sudo mount -t efs -o tls fs-0123456789abcdef0:/ /mnt/shared
```

### Symptoms
On the first instance, the command hangs for a minute and then fails:
```
mount.nfs4: Connection timed out
```
On the second instance, it fails straight away:
```
Failed to resolve "fs-0123456789abcdef0.efs.us-east-1.amazonaws.com"
```

### Diagnosis
EFS is reached through a **mount target** in each Availability Zone, over **NFS on TCP port 2049**.

```bash
aws efs describe-mount-targets --file-system-id fs-0123456789abcdef0 \
  --query "MountTargets[].{az:AvailabilityZoneName,ip:IpAddress,state:LifeCycleState,id:MountTargetId}"
aws efs describe-mount-target-security-groups --mount-target-id fsmt-0aaa1111
```

| Symptom | Cause |
|---|---|
| Hangs, then `Connection timed out` | The mount target's security group doesn't allow inbound TCP 2049 from the instances |
| `Failed to resolve` | No mount target in the second instance's Availability Zone — or the VPC has DNS hostnames turned off |
| `unknown filesystem type 'efs'` | `amazon-efs-utils` isn't installed |

Here the mount target uses the VPC's default security group, and only `us-east-1a` has a mount target.

### Fix
1. Allow NFS from the instances' security group to the EFS security group:
   ```bash
   aws ec2 authorize-security-group-ingress --group-id sg-0efs \
     --protocol tcp --port 2049 --source-group sg-0gpu-instances
   aws efs modify-mount-target-security-groups --mount-target-id fsmt-0aaa1111 \
     --security-groups sg-0efs
   ```
2. Add a mount target in the second Availability Zone:
   ```bash
   aws efs create-mount-target --file-system-id fs-0123456789abcdef0 \
     --subnet-id subnet-0in-us-east-1b --security-groups sg-0efs
   ```
3. Check that the VPC resolves DNS names, and that the mount helper is installed:
   ```bash
   aws ec2 modify-vpc-attribute --vpc-id vpc-0123abcd --enable-dns-hostnames "{\"Value\":true}"
   sudo dnf install -y amazon-efs-utils
   ```
4. To mount it again after a reboot, add it to `/etc/fstab`:
   ```
   fs-0123456789abcdef0:/  /mnt/shared  efs  _netdev,tls  0  0
   ```

### Verification
`df -h /mnt/shared` shows the file system on both instances. A file written on one (`echo ok | sudo tee /mnt/shared/test.txt`) can be read on the other.

**EFS or EBS?** Use EBS when one instance needs its own fast disk. Use EFS when many instances need the same files.

---

## A Note on Architecture Trade-offs

The section ends with design lectures rather than labs. The decisions I'd write down before building:

| Decision | Question to ask first |
|---|---|
| Add a layer (WAF, cache, Guardrails…) | What real risk does it remove, and is that worth the cost and delay? |
| Vector store | What does the team already run? Do we need hybrid search, the lowest latency, graph relationships, or the lowest cost? |
| Lambda or Step Functions | One quick call, or fixed steps, waits for approval, or jobs over 15 minutes? |
| Internal or user-facing design | Who is on the other end — known employees, or anyone on the internet? |

---

## Cleanup (avoid surprise charges)
- **Load balancers** bill per hour while they exist. Delete the ALB, then its target groups:
  ```bash
  aws elbv2 delete-load-balancer --load-balancer-arn <alb-arn>
  aws elbv2 delete-target-group --target-group-arn <chat-tg-arn>
  ```
- **Auto Scaling groups:** set the desired count to 0 or delete the group, or it will start new instances to replace the ones you terminate.
- **EFS:** delete the mount targets, then the file system.
- **EBS:** terminating an instance doesn't delete *extra* volumes. Also delete old snapshots:
  ```bash
  aws ec2 describe-volumes --filters Name=status,Values=available
  ```
- **VPC interface endpoints** bill per hour, in each AZ.
- **Global Accelerator** has a fixed hourly fee. **Transfer Family** endpoints bill per hour while they're on. **CloudFront** distributions must be disabled before you can delete them. **Route 53** hosted zones cost a small monthly fee.
- **Managed Grafana** workspaces, and DataSync agents and tasks.

## Common debugging checklist for operations, networking and storage
- **Ask the service why.** `describe-instance-information`, `describe-target-health` and `describe-mount-targets` usually name the cause.
- **Check the role.** Instance profile for the SSM Agent, plus `kms:Decrypt` for SecureString parameters.
- **Check the path.** Private subnets need a NAT gateway or VPC endpoints.
- **Check security groups in both directions.** Allow traffic from the *source security group* (ALB → 8080, instances → 2049), not from `0.0.0.0/0`.
- **Check per-AZ pieces.** EFS needs a mount target in every AZ you mount from.
- **Remember GenAI timing.** Long model answers need a longer idle timeout, or streaming.
