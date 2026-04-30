# Getting Started

If you want to deploy your own OpenClaw — read this first.

## Two paths

### Path 1: AI-assisted setup (recommended)

This repo includes [`.cursor/skills/`](../.cursor/skills/) — guidance files for an AI coding agent (Cursor, Claude Code, etc.) helping a human deploy OpenClaw.

The flow:

1. Clone this repo locally
2. Open it in Cursor (or your AI coding agent of choice)
3. Tell the agent: *"Help me deploy my own OpenClaw using the skills in `.cursor/skills/`. Start with `openclaw-prerequisites`."*
4. The agent reads the skill, walks you through the prerequisites, then proceeds to bootstrap, customize, deploy, and connect a Telegram bot

The agent handles the variability — your AWS account state, your OAuth flow, your specific Telegram setup. You handle the decisions: which region, which model, what the persona should sound like.

**Expected effort:** ~2 hours clock time, ~30 minutes active attention.

### Path 2: Manual setup

If you'd rather not use an AI agent, the skills are still readable as documentation. Work through them in order:

1. [`prerequisites`](../.cursor/skills/openclaw-prerequisites/SKILL.md) — AWS, Bedrock, SSM, Telegram
2. [`aws-bootstrap`](../.cursor/skills/openclaw-aws-bootstrap/SKILL.md) — provision the infrastructure
3. [`customize-workspace`](../.cursor/skills/openclaw-customize-workspace/SKILL.md) — fill in the prompt files
4. [`deploy`](../.cursor/skills/openclaw-deploy/SKILL.md) — push to EC2
5. [`channel-setup`](../.cursor/skills/openclaw-channel-setup/SKILL.md) — connect Telegram
6. [`research-pipeline`](../.cursor/skills/openclaw-research-pipeline/SKILL.md) — *(optional)* deep research
7. [`troubleshooting`](../.cursor/skills/openclaw-troubleshooting/SKILL.md) — when things go wrong

**Expected effort:** ~3–5 hours clock time. More if AWS/Bedrock setup is unfamiliar.

## Prerequisites at a glance

You'll need:

- An AWS account with permissions to create EC2, IAM, S3, SSM resources
- Bedrock model access enabled in your target region (one-click in the console)
- The SSM Session Manager plugin installed locally
- A Telegram bot token from [@BotFather](https://t.me/BotFather)

Optional, only if you want research capabilities:

- A Parallel AI API key
- A GitHub personal access token with `repo` scope
- A separate GitHub repo to publish research into

## Cost expectations

| Usage | Approx monthly cost |
|---|---|
| Light (~10 turns/day) | ~$40 |
| Moderate (~50 turns/day) | ~$80 |
| Heavy (~200 turns/day) | ~$125–$235 |

EC2 (`t4g.medium`) is the fixed floor at ~$27/month. Bedrock token usage is the dominant variable cost. Prompt caching (which OpenClaw gets for free thanks to stable workspace files) cuts Bedrock costs by ~70% in practice.

Full breakdown: [`cost-model.md`](cost-model.md).

## What you'll end up with

A personal AI assistant on AWS:

- Reachable from Telegram (DMs and group topics)
- With a personality and identity that you control via six markdown files
- That remembers across sessions in plain markdown files you can read and edit
- With cross-session memory, voice memo transcription, and (optionally) deep research
- Costing $40–$235/month depending on use
- That you fully own — no third-party SaaS, no vendor lock-in

You'll also have a private working repo (your equivalent of the original author's `rvaopenclaw`) where the actual config lives. **This public repo is documentation, not your runtime config.** Your `MEMORY.md` and `USER.md` belong in your own private repo, never published.

## What you won't get

- A one-click installer. There isn't one — and intentionally so. See [`patterns/README.md`](../patterns/README.md) and [`.cursor/skills/README.md`](../.cursor/skills/README.md) for the reasoning.
- Multi-tenancy. OpenClaw is single-user by design.
- A web UI for management. Configuration is done by editing markdown and running `make deploy`.
- Support. This is a personal project, not a maintained product. Issues and PRs are welcome but not promised attention.

## Architecture before you start

If you haven't yet read the architecture docs, do that before you begin a deploy. Understanding how the pieces fit together makes the skill-driven flow much smoother:

- [`README.md`](../README.md) — the overview
- [`docs/architecture.md`](architecture.md) — sessions, prompt assembly, memory model
- [`docs/origin.md`](origin.md) — how the original was built (especially the bug appendix — it'll save you time)

## Privacy and safety

The customized workspace files (`SOUL.md`, `USER.md`, `IDENTITY.md`, `MEMORY.md`, etc.) **contain personal information by definition.** They go in a private working repo, not a fork of this public one. The `customize-workspace` skill explains the structure.

If you publish your config publicly, you'll be publishing whatever's in `MEMORY.md` and `USER.md`. Don't.
