---
name: openclaw-prerequisites
description: Verify prerequisites before deploying OpenClaw — AWS account access, Bedrock model enablement, SSM plugin, Telegram bot token, optional Parallel AI key. Use this skill at the start of any setup session, before any AWS resource creation or repo configuration.
---

# OpenClaw Prerequisites

Before deploying OpenClaw, confirm every prerequisite below is in place. Each one has a verification step that returns a definite yes/no — do not skip the verifications. Half-configured prerequisites are the most common source of failed deploys.

## 1. AWS account with admin or scoped IAM permissions

OpenClaw provisions an EC2 instance, an IAM role, an S3 bucket, and uses Bedrock + SSM. The deployer needs permissions for: `cloudformation:*`, `ec2:*`, `iam:CreateRole`, `iam:PutRolePolicy`, `iam:CreateInstanceProfile`, `bedrock:*`, `s3:*` on the audio-staging bucket, `ssm:StartSession`, `ssm:SendCommand`.

**Verify:**
```bash
aws sts get-caller-identity
aws iam list-roles --max-items 1 --query 'Roles[0].RoleName' --output text
```

Both should succeed with no error. Default profile only — never `--profile <name>`.

## 2. Bedrock model access

Claude Sonnet (or whichever model you'll run) must be **explicitly enabled** in the target region. Bedrock denies access by default; enabling is a one-click action in the console but invisible from the CLI without checking.

**Verify:**
```bash
aws bedrock list-foundation-models \
  --region us-east-1 \
  --query 'modelSummaries[?contains(modelId, `claude-sonnet`)].[modelId,modelLifecycle.status]' \
  --output table
```

If the inference profile you want is not listed, go to the Bedrock console → Model access → request access for Anthropic Claude. Approval is usually instant for individual accounts.

**Region matters.** Pick a region where Bedrock has the model you want. `us-east-1` is the safest default. If your region has limited Bedrock availability, change region or live with the available models.

## 3. SSM Session Manager plugin (local machine)

The Makefile pattern uses `aws ssm start-session` for shell access and command execution. The plugin is a separate binary, not part of the AWS CLI.

**Verify:**
```bash
session-manager-plugin --version
```

Should print a version string. If "command not found":

```bash
# macOS (Homebrew)
brew install --cask session-manager-plugin

# macOS (manual, no sudo) — see openclaw-aws-bootstrap skill for full instructions
```

## 4. Telegram bot token

OpenClaw is bot-driven on Telegram. You need a bot token from BotFather before deploy — the gateway needs it to authenticate at startup.

**Get the token:**
1. Message [@BotFather](https://t.me/BotFather) on Telegram
2. `/newbot` → follow the prompts → BotFather returns a token of the form `<digits>:<base64ish>`
3. Save the token securely (it grants full control of the bot)

**Verify the token works:**
```bash
curl -s "https://api.telegram.org/bot<TOKEN>/getMe" | jq .ok
```

Should print `true`. If `false`, token is wrong or the bot was deleted.

## 5. (Optional) Parallel AI API key

Required only if the deploy will include the deep-research pipeline. Skip this prerequisite if you don't want research capabilities.

**Verify:**
```bash
curl -s -H "x-api-key: $PARALLEL_API_KEY" https://api.parallel.ai/v1/health
```

Should return a healthy response.

## 6. (Optional) GitHub personal access token

Required only if the deploy will publish research output to a GitHub repo or file issues from chat. Token needs `repo` scope on the target repo(s).

**Verify:**
```bash
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" https://api.github.com/user | jq .login
```

Should print the GitHub username.

---

## Decision: which prerequisites are mandatory for THIS deploy?

Before proceeding, confirm with the user which capabilities they want:

| Capability | Requires | Decision |
|---|---|---|
| Core agent (chat, tools, memory) | 1, 2, 3, 4 | mandatory |
| Voice memo transcription | + AWS Transcribe (auto-enabled) | optional |
| Deep research | + 5 + 6 | optional |
| GitHub issue filing from chat | + 6 | optional |

A minimal deploy needs only 1–4. Skip anything optional that the user doesn't want — every capability adds setup steps.

---

## Output of this skill

When done, the agent should have a confirmed list of:

- AWS account ID and the default region
- Confirmed Bedrock model ID and access status
- Confirmed working SSM plugin version
- Telegram bot token saved (location to be decided in `openclaw-channel-setup`)
- Decisions about optional capabilities

If any prerequisite fails verification, **stop here.** Do not proceed to bootstrap. Get the prerequisite working first, then resume.
