---
name: openclaw-customize-workspace
description: Turn the patterns/*.example.md templates into a real personal workspace for an OpenClaw deployment — fill in identity, user profile, tone, tools, operating rules, and initial memory. Use after AWS bootstrap and before deploy.
---

# Customize the OpenClaw Workspace

The six markdown files in `patterns/` are templates. They show the structure of an OpenClaw agent's prompt-time identity but contain placeholders, not content. This skill helps the user produce a real workspace ready to deploy.

**Important:** the customized workspace files contain personal information. They should live in a **private** working repo or a gitignored directory — never commit them to a public repo.

## 1. Create a private working repo

The customized workspace, deploy `Makefile`, and any tool implementations live here. This is the equivalent of `rvaopenclaw` for the original author — your own private fork-equivalent, not a fork of the public repo.

```bash
# In a directory of your choosing
mkdir -p ~/projects/my-openclaw
cd ~/projects/my-openclaw
git init -b main
echo "# My OpenClaw Config (private)" > README.md
git add README.md
git commit -m "init"
```

Push to a **private** GitHub repo. Do not push to a public repo.

## 2. Copy the patterns into a workspace directory

```bash
cd ~/projects/my-openclaw
mkdir -p workspace
cp /path/to/openclaw-public/patterns/*.example.md workspace/

# Rename, dropping .example
cd workspace
for f in *.example.md; do mv "$f" "${f%.example.md}.md"; done
```

Result: `workspace/SOUL.md`, `IDENTITY.md`, `USER.md`, `TOOLS.md`, `AGENTS.md`, `MEMORY.md`.

## 3. Fill each file in priority order

### SOUL.md (15 minutes)

This is the agent's personality. Every reply will inherit this tone.

- Pick a specific persona, not a generic one. "Be helpful and friendly" gets washed out by the rest of the prompt. Sharp, distinctive personas hold up.
- Two or three sentences describing the persona's voice are enough.
- Include 3–4 example replies showing the tone in action — these anchor the model better than abstract descriptions.

If the user is stuck, suggest they write 3–5 example agent replies first, in the tone they want, and reverse-engineer the persona description from those.

### IDENTITY.md (10 minutes)

What the agent is and what it can do — used when someone asks "what can you do?"

- Name (the user picks)
- Single emoji (used in confirmations)
- Capabilities — only list what's actually wired up. Don't claim research if you're not setting up the research pipeline.
- Channel info ("Available on Telegram as @<botname>")
- Pointer to the runtime location ("Running on AWS EC2 with Claude via Bedrock")

### USER.md (5 minutes)

Smallest file (~150 tokens). Stable facts about the user.

- Name and preferred form of address
- Pronouns
- Timezone
- Location (city or region — granularity is the user's call)
- Communication preferences in 2–3 bullets
- Any anti-preferences ("no corporate tone", "no emojis")

**Do not** put recurring facts here that change frequently — those go in `MEMORY.md`.
**Do not** put capability descriptions here — those go in `IDENTITY.md`.

### TOOLS.md (varies — depends on which tools are enabled)

Only document tools that are actually wired up on the EC2 instance. Each tool needs:

1. Plain-language description of when to use it
2. Exact shell command
3. Example invocations
4. What to do on failure

If a tool isn't installed, omit it entirely — don't document aspirational capabilities.

### AGENTS.md (60+ minutes — largest file)

Operating rules. This is where most behavior tuning lives.

- Capability corrections (model thinks it can't do something it actually can)
- Config change protocol
- Core rules for response style
- Topic-based routing (if using Telegram forum topics)
- Tool usage defaults
- Response formatting for the channel
- Any user-specific protocols (e.g. "ask before running expensive operations")

Start with the template's structure and adjust. Most users don't need every section.

### MEMORY.md (5 minutes initially, grows over time)

Long-term curated memory. Start almost empty.

- 5–10 bullets about the user's setup (instance ID, region, bucket name)
- 3–5 bullets about preferences worth remembering (tone, format)
- A few projects or contexts the agent should recognize
- Empty "Lessons Learned" section

The agent will grow this file over time via heartbeat cycles. Don't pre-populate it heavily.

**Privacy reminder:** `MEMORY.md` is the file most likely to contain personal information. It is withheld from group chats by default. Never publish it.

## 4. Sanity-check the token budget

Run a rough word count:

```bash
cd workspace
wc -w SOUL.md IDENTITY.md USER.md TOOLS.md AGENTS.md MEMORY.md
```

Total should be roughly 4,000–5,500 words (~5,000–7,000 tokens). If you're well above 8,000 words, the bootstrap is too heavy and will eat context window unnecessarily.

## 5. (Optional) Add a Makefile

Copy the Makefile pattern from the original author's repo, replacing:

- `STACK_NAME` with your stack name from the AWS bootstrap
- `REPO_URL` with your private repo URL
- `REPO_DIR` with the path on EC2 where the repo will be cloned

This is your deploy entry point. The `openclaw-deploy` skill explains the targets.

## 6. Commit and push to private repo

```bash
cd ~/projects/my-openclaw
git add workspace/ Makefile
git commit -m "initial workspace customization"
git push origin main
```

---

## Output of this skill

A private repo containing six populated workspace markdown files. The agent now has a personality, knows who the user is, knows what tools it has, and has a baseline of operating rules. It does not yet exist on EC2 — that's the next skill (`openclaw-deploy`).

## Common pitfalls

- **Generic persona.** "Helpful and friendly" produces flavorless replies. Be specific.
- **Aspirational capabilities.** Don't list tools in `IDENTITY.md` or `TOOLS.md` that aren't actually deployed. The agent will try to use them and fail.
- **Heavy initial MEMORY.md.** Let it grow. Pre-loading it with 50 bullets you think the agent should remember produces a stilted, fact-recital tone in early conversations.
- **Committing to a public repo.** Don't. The customized files contain personal data by definition.
