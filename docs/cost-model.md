# Cost Model

Per-component cost breakdown for OpenClaw, observed and projected.

---

## Monthly cost across usage tiers

| Service | Light (~10 turns/day) | Moderate (~50/day) | Heavy (~200/day) |
|---|---|---|---|
| EC2 t4g.medium (24/7) | $26.93 | $26.93 | $26.93 |
| Claude Sonnet 4.6 (Bedrock) | $9.45 | $47.25 | $189.00 |
| AWS Transcribe | $2.40 | $2.40 | $12.00 |
| Parallel AI (1 research/day) | $0.75 | $2.25 | $7.50 |
| **Total (projected)** | **~$39.53** | **~$78.83** | **~$235.43** |

**Observed:** heavy usage actually ran ~$125/month vs. the $189 projection — prompt caching cut input token costs by ~10x.

---

## Where the money goes

### EC2 (~$27/month, fixed)

`t4g.medium` is ARM64/Graviton, 2 vCPU, 4GB RAM. Plenty for a Node.js gateway and an occasional Whisper transcription. Reserved instances drop this further if you commit. Spot is risky for an always-on agent.

The cost is the floor — even at zero turns, you're paying $27/month. It's a tax on persistence. Worth it.

### Bedrock — Claude Sonnet 4.6 (variable, dominant)

The biggest cost driver. With prompt caching:

- Cached input: ~$0.30/M tokens
- Uncached input: ~$3.00/M tokens
- Output: ~$15.00/M tokens

Workspace files (~6,650 tokens) are the same on every turn → 100% cache hit. That's the leverage. The session JSONL grows over time, but is also re-sent → high cache hit rate after the first turn of the day.

Heavy usage projection assumed 200 turns/day with 5K input tokens average. Real heavy days are more like 1.5K–3K average input (cached) plus 500 tokens output. Caching cuts ~70% of the projection.

### AWS Transcribe (variable, small)

$0.024/minute. A 5-minute voice memo costs $0.12. Heavy usage (10 voice memos/day at 2 min average) → ~$12/month.

If you replace Transcribe with locally-hosted faster-whisper on the same EC2 instance, this cost goes to zero — at the cost of CPU contention with the gateway during transcription. We've found it acceptable on `t4g.medium`.

### Parallel AI Task API (variable, small)

Per-task pricing, not per-token:

| Processor | Cost per task |
|---|---|
| `lite-fast` | ~$0.005–$0.01 |
| `base-fast` | ~$0.02–$0.03 |
| `core-fast` | ~$0.03–$0.05 |
| `pro-fast` | ~$0.04–$0.08 |
| `ultra-fast` | ~$0.10–$0.20 |
| `ultra2x` | ~$0.40–$0.60 |
| `ultra4x` | ~$0.80–$1.20 |
| `ultra8x` | ~$2.40 |

One research task per day at `pro-fast` → ~$2/month. The cost is bounded — you can't accidentally spend $50 on a research query the way you can with a runaway loop on a per-token API.

---

## What you're paying for vs. ChatGPT Plus ($20/month)

| Capability | ChatGPT Plus | OpenClaw (light) |
|---|---|---|
| Cost | $20/month | ~$40/month |
| Persistent memory across sessions | Limited, opaque | Full, file-backed, you own it |
| Always-on (no manual session start) | No | Yes |
| Reachable from any chat client | Web/app only | Telegram |
| Voice memo transcription | App-only, opaque | Pipelined into knowledge base |
| Deep research output | Plus tier | Same; cheaper per task |
| Proactive (can message you first) | No | Yes (via heartbeats) |
| Multi-channel (DM + groups) | No | Yes |
| You own the conversation history | No | Yes (`.jsonl` files on your VM) |
| Hackable, scriptable, tool-extensible | Limited | Yes |

The premium over ChatGPT Plus buys: ownership of state, persistence, proactivity, and the ability to wire arbitrary tools into the loop. Whether that's worth $20/month extra is personal.

---

## Hidden costs

**Time.** Three days to build initially. ~1 hour/week of maintenance once stable. Bug fixes, prompt refinements, new tools.

**Cognitive overhead.** When the agent gets something wrong, it's on you to figure out which workspace file or tool needs fixing. ChatGPT does this for you (badly, opaquely). You're trading "magic that mostly works" for "transparency you can debug."

**Lock-in.** None. The whole thing is markdown files and bash scripts. Switch providers, switch channels, switch models — the workspace files port unchanged.
