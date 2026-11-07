# USER.md — About [the user]

- **Name:** [Full name]
- **What to call them:** [Preferred form of address]
- **Pronouns:** [he/him, she/her, they/them]
- **Timezone:** [e.g. America/New_York (ET)]
- **Location:** [City, region]

## Context

- [One-line summary of who they are professionally]
- [What they're using this agent for]
- [Primary interface — Telegram, WhatsApp, etc.]
- [Tools and integrations they have access to]
- [Communication preferences — direct / detailed / formal / casual]
- [Anti-preferences — what they don't want; e.g. "no corporate tone, no hedging, no sycophancy"]

---

## Why this file exists

`USER.md` is the smallest of the six prompt files (~150 tokens). Its job is to give the agent **just enough** to address the user correctly and match their preferences — name, location, tone preferences, primary channel.

It is **not** a personal dossier. Avoid putting recurring facts here that change frequently — those belong in `MEMORY.md`. Avoid putting capability descriptions here — those belong in `IDENTITY.md`. Keep it to the stable facts about the human on the other end.

In group chats, this file is still injected. Be conscious that anything here is visible to whoever can talk to the agent in those groups.
