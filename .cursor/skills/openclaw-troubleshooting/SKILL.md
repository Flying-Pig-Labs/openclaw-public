---
name: openclaw-troubleshooting
description: Diagnose and fix common OpenClaw deployment issues — bot not responding, gateway service crashed, config rejected, deploy failing, voice memos ignored, etc. Use this skill when something is broken; consult it BEFORE proposing fixes, to avoid the Sisyphus pattern of fixing the wrong file.
---

# OpenClaw Troubleshooting

Map of symptoms → likely causes → fixes. Drawn from the original author's bug catalog. Read this **before** proposing a fix — many of these failures look like one thing and are actually another.

## Master rule: read before you fix

The most expensive bug in the original deployment was Bug 8 — the agent kept fixing `lib/git.sh` because the script `source`d it, even though the actual `git push` was inline in a different file and the library functions were never called. The agent fixed the wrong file, repeatedly, elegantly, pointlessly.

Before changing any file in response to a failure:

1. Read the script being executed end-to-end
2. Identify the **actual** code path that runs (not the imports, not the includes — the lines that execute)
3. Confirm your proposed fix is on that path

If you can't quickly trace the code path that's failing, **stop and ask** instead of guessing. Structural reading beats local reading.

---

## Symptom: bot doesn't respond in DMs

| Likely cause | Verification | Fix |
|---|---|---|
| Wrong token | `curl https://api.telegram.org/bot<TOKEN>/getMe` returns `ok:false` | Get a fresh token from BotFather |
| Token in wrong config field | `jq '.channels.telegram.botToken' ~/.openclaw/openclaw.json` returns null but `.token` exists | Rename `token` → `botToken` |
| Gateway not running | `systemctl --user status openclaw-gateway.service` shows inactive/failed | `systemctl --user restart openclaw-gateway.service`; check logs |
| Webhook left over | `curl https://api.telegram.org/bot<TOKEN>/getWebhookInfo` shows a URL | Clear it: `curl -X POST .../setWebhook -d url=` |
| Schema validation rejected the config | gateway logs show `config validation failed` | Restore `openclaw.json.bak`; edit with `jq` not text editor |
| Bedrock not enabled in region | logs show `AccessDeniedException` from bedrock | Enable model access in the Bedrock console |

---

## Symptom: bot doesn't respond in groups

| Likely cause | Verification | Fix |
|---|---|---|
| Group not in allowlist | logs don't show the message arriving from that chat ID | Add `chat_id` (as number, not string) to `groupAllowFrom` |
| Privacy mode on at BotFather | bot only sees @-mentions in the group | BotFather → `/setprivacy` → Disable |
| `requireMention: true` and message has no mention | message arrives in logs but no response | Either mention the bot or set `requireMention: false` for that topic |
| Wrong topic ID | response goes to General topic instead of the intended one | Check `MessageThreadId` in logs; update topic config |

---

## Symptom: deploy command appears to succeed but workspace doesn't update

| Likely cause | Verification | Fix |
|---|---|---|
| `git pull` failed silently | SSH in, `cd /home/ubuntu/<repo> && git status` | Resolve conflicts; redeploy |
| Permissions wrong on `~/.openclaw` | `ls -ld ~/.openclaw` shows root ownership | `sudo chown -R ubuntu:ubuntu /home/ubuntu/.openclaw` |
| Deploy ran as root, not ubuntu | SSM command didn't use `sudo -u ubuntu` | Update Makefile to use `sudo -u ubuntu bash -c "..."` |
| Stale gateway holding old session | new behavior not taking effect | `systemctl --user restart openclaw-gateway.service` |

---

## Symptom: voice memos are silently ignored

The agent replies normally to text but ignores voice memos. This was Bug 6 in the original build. The cause: the gateway delivers a `[media attached: <path>]` annotation in the message, but without the right tool wired up, the model has nothing to do with it — it just sees an annotation and treats it like text.

**Verify:**
1. Send a voice memo
2. Check `~/.openclaw/agents/*/sessions/<id>.jsonl` for the message
3. Look for `[media attached: ...]` in the message text

**Fix:**
- Confirm `~/.openclaw/tools/transcribe-voice.sh` exists and is executable
- Confirm `AGENTS.md` has a "Voice Memo Protocol" section telling the agent to invoke the transcription tool when it sees the media annotation
- Confirm the tool's IAM permissions are right (Transcribe + S3 write/read on the staging bucket)
- Confirm the staging bucket name in env matches the one in the script (Bug 9 — silent prefix mismatch)

---

## Symptom: research output is garbled JSON instead of readable text

