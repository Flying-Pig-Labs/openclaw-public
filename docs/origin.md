---
title: "Building a Personal AI Assistant on AWS"
date: 2026-04-14
author: Will Prior
status: build narrative
---

# Building a Personal AI Assistant on AWS

The story of how OpenClaw came together over three days in April 2026 — the architecture decisions, the cost model, the emergent patterns, and the bugs that taught the most.

---

## What changed

Three things converged between 2023 and 2026:

1. **Context windows grew by two orders of magnitude.** Claude Sonnet 4.6 has a 200,000-token context window. Workspace files are ~6,650 tokens — 3% of that window. Re-injecting personality, profile, tools, and curated memory on every turn is suddenly free.
2. **Models crossed the instruction-following threshold at depth.** Earlier models degraded as context grew — instructions buried at depth got progressively ignored. Sonnet 4.6 holds them through the entire window.
3. **Inference costs dropped** to where persistence is a feature, not a budget decision. Prompt caching brings input tokens to $0.30/M instead of $3.00/M.

---

## The stack

- **OpenClaw** — open-source Node.js gateway
- **AWS EC2** — t4g.medium, ARM64/Graviton, no public ports, SSM access only
- **Amazon Bedrock** — Claude Sonnet 4.6 via IAM role, no credentials stored
- **Telegram** — bot-native, forum topics give isolated thread history per use case

Six markdown files injected into every prompt: `SOUL.md`, `USER.md`, `AGENTS.md`, `TOOLS.md`, `MEMORY.md`, `IDENTITY.md`. ~6,650 tokens combined. To change behavior: edit a file, push, done. No restart required.

---

## Cost

| Service | Light (~10 turns/day) | Moderate (~50/day) | Heavy (~200/day) |
|---|---|---|---|
| EC2 t4g.medium | $26.93 | $26.93 | $26.93 |
| Claude Sonnet 4.6 | $9.45 | $47.25 | $189.00 |
| AWS Transcribe | $2.40 | $2.40 | $12.00 |
| Parallel AI (1 research/day) | $0.75 | $2.25 | $7.50 |
| **Total** | **~$39.53** | **~$78.83** | **~$235.43** |

Observed: heavy usage ran ~$125/month vs. the $189 projection — prompt caching cut input token costs by ~10x.

---

## How it works

**Stage 1 — Channel normalization.** grammY (Telegram) normalizes every incoming message to the same internal object before the model sees it.

**Stage 2 — Session lock.** One session, one active task at a time. Prevents concurrent tool calls from corrupting shared state.

**Stage 3 — Context assembly.** Workspace files (re-read from disk every turn), `MEMORY.md`, the session JSONL transcript, skill manifests (names only — full file loaded on demand).

**Stage 4 — Inference.** Structured function calling via Bedrock. Schema enforced at the provider level.

**Stage 5 — ReAct loop.** Tool call → run tool → feed result back → next decision. No fixed depth limit.

**Stage 6 — Memory.** Flat files: `MEMORY.md` for persistent facts, `memory/YYYY-MM-DD.md` for the daily log. Re-indexed into SQLite automatically. Human-readable, diffable, correctable.

**Stage 7 — Automation substrate.** Hooks, cron, Task Flows, Webhooks plugin. Durable stateful orchestration vs. stateless trigger-action pipelines.

---

## The knowledge base

```
voice memo (Telegram)
   → .oga file
   → S3 (staging bucket, 1-day lifecycle)
   → AWS Transcribe (async, ~60–120s)
   → transcript
   → LLM integration pass
   → wiki pages (entities, concepts, synthesis)
   → GitHub commit
   → index rebuild via GitHub Actions
   → SSM notify back to EC2
   → Telegram confirmation
```

Flat file library, not a vector database. The index is ~15K tokens at 500 entries — small enough to fetch entirely per query. Human-readable, correctable by editing files.

---

## The research agent

Parallel AI Task API. Structured query construction → async task submission → backoff polling → report received → GitHub commit → Actions rebuild → SSM notify → Telegram confirmation.

Fixed per-task pricing ($0.005–$0.250 depending on tier), not per-token. Predictable cost, no surprise bills from a runaway query.

---

## Emergent patterns

**Skills as operational memory.** Every solved problem becomes a reusable skill. Each session starts from a richer baseline. After a month, the agent knows how to do dozens of specific operations correctly without rediscovering them.

**The bot's self-healing instinct.** When something breaks, the agent works the thread. But it fixes the diagnosed target, not necessarily the actual code path in use. (See Bug 8 below — the most important bug in the appendix.)

**The working loop.** Error → feed to model → fix or diagnostic step → run → feed output back. Human driving, AI navigating when the environment gets noisy. The loop is the product.

---

## Bug appendix

The bugs that taught the most. Most are small. A few are structural.

**Bug 1: SSM Plugin install.** Manual `.pkg` extraction required. ARM64 vs x86_64 matters.

**Bug 2: SSM + WebSockets.** SSM port forwarding drops persistent WebSocket connections. The OpenClaw control UI's QR-flow (for WhatsApp pairing) requires a persistent WebSocket. Three hours lost to this.

