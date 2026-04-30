# Agent Skills

Guidance files for an AI coding agent helping a user set up their own OpenClaw deployment. Each skill is a single `SKILL.md` with frontmatter (`name`, `description`) and concrete steps the agent can follow.

These skills are designed for Cursor / Claude / any AI coding agent that loads `.cursor/skills/*/SKILL.md` files into its context. **They do not run by themselves.** They guide an agent through a setup conversation with a human user.

## Skill order

The skills are designed to be used in this order:

1. **`openclaw-prerequisites`** — Verify everything that needs to exist before any AWS resources are touched
2. **`openclaw-aws-bootstrap`** — Provision the EC2 instance, IAM role, S3 bucket, SSM access
3. **`openclaw-customize-workspace`** — Turn the `patterns/*.example.md` files into a real personal workspace
4. **`openclaw-deploy`** — Sync the workspace onto the EC2 instance and start the gateway
5. **`openclaw-channel-setup`** — Connect a Telegram bot to the running gateway
6. **`openclaw-research-pipeline`** — *(optional)* Add the Parallel AI research pipeline + indexed library
7. **`openclaw-troubleshooting`** — Reference for when things break (use throughout, not just at the end)

A complete first-time setup, with an AI agent helping, takes ~2 hours of clock time and probably ~30 minutes of active human attention. Most of the time is waiting on CloudFormation, OAuth flows, and DNS-style propagation.

## What's not here

- Bootstrap scripts. There's no `setup.sh`. The skills tell the agent what to do; the agent does it interactively, asking the user for inputs and confirmations as needed.
- A generic Makefile. The original author's Makefile is fused with their specific environment. The deploy skill explains the pattern; the user (with the agent's help) writes their own.
- Tested CloudFormation templates beyond what `aws-samples/sample-OpenClaw-on-AWS-with-Bedrock` provides. The architecture doc explains what AWS resources you need; agents adapt the upstream sample.

## Why agent-facing instead of script-based

A `setup.sh` that fails on step 6 with a cryptic error is worse than no `setup.sh` at all. The user has no way to recover, and the repo author becomes responsible for support.

An AI agent walking through documented skills handles the variability — different AWS account states, different OAuth flows, different Telegram setups, different things-that-go-wrong — and asks the user for clarification when needed. The repo author isn't on the hook for every deployment's edge cases; the user's own AI agent is.

## Format

Each skill follows this shape:

```markdown
---
name: skill-name
description: One sentence explaining when to use this skill — the description is what the agent reads to decide whether to invoke it.
---

# Skill Title

## Step 1
...

## Step 2
...

## Output of this skill
...

## Common pitfalls
...
```

The frontmatter `description` is critical — it's how the agent decides which skill applies to a given moment.
