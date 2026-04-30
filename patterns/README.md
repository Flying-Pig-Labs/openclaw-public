# Patterns — Workspace Prompt Files

The six markdown files that make up an OpenClaw agent's prompt-time identity. These templates show the **structure** of each file — the personal content lives in your own private working repo.

| File | Role | Approx tokens |
|---|---|---|
| `SOUL.example.md` | Personality and tone | ~500 |
| `IDENTITY.example.md` | What the agent is, what it can do | ~400 |
| `USER.example.md` | Facts about the user | ~150 |
| `TOOLS.example.md` | Tool catalog and schemas | ~500 |
| `AGENTS.example.md` | Operating rules and protocols | ~2,500 |
| `MEMORY.example.md` | Long-term curated facts | ~1,000 |

**Bootstrap total: ~6,650 tokens.** About 3% of a 200K context window.

## How they're used

The OpenClaw gateway re-reads all six files from disk on every turn and concatenates them into the system prompt in this order:

1. System-level framework instructions
2. `SOUL.md`
3. `USER.md`
4. `TOOLS.md`
5. `AGENTS.md`
6. `MEMORY.md` (DM-only — withheld in group chats)
7. Today's + yesterday's `memory/YYYY-MM-DD.md` daily logs
8. Conversation history
9. Incoming message

To change agent behavior: edit a file, push to the deploy repo, sync to the EC2 instance. No restart. The next message uses the new prompt.

## Why six files instead of one

- **Token budgeting is per-file.** Easier to see what's eating context.
- **Different files change at different rates.** `SOUL` rarely; `MEMORY` daily.
- **`MEMORY` is conditional.** Group chats get the same `SOUL` and `AGENTS` but no `MEMORY`. The split makes that filter clean.
- **Diffs are scoped.** A change to tone is one file. A change to a tool is another. Easier to review.

## What's not here

- The runtime configuration (`openclaw.json`)
- The deploy `Makefile`
- The Lambda code for tool implementations
- The actual personal memory content

Those belong in a private working repo. These templates show the shape; what fills them is yours.
