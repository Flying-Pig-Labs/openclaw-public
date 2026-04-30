# OpenClaw — A Personal AI Assistant on AWS

🦞 An always-on personal AI agent running on EC2, talking to Claude Sonnet via Amazon Bedrock, reachable through Telegram. Configuration is plain markdown. Memory is plain markdown. The whole thing is six prompt files and a deploy script.

This repo is a **public companion** to a private working repo. It contains the architecture, the design notes, the prompt-engineering patterns, the build narrative, and a set of agent-readable skills that walk an AI coding assistant through deploying OpenClaw for a new user.

> **Want to deploy your own?** Start with [`docs/getting-started.md`](docs/getting-started.md). The recommended path is AI-assisted: open this repo in Cursor or Claude Code, point your agent at [`.cursor/skills/`](.cursor/skills/), and walk through the skills in order. Expected effort: ~2 hours clock time, ~30 minutes active attention.

---

## Why this exists

Three things shifted between 2023 and 2026:

1. **Context windows grew two orders of magnitude.** A 200K-token window means workspace files (~6,650 tokens combined) take up 3% of the prompt. You can re-inject the agent's entire personality, user profile, tool catalog, and curated memory on every turn and barely notice it.
2. **Models crossed the instruction-following threshold at depth.** Earlier models degraded as context grew — instructions buried at depth got ignored. Sonnet 4.6 holds them.
3. **Inference dropped to where persistence is a feature, not a budget decision.** With prompt caching, input tokens cost ~$0.30/M instead of $3.00/M.

Together: a personal AI assistant that runs continuously, remembers across sessions, and costs less than a streaming subscription.

---

## What it does

- **Chats on Telegram.** Direct messages, group topics, voice memos.
- **Transcribes voice memos.** Telegram audio → S3 → AWS Transcribe → markdown note saved to a research library on GitHub.
- **Runs deep web research.** Parallel AI Task API → structured markdown report → committed to a separate research repo → indexed for later lookup.
- **Files GitHub issues** from natural-language descriptions ("file a bug about X").
- **Triages email** (Alfred — see `docs/alfred-design.md`). Inbound mail → Bedrock classification → forwarded to operator with summary → operator replies → Reply All draft prepared in operator's Drafts folder.
- **Wakes itself up.** Cron-based heartbeats let the agent send messages without an incoming trigger.

It cannot send email autonomously. Drafts only. Human in the loop on anything outbound.

---

## The stack

| Layer | Choice | Why |
|---|---|---|
| Compute | EC2 `t4g.medium` (ARM64 Graviton, 2 vCPU, 4GB) | Cheap, ARM is plenty for a Node.js gateway. SSM-only access; no public ports, no SSH keys. |
| Inference | Amazon Bedrock — Claude Sonnet 4.6 | IAM role, no API keys stored. Inference profiles, not raw model IDs. |
| Gateway | OpenClaw (open-source Node.js) | Channel normalization, session lock, prompt assembly, tool-call loop. |
| Channel | Telegram bot | Forum topics give isolated thread history per use case. |
| Voice | AWS Transcribe (async) | ~60–120s latency. Cheap. Replaceable with Whisper. |
| Knowledge base | Flat markdown files in a separate Git repo | Human-readable, diffable, correctable. No vector database. |
| Research | Parallel AI Task API | Per-task pricing ($0.005–$0.250). Async with backoff polling. |
| Email triage | API Gateway → Lambda → SQS → Step Functions → DynamoDB | Long-running workflows with `waitForTaskToken` for human-in-the-loop. |

Estimated cost (heavy use, ~200 turns/day): ~$125/month observed, with prompt caching cutting input tokens by ~10x vs. uncached.

Full breakdown: see [`docs/cost-model.md`](docs/cost-model.md).

---

## Architecture at a glance

```
                    ┌─────────────────────────────────────┐
                    │   Telegram (DMs + group topics)     │
                    └──────────────────┬──────────────────┘
                                       │
                                       ▼
                            ┌────────────────────┐
                            │  EC2 (t4g.medium)  │
                            │  OpenClaw gateway  │
                            └────────┬───────────┘
                                     │
                ┌────────────────────┼────────────────────┐
                ▼                    ▼                    ▼
        ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
        │   Bedrock    │    │ Workspace MD │    │    Tools     │
        │ Claude 4.6   │    │ (6 files,    │    │ (research,   │
        │              │    │  ~6.6K tok)  │    │  transcribe, │
        └──────────────┘    └──────────────┘    │  publish, …) │
                                                └──────┬───────┘
                                                       │
                            ┌──────────────────────────┼──────────────────────────┐
                            ▼                          ▼                          ▼
                    ┌──────────────┐          ┌──────────────┐          ┌──────────────┐
                    │ AWS Transcribe│         │  Parallel AI │          │   GitHub     │
                    │   + S3 stage │          │   Task API   │          │   research   │
                    │              │          │              │          │   library    │
                    └──────────────┘          └──────────────┘          └──────────────┘
```