**Bug 3: WhatsApp death spiral.** Health monitor restarts → stale credentials → Baileys retries → rate limit → repeat. Fix: stop service, delete credentials, pair fresh via terminal QR. Don't use WhatsApp for bots.

**Bug 4: Wrong Telegram config key.** `token` rejected, `botToken` required. Schema validator catches it but doesn't name the field.

**Bug 5: API key never reached the process.** `.env` correct, `.profile` source present — but systemd doesn't source `.profile`. Fix: `EnvironmentFile=` in the service unit. Verify via `/proc/<pid>/environ`.

**Bug 6: Voice memos silently ignored.** Gateway delivers a file path annotation. Model can't read audio. Every voice memo discarded since Telegram was connected.

**Bug 7: Wrong model was running.** Config change never deployed to EC2. A weaker model ran for days while Sonnet was assumed. Tell: workspace files progressively ignored as context grew. Fix: one sed command, restart.

**Bug 8: Bot diagnosed the wrong code path (the Sisyphus bug).** `ingest-voice.sh` sources `lib/git.sh` but never calls any of its functions. An inline `git push` on line ~229 had no retry logic. The bot fixed `lib/git.sh` correctly and repeatedly — the wrong file. Fix required human structural reading of the scripts. A race condition test (a second local clone positioned 1 commit ahead) would have caught this immediately.

**Bug 9: IAM permission prefix mismatch.** A script used `openclaw-wiki-*` job names. The IAM policy allowed `openclaw-voice-*`. Silent failure at runtime, not at deploy.

**Bug 10: `fileb://` vs `file://`.** AWS CLI's `--body file://` validates for ASCII. The prompt contained em-dashes. Fix: `fileb://` passes raw bytes, skips ASCII check.

**Bug 11: `git pull --rebase` syntax.** `git pull --rebase origin HEAD` is wrong. Should be `git pull --rebase origin main`. Retry logic never executed despite being present in the script.

**Bug 12: Research output rendering as raw JSON.** `output_schema: { type: "auto" }` returns JSON. `output_schema: { type: "text" }` returns readable markdown. One parameter, one week of broken reports.

**Bug 13: Speech-to-text errors propagating through the pipeline.** Transcribe heard "Lagos" instead of "Richmond." LLM pass treated transcript as ground truth. Seven files required correction.

Governance response: all transcribed content is a draft. Human review before promotion to canonical pages. Corrections commit with provenance notes. Raw transcripts stay immutable — the error chain must always be traceable.

---

## Detail on the structural bugs

### Bug 8 in depth: the Sisyphus pattern

The agent was asked to fix retry logic in voice ingestion. It read `ingest-voice.sh`, saw `source lib/git.sh`, and reasonably assumed git operations were happening through that library. They weren't. The actual `git push` was inline, in a different file, on a line the agent never opened.

The agent fixed `lib/git.sh` with elegant retry logic. Tested the change. Confirmed the fix. Deployed. The actual bug was untouched. Next failure, the agent fixed `lib/git.sh` again — slightly differently, equally elegantly, equally pointlessly.

The lesson: **structural reading beats local reading.** When a fix isn't taking, don't make a better local fix. Map the call graph. Find what's actually executing. The agent doesn't do this on its own — the human has to.

### Bug 9 continued: IAM prefix mismatch

The fix is simple — align the prefix in the script to match the policy. Finding it requires knowing to look at the IAM policy at all. The error surface said "AccessDenied" with no breadcrumb to the prefix mismatch. AWS errors are good at being technically accurate and useless.

### Bug 10: non-ASCII in AWS CLI request

`wiki-integrate.sh` included em-dashes in a prompt template. AWS CLI's `--body file://` expects ASCII and rejects anything else with a confusing error about the file. Fix: `fileb://` reads raw bytes, handles any encoding. Use `fileb://` any time a request body might include typographic punctuation.

### Bug 11: GitHub Actions multi-line output parsing

`grep -m1` on frontmatter worked until body content matched the pattern. Multi-line values written via `echo "summary=..." >> $GITHUB_OUTPUT` are truncated. Fix: Python frontmatter parser + heredoc delimiter syntax for multi-line `GITHUB_OUTPUT` values.

### Bug 12: research output as raw JSON

Parallel AI's `output_schema: { type: "auto" }` returns structured JSON. `{ type: "text" }` returns readable markdown. One parameter change deleted an entire downstream reformatting pipeline stage and the bugs that lived in it.

### Bug 13: speech-to-text errors propagating

Transcribe heard "Lagos" instead of "Richmond." The LLM integration pass treated the transcript as ground truth and built wiki pages around the wrong location. Seven files had to be corrected by hand.

The structural fix wasn't to make Transcribe more accurate — that's not in our control. It was to treat all transcribed content as **draft** until reviewed, and to keep raw transcripts **immutable** so the error chain is always traceable.

---

## What's next

- Knowledge base enrichment propagation (semantic cross-linking across wiki graph)
- Transcription upgrade to Whisper (faster-whisper running locally on the EC2 instance, with VAD filtering)
- Structured self-improvement via GitHub issues → cloud agent → deployed changes
- Dedicated Q&A Telegram topic grounded in the full knowledge base
