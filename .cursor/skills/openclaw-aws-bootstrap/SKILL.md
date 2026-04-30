---
name: openclaw-aws-bootstrap
description: Bootstrap the AWS infrastructure for an OpenClaw deployment — region selection, instance type sizing, Bedrock model selection, IAM role, S3 bucket, SSM access. Use after openclaw-prerequisites passes and before openclaw-deploy.
---

# AWS Bootstrap for OpenClaw

This skill walks the agent through the AWS-side setup: provisioning the EC2 instance, the IAM role, the S3 staging bucket, and confirming SSM access works. It assumes the prerequisites skill has already verified Bedrock model access and the SSM plugin.

## 1. Region selection

Default: `us-east-1`. Change only if you have a specific reason (data residency, latency, regional Bedrock availability).

**Tradeoffs:**

| Region | Bedrock model availability | EC2 cost | Notes |
|---|---|---|---|
| `us-east-1` | Most models, earliest access | Lowest | Safe default |
| `us-west-2` | Most models | Slightly higher | Good alternative |
| `eu-west-1` | Subset of models | Higher | EU data residency |
| Other | Verify per-model | Varies | Check `aws bedrock list-foundation-models` first |

Pick a region and use it consistently. The skill examples below assume `us-east-1`.

## 2. Instance type

| Type | vCPU / RAM | Cost (us-east-1, on-demand) | Use case |
|---|---|---|---|
| `t4g.small` | 2 / 2 GB | ~$13.50/mo | Tight on RAM if running local Whisper |
| `t4g.medium` | 2 / 4 GB | ~$26.93/mo | **Recommended default** |
| `t4g.large` | 2 / 8 GB | ~$53.86/mo | Heavy local processing |
| `m7g.medium` | 1 / 4 GB | ~$36/mo | Burstable workloads |

ARM (Graviton) is cheaper and plenty fast for a Node.js gateway. Stick with `t4g.medium` unless you have a reason.

## 3. CloudFormation bootstrap

The upstream AWS sample (`aws-samples/sample-OpenClaw-on-AWS-with-Bedrock`) provides a working CloudFormation template. The pattern below uses it as the base.

**Get the template:**
```bash
git clone --depth 1 \
  https://github.com/aws-samples/sample-OpenClaw-on-AWS-with-Bedrock.git \
  /tmp/openclaw-sample
```

**Deploy the stack:**
```bash
aws cloudformation deploy \
  --template-file /tmp/openclaw-sample/clawdbot-bedrock.yaml \
  --stack-name openclaw-bedrock \
  --capabilities CAPABILITY_IAM \
  --region us-east-1 \
  --parameter-overrides \
    InstanceType=t4g.medium \
    CreateVPCEndpoints=false
```

Takes ~8 minutes. `CreateVPCEndpoints=false` saves ~$22/month and is the right default for a single-instance personal deploy.

**Verify:**
```bash
aws cloudformation describe-stacks \
  --stack-name openclaw-bedrock \
  --region us-east-1 \
  --query 'Stacks[0].Outputs' \
  --output table
```

Capture these output values — the deploy skill will need them:

- `InstanceId` — the EC2 instance ID
- `Step3AccessURL` — contains a one-time token used for the gateway UI
- `BucketName` (if the template provisions one) — the audio staging bucket

## 4. IAM role for Bedrock

The CloudFormation template should attach an IAM role to the instance with Bedrock invocation permissions. Verify:

```bash
INSTANCE_ID=<from outputs>
ROLE_NAME=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].IamInstanceProfile.Arn' \
  --output text | awk -F'/' '{print $NF}')

aws iam list-attached-role-policies --role-name "$ROLE_NAME"
```

Should show a policy attaching `bedrock:InvokeModel` (or similar). If the upstream template doesn't include Bedrock permissions, add a policy:

```bash
aws iam put-role-policy \
  --role-name "$ROLE_NAME" \
  --policy-name openclaw-bedrock-invoke \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": ["bedrock:InvokeModel", "bedrock:InvokeModelWithResponseStream"],
      "Resource": "*"
    }]
  }'
```

## 5. S3 audio staging bucket (if voice memos are enabled)

The voice memo pipeline needs an S3 bucket with a 1-day lifecycle. If the template didn't create one:

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET="openclaw-audio-staging-${ACCOUNT_ID}"

aws s3api create-bucket --bucket "$BUCKET" --region us-east-1
aws s3api put-bucket-lifecycle-configuration \
  --bucket "$BUCKET" \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "expire-1day",
      "Status": "Enabled",
      "Filter": {"Prefix": ""},
      "Expiration": {"Days": 1}
    }]
  }'
```

Add `s3:PutObject` and `s3:GetObject` for `arn:aws:s3:::${BUCKET}/*` and `transcribe:StartTranscriptionJob` + `transcribe:GetTranscriptionJob` to the instance role.

## 6. Verify SSM access works

```bash
aws ssm start-session --target "$INSTANCE_ID" --region us-east-1
```

Should drop into a shell on the instance. Type `exit` to leave.

If it fails with "TargetNotConnected":
- Wait 2 minutes and retry — the SSM agent may not have registered yet
- Verify the instance has an IAM role with `AmazonSSMManagedInstanceCore` attached
- Verify the instance is in a subnet with internet access (for SSM agent → SSM endpoint communication)

## 7. Stash the outputs

Save the captured values somewhere the user can reference them later. Suggested location: a gitignored `.env` in their working directory (NOT the public repo):

```bash
cat > ~/.openclaw-deploy.env <<EOF
AWS_REGION=us-east-1
STACK_NAME=openclaw-bedrock
INSTANCE_ID=<id>
AUDIO_BUCKET=<bucket>
EOF
```

Subsequent skills assume these values are available.

---

## Output of this skill

- A running EC2 instance, accessible via SSM
- An IAM role with Bedrock + S3 + Transcribe permissions
- An S3 audio staging bucket with 1-day lifecycle (if voice memos are enabled)
- A `.env` (or equivalent) capturing instance ID, region, bucket name

The instance is provisioned but **not yet configured** — there's no OpenClaw gateway running, no workspace files, no Telegram connection. Those come in `openclaw-deploy` and `openclaw-channel-setup`.