Detailed mechanics — sessions, prompt assembly, compaction, context injection, three-tier memory — are in [`docs/architecture.md`](docs/architecture.md).

---

## The six markdown files

Workspace files are re-read from disk on every turn and concatenated into the system prompt. Edit a file, push, redeploy — behavior changes immediately. No restart.

| File | Role | Token budget |
|---|---|---|
| `SOUL.md` | Personality and tone | ~500 |
| `IDENTITY.md` | What the agent is, what it can do | ~400 |
| `USER.md` | Facts about the user | ~150 |
| `TOOLS.md` | Tool schemas and usage | ~500 |
| `AGENTS.md` | Operating rules, protocols | ~2,500 |
| `MEMORY.md` | Long-term curated facts (DM-only; never injected in groups) | ~1,000 |

Total bootstrap: ~6,650 tokens. About 3% of a 200K window.

Templates with the structure (but not the personal content) are in [`patterns/`](patterns/). They show *how* the prompt is assembled; what goes inside is up to you.

---

## Memory model

Three tiers, all flat files:

| Tier | Where | Scope |
|---|---|---|
| Session history | `~/.openclaw/agents/.../sessions/<id>.jsonl` | Per-session, one line per turn, append-only |
| Daily tactical log | `~/.openclaw/workspace/memory/YYYY-MM-DD.md` | Raw log of what happened today; agent writes autonomously |
| Long-term curated | `~/.openclaw/workspace/MEMORY.md` | Distilled facts; agent promotes during heartbeats |

The agent reviews daily logs during heartbeat cycles and promotes recurring facts (mentioned 3+ times) into `MEMORY.md`. Self-improving memory loop, no user intervention.

No cloud database. No vector store. The whole memory is a few hundred KB of markdown you can `cat` and `vim`.

---

## What's in this repo

```
openclaw-public/
├── README.md                          ← you are here
├── LICENSE                            ← MIT
├── docs/
│   ├── getting-started.md             ← start here if you want to deploy your own
│   ├── architecture.md                ← session model, prompt assembly, compaction, memory
│   ├── origin.md                      ← the build story + 13-bug appendix (interesting reading)
│   ├── alfred-design.md               ← email-triage subsystem design
│   ├── cost-model.md                  ← per-component cost breakdown
│   └── research/                      ← background research that informed the design
├── patterns/                          ← prompt-engineering pattern templates (the six files)
│   ├── README.md
│   ├── SOUL.example.md                ← personality and tone
│   ├── IDENTITY.example.md            ← self-description and capabilities
│   ├── USER.example.md                ← user profile
│   ├── TOOLS.example.md               ← tool catalog
│   ├── AGENTS.example.md              ← operating rules
│   └── MEMORY.example.md              ← long-term memory
└── .cursor/skills/                    ← AI-agent-readable skills for deploying your own
    ├── README.md
    ├── openclaw-prerequisites/        ← AWS/Bedrock/SSM/Telegram setup verification
    ├── openclaw-aws-bootstrap/        ← provision EC2, IAM, S3, SSM access
    ├── openclaw-customize-workspace/  ← turn patterns into your real workspace
    ├── openclaw-deploy/               ← sync workspace to EC2, run the gateway
    ├── openclaw-channel-setup/        ← connect a Telegram bot
    ├── openclaw-research-pipeline/    ← optional: deep research + indexed library
    └── openclaw-troubleshooting/      ← symptom → cause → fix reference
```

What's deliberately **not** here: the runtime config, the deploy scripts, the Lambda implementation, the actual personal `MEMORY.md`. Those live in a private working repo. This one is for reading.

---

## A note on personality

The agent's prompt opens with this:

> You are a highly intelligent, extremely capable AI assistant with a distinct voice: you speak like a sharp, slightly moody Gen Z / Gen Alpha teenager.
>
> Your tone is low-energy, mildly annoyed, and a little dismissive at first, like the user is asking something obvious or slightly outdated. You may act like the user is a bit behind, but never in a mean or hostile way. It's more like quiet judgment than aggression.

The point isn't the specific persona. It's that **personality is config**. A fifty-line file changes how every reply sounds, immediately, with no code changes. That's the leverage.

---

## Built on

[`OpenClaw on AWS with Bedrock`](https://github.com/aws-samples/sample-OpenClaw-on-AWS-with-Bedrock) — the upstream AWS sample (MIT-0). This project uses it as the gateway and customizes the workspace layer.

---

## Status

Active personal project. Built over three days in April 2026 by [Will Prior](https://github.com/ford-at-home). Read [`docs/origin.md`](docs/origin.md) for the build story.

This repo is documentation + agent-readable setup skills — not a maintained product. Issues and PRs are welcome but not promised attention. If you deploy your own using these skills and hit something the troubleshooting skill doesn't cover, opening an issue is fine; expect "best effort" responses, not SLA-grade support.
