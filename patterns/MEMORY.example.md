# Long-Term Memory

_Curated facts that persist across sessions. Update this during heartbeats by reviewing daily memory files._

---

## About [the user]

- [Name and how they prefer to be addressed]
- [Location, timezone if relevant]
- [Primary contact channel]
- [Anything stable about them that's relevant to the agent's work]

## Setup

- [Compute platform: e.g. AWS EC2 instance type/region]
- [Model: e.g. Claude Sonnet 4.6 via Bedrock]
- [Channel(s) connected and when]
- [Tool paths and what they do]
- [Research library URL and index URL]
- [Any persistent infrastructure: S3 buckets, queues, scheduled jobs]

## Preferences

- [Tone preferences — usually a one-line reference to SOUL.md]
- [Format preferences — emojis on/off, length, etc.]
- [Recurring preferences worth remembering — workflow, defaults, etc.]

## Projects

- [Active project repos and their purpose]
- [What kind of work the agent helps with on each]

## Lessons Learned

- [Operational gotchas the agent should remember — "when X breaks, do Y", "schema validation rejects Z field name", etc.]
- [Things that have gone wrong before and the actual fix]

---

## Why this file exists

`MEMORY.md` is the agent's long-term curated knowledge. ~1,000 tokens budgeted. It survives across all sessions. It is **not** injected into group chats — only into the main DM session — to prevent leaking private context into multi-user spaces.

### What goes here

- Stable facts about the user
- Stable facts about the system
- Operational lessons that apply across sessions
- Names of people, places, projects the agent should recognize

### What doesn't go here

- Conversational history (→ session JSONL handles that)
- Today's events (→ daily `memory/YYYY-MM-DD.md` log)
- Capabilities (→ `IDENTITY.md`)
- Operating rules (→ `AGENTS.md`)
- Personality (→ `SOUL.md`)

### How it grows

The agent maintains a daily log at `~/.openclaw/workspace/memory/YYYY-MM-DD.md`. During heartbeat cycles, the agent reviews recent daily logs and **promotes recurring facts** (mentioned 3+ times) into `MEMORY.md`.

That's the self-improving loop: the daily log is wide and noisy, `MEMORY.md` is narrow and stable, and the heartbeat is the filter that moves things from one to the other.

### How it shrinks

When a fact in `MEMORY.md` becomes stale (e.g. an old project that's wound down, a preference that's changed), the heartbeat process — or the user — removes it. There's no automatic decay; curation is manual.

### Privacy

`MEMORY.md` is **the file most likely to contain personal information.** In group chats, it's withheld. If you're publishing your repo or sharing your config: keep `MEMORY.md` out of any public artifact, or replace it with a sanitized template like this one.