Bug 12 in the original build. Cause: Parallel AI's `output_schema: { type: "auto" }` returns structured JSON; `{ type: "text" }` returns markdown.

**Fix:** in the research script, change the output schema to `{ "type": "text" }`. Re-run.

---

## Symptom: workspace files revert after a while

The auto-sync cron is overwriting the deployed workspace with an older version, or vice versa.

**Verify:**
```bash
# On the instance
crontab -l
cat /var/log/openclaw-sync.log | tail -20
git -C /home/ubuntu/<repo> log --oneline -5
```

**Likely causes:**
- Cron pulling from the wrong branch
- Cron pushing edits made by an earlier deploy back over your latest local edit
- Auto-sync interval too short, racing with your deploy

**Fix:** disable the cron temporarily, do a clean deploy, re-enable. If repeated: widen the cron interval, or stop it from running during active development.

---

## Symptom: AWS CLI rejects a request body for "non-ASCII content"

Bug 10 in the original build. Cause: `--body file://<path>` validates ASCII; em-dashes and similar typographic punctuation in your prompt template are rejected.

**Fix:** change `file://` to `fileb://` (raw bytes, no ASCII check).

---

## Symptom: gateway crashes on startup with config validation errors

The schema validator catches malformed config but doesn't always name the field clearly.

**Fix:**
1. SSH to the instance
2. `cd ~/.openclaw && cp openclaw.json.bak openclaw.json` (revert)
3. Restart, confirm it starts
4. Make changes via `jq` filter (not text editing), one field at a time
5. Restart between each change

---

## Symptom: deploys take effect but EC2-side edits to MEMORY.md disappear

If you don't have the auto-sync cron, runtime edits the agent makes to its own files (especially `MEMORY.md` during heartbeats) get overwritten on the next `make deploy` because deploy copies `workspace/*.md` from the repo over the live files.

**Fix:** enable the auto-sync cron (see `openclaw-deploy` skill, section 5). It pushes runtime edits back to the repo so deploys don't clobber them.

---

## Symptom: Bedrock returns AccessDeniedException

| Likely cause | Fix |
|---|---|
| Model not enabled in region | Bedrock console → Model access → request access |
| Wrong model ID (raw vs inference profile) | Use the inference profile ID (e.g. `us.anthropic.claude-sonnet-4-6`), not the raw model ID |
| Instance role missing `bedrock:InvokeModel` | Add the policy to the instance IAM role |
| Wrong region in API call | Confirm region matches where Bedrock model is enabled |

---

## Symptom: agent claims it can't do something it should be able to

The base model's training tells it it can't do certain things — process audio, browse the web, etc. Without an explicit "Capability Corrections" section in AGENTS.md, the model defaults to its training and confabulates limitations.

**Fix:** add a "Capability Corrections" block to AGENTS.md listing every actually-deployed capability and explicitly correcting the false negatives. See `patterns/AGENTS.example.md` for the pattern.

---

## Symptom: cost is higher than expected

| Likely cause | Verification | Fix |
|---|---|---|
| Prompt caching not engaging | input token costs in Bedrock metrics are full price, not 10% | Check that workspace files are stable across turns (caching requires identical prefix) |
| Wrong model running | logs show a different model ID than expected | Bug 7 — config change wasn't deployed. `make deploy`, restart |
| VPC endpoints enabled when not needed | CloudFormation parameter `CreateVPCEndpoints=true` | Re-deploy with `false` if you don't need them; saves ~$22/mo |
| Unused S3 lifecycle | audio bucket has no expiration → builds up | Add 1-day lifecycle (see `openclaw-aws-bootstrap`) |

---

## Last resort: clean restart

If everything is broken and you can't figure out why:

```bash
# On the instance
systemctl --user stop openclaw-gateway.service
cd ~/.openclaw

# Save current state
tar czf ~/openclaw-broken-$(date +%Y%m%d-%H%M).tgz workspace/ openclaw.json

# Restore from repo
cd ~/<repo>
git pull
cp -f workspace/*.md ~/.openclaw/workspace/

# Restart
systemctl --user restart openclaw-gateway.service
journalctl --user -u openclaw-gateway.service -n 30 --no-pager
```

If even that doesn't work, the issue is probably in `openclaw.json` or in IAM. Verify those last because they're the most invasive to change.

---

## When to give up and ask

If the symptoms don't match any of the above, and the agent has tried 2 fixes that didn't work, **stop fixing and report**. Send the user:

1. The exact symptom (with logs)
2. What's been tried
3. The current state

Don't continue the Sisyphus pattern of fixing one more thing. The right next step is human structural review.
