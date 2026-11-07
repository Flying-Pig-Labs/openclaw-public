---
name: openclaw-channel-setup
description: Connect the deployed OpenClaw instance to a Telegram bot — token configuration, group/topic setup, allowlist gates, the requireMention semantics. Use after openclaw-deploy; assumes the gateway is running on EC2.
---

# Connect Telegram to OpenClaw

OpenClaw is a Node.js gateway that handles channel I/O. Telegram is the primary supported channel. This skill walks through linking a Telegram bot to a running OpenClaw gateway.

## 1. Confirm the bot token

You should already have a token from the prerequisites skill. Verify it still works:

```bash
TELEGRAM_TOKEN=<your token>
curl -s "https://api.telegram.org/bot${TELEGRAM_TOKEN}/getMe" | jq '.result | {username, first_name}'
```

Should print the bot's username and display name.

## 2. Configure the gateway

OpenClaw reads channel config from `~/.openclaw/openclaw.json`. The Telegram block lives at `channels.telegram`.

**Important:** the config key is `botToken`, not `token`. (This is one of the bugs from the build narrative — the schema validator rejects `token` but doesn't name the field clearly.)

**Minimum config:**
```json
{
  "channels": {
    "telegram": {
      "botToken": "<your-token>",
      "groupPolicy": "allowlist",
      "dmPolicy": "open",
      "requireMention": true
    }
  }
}
```

**Edit on the instance via SSM:**
```bash
aws ssm start-session --target "$INSTANCE_ID" --region us-east-1

# On the instance:
sudo -u ubuntu bash
cd ~/.openclaw

# Back up first — the schema validator is strict
cp openclaw.json openclaw.json.bak

# Edit with jq (safer than ad-hoc text editing)
jq '.channels.telegram = {
  "botToken": "'"$TELEGRAM_TOKEN"'",
  "groupPolicy": "allowlist",
  "dmPolicy": "open",
  "requireMention": true
}' openclaw.json > openclaw.json.new && mv openclaw.json.new openclaw.json

# Restart so the new config is read
systemctl --user restart openclaw-gateway.service
systemctl --user status openclaw-gateway.service
```

## 3. Test direct messages first

Before any group setup, confirm DMs work:

1. On Telegram, search for your bot by username
2. Start a chat
3. Send a message: `hi`

The bot should reply within a few seconds. If not:

```bash
# On the instance:
journalctl --user -u openclaw-gateway.service -n 50 --no-pager
```

Look for: `[telegram] connected as @<botname>` or similar success messages. If you see auth errors, the token is wrong. If you see no messages reaching the gateway, check that the bot's webhook isn't pointing somewhere else (`getWebhookInfo`).

## 4. Group chat configuration

If you want the bot in groups (Telegram supergroups, with or without forum topics), you need to:

### a. Add the bot to the group

In Telegram, add the bot as a member like any other user.

### b. Disable Telegram's privacy mode (optional but usually needed)

By default, group bots only see messages that mention them or reply to them. To let the bot see all messages in a group:

1. Message [@BotFather](https://t.me/BotFather)
2. `/setprivacy` → choose your bot → `Disable`

This affects what the bot *receives*. The OpenClaw `requireMention` setting affects what it *responds to*.

### c. Get the group's chat ID

```bash
# On the instance, watch the gateway logs
journalctl --user -u openclaw-gateway.service -f

# In Telegram, send any message in the group
# The log will show something like:
#   [telegram] message from chat_id=-1001234567890 thread_id=2 ...
```

Group chat IDs start with `-100` for supergroups. Save the ID.

### d. Add the group to the allowlist

```bash
# On the instance:
cd ~/.openclaw

GROUP_ID="-100XXXXXXXXXX"

jq --arg gid "$GROUP_ID" '.channels.telegram.groupAllowFrom = [($gid | tonumber)]' \
   openclaw.json > openclaw.json.new && mv openclaw.json.new openclaw.json

systemctl --user restart openclaw-gateway.service
```

## 5. Forum topics (optional, but recommended)

Telegram forum topics let one group host multiple isolated agent contexts. Each topic gets its own session JSONL.

For a forum group:

1. Enable "Topics" in the group's settings (Telegram app)
2. Create topics — for example: `#research`, `#knowledge-base`, `#ops`
3. Each topic has a `MessageThreadId` — a small integer (1, 2, 3, etc.). The first topic is usually `1`. The "General" topic is always `1`.

To get a topic's thread ID, send a message in it and watch the logs:
```
[telegram] message from chat_id=-1001234567890 thread_id=8 ...
```

Configure per-topic behavior in `openclaw.json`:

```json
{
  "channels": {
    "telegram": {
      "groups": {
        "-1001234567890": {
          "topics": {
            "2":  { "requireMention": false, "systemPromptAddendum": "Research mode. Auto-publish all queries." },
            "8":  { "requireMention": false, "systemPromptAddendum": "Knowledge-base mode. Ingest notes; answer questions from KB." },
            "251":{ "requireMention": false, "systemPromptAddendum": "Sysadmin mode. System ops + issue filing only." }
          }
        }
      }
    }
  }
}
```

`requireMention: false` means the bot responds to every message in the topic — appropriate for dedicated single-purpose topics.

## 6. The three access gates (mental model)

For every incoming group message, three checks run:

1. **Access gate** — is the group/sender in the allowlist? If not, the message is dropped entirely (not even buffered).
2. **Mention gate** — does the message `@mention` the bot or reply to it? Only matters if `requireMention: true`.
3. **Topic gate** — if topic-specific config exists, it overrides group-level defaults.

Common misconfigurations:

- Bot added to group but not in allowlist → silent. No errors, no responses.
- Allowlist entry is a string instead of a number → silent. JSON schema accepts strings but the gate compares as numbers.
- Privacy mode still on at BotFather → bot only sees mentions, even in topics with `requireMention: false`.

## 7. Verify end-to-end

In each topic (or the main group):

1. Send a test message
2. Confirm the bot responds (or stays silent, depending on `requireMention`)
3. Check the JSONL transcript on the instance:
   ```bash
   ls ~/.openclaw/agents/*/sessions/
   tail -5 ~/.openclaw/agents/*/sessions/<sessionId>.jsonl | jq .
   ```

You should see your message and the bot's reply as separate JSONL entries.

---

## Output of this skill

- The bot is responding in DMs
- The bot is in any intended groups, with the allowlist correctly populated
- (If used) Forum topics have per-topic config
- The gateway logs show clean message → response cycles

## Common pitfalls

- **Wrong config key.** `token` instead of `botToken` → silent rejection by the schema validator.
- **String vs number for chat IDs.** Allowlist must contain numbers. Quotes around the ID kills the gate.
- **Privacy mode left on.** Bot only sees mentions even in `requireMention: false` topics.
- **Forgetting to restart the gateway.** Config changes don't apply until restart.
- **Webhook leftover from a previous deploy.** If a Telegram webhook is set, the gateway's polling client gets nothing. Run `setWebhook` with an empty URL to clear: `curl -X POST "https://api.telegram.org/bot${TOKEN}/setWebhook" -d url=`
