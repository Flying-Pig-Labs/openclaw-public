---
name: openclaw-deploy
description: Deploy a customized OpenClaw workspace to a running EC2 instance via SSM. Covers initial deployment, the auto-sync cron pattern, and how to verify a deploy succeeded. Use after openclaw-customize-workspace; assumes the EC2 instance is provisioned and the workspace files exist in a private repo.
---

# Deploy OpenClaw to EC2

OpenClaw deploys via SSM `send-command`, not SSH. The pattern: clone the private workspace repo on the instance, copy the workspace markdown into `~/.openclaw/workspace/`, copy any tool scripts into `~/.openclaw/tools/`, restart the gateway service.

## 1. One-time bootstrap on the instance

The first deploy needs the private repo cloned on the instance. This requires the instance to authenticate to GitHub — easiest path is a personal access token passed in via SSM.

**Make sure `GITHUB_TOKEN` is exported locally:**
```bash
export GITHUB_TOKEN=<your PAT with repo scope>
```

**Send the clone command via SSM:**
```bash
INSTANCE_ID=<from openclaw-aws-bootstrap output>
REGION=us-east-1
REPO_URL=https://github.com/<your-user>/<your-private-repo>.git
REPO_DIR=/home/ubuntu/openclaw-config

aws ssm send-command \
  --instance-ids "$INSTANCE_ID" \
  --document-name AWS-RunShellScript \
  --parameters "{\"commands\":[\"sudo -u ubuntu bash -c 'cd /home/ubuntu && git clone https://${GITHUB_TOKEN}@github.com/<your-user>/<your-private-repo>.git ${REPO_DIR}'\"]}" \
  --region "$REGION"
```

Wait ~10 seconds, then verify by SSH-ing in:

```bash
aws ssm start-session --target "$INSTANCE_ID" --region "$REGION"
ls /home/ubuntu/openclaw-config/workspace/
exit
```

You should see `SOUL.md`, `USER.md`, `AGENTS.md`, etc.

**Security note:** the GitHub PAT is now stored in the EC2 instance's git config. That's intentional — the auto-sync cron needs it to push back. But it does mean the PAT is recoverable by anyone with shell access to the instance. Use a token scoped to only the repos this deployment needs, and rotate it periodically.

## 2. The deploy command

The deploy itself copies workspace files from the cloned repo into the OpenClaw runtime directory. Pattern:

```bash
aws ssm send-command \
  --instance-ids "$INSTANCE_ID" \
  --document-name AWS-RunShellScript \
  --parameters '{"commands":["sudo -u ubuntu bash -c \"set -e; cd '"$REPO_DIR"' && git pull --ff-only; mkdir -p ~/.openclaw/workspace ~/.openclaw/tools; cp -v workspace/*.md ~/.openclaw/workspace/; if ls tools/*.sh >/dev/null 2>&1; then cp -v tools/*.sh ~/.openclaw/tools/ && chmod +x ~/.openclaw/tools/*.sh; fi; echo DEPLOY_DONE\""]}' \
  --region "$REGION" \
  --output text \
  --query 'Command.CommandId'
```

This pattern works as a Makefile target. See the original author's `Makefile` for the canonical implementation — copy it and adjust `STACK_NAME`, `REPO_DIR`, and `REGION` for your deploy.

## 3. Verify the deploy

```bash
# SSH in
aws ssm start-session --target "$INSTANCE_ID" --region "$REGION"

# On the instance:
ls -la ~/.openclaw/workspace/
cat ~/.openclaw/workspace/SOUL.md | head -5

# Check the gateway service is running
systemctl --user status openclaw-gateway.service
```

If the gateway service isn't installed yet, that's expected on a fresh deploy — see step 4.

## 4. Restart (or start) the gateway

After every deploy, the gateway should pick up changes automatically (it re-reads workspace files on every turn). But for the very first deploy, you need to start it once:

```bash
# On the instance:
systemctl --user enable openclaw-gateway.service
systemctl --user start openclaw-gateway.service
systemctl --user status openclaw-gateway.service
```

For most subsequent deploys, **no restart is needed** — workspace file changes are picked up live. The exceptions:

- Changes to `~/.openclaw/openclaw.json` (config file)
- Changes to environment variables in `/etc/environment`
- Tool script changes that involve dependency installation

For those, restart explicitly: `systemctl --user restart openclaw-gateway.service`.

## 5. (Optional) Auto-sync cron — bidirectional sync

Some deployments use a cron on the instance to push workspace edits back to the private repo. This is what makes runtime agent edits durable across `make deploy` cycles.

**The pattern:**
```bash
# /home/ubuntu/openclaw-config/tools/sync-workspace-to-git.sh
#!/usr/bin/env bash
set -euo pipefail
cd /home/ubuntu/openclaw-config

# Copy live workspace files back into the repo
cp ~/.openclaw/workspace/*.md workspace/

# Commit and push if there are changes
if ! git diff --quiet; then
  git add workspace/
  git commit -m "auto-sync: live workspace changes [$(date -u +'%Y-%m-%d %H:%M UTC')]"
  git push origin main
fi
```

**Crontab entry:**
```
*/5 * * * * /home/ubuntu/openclaw-config/tools/sync-workspace-to-git.sh >>/var/log/openclaw-sync.log 2>&1
```

**Tradeoffs of enabling auto-sync:**

- **Pro:** runtime edits (e.g., agent updating its own `MEMORY.md`) survive deploys
- **Pro:** acts as a continuous backup of the live workspace
- **Con:** every 5 minutes, the cron force-pushes whatever's in `~/.openclaw/workspace/` — if anything in your workspace contains data you don't want in your private repo, it gets there fast
- **Con:** if you gitignore a file in the public repo (e.g., for sanitization), the cron will keep recommitting it unless the script is also taught to skip it

If you enable auto-sync, **make sure your repo is private.** And make sure the deploy and sync don't fight each other (deploy pulls, sync pushes — they alternate cleanly if the cron interval is wider than the deploy window).

## 6. Iterating after the first deploy

Once everything is running, the loop is short:

```bash
# Local
vim workspace/AGENTS.md   # edit a rule
git add workspace/AGENTS.md
git commit -m "tighten AGENTS routing"
git push origin main

# Deploy to instance
make deploy
```

The agent picks up `AGENTS.md` changes on the very next turn. No restart.

---

## Output of this skill

- A private repo cloned on the instance under `/home/ubuntu/openclaw-config` (or wherever)
- Workspace markdown files synced into `~/.openclaw/workspace/` on the instance
- `openclaw-gateway.service` running and enabled
- (Optionally) auto-sync cron registered

The agent is now alive on EC2, but **no channel is connected yet** — Telegram, voice memos, etc. come in `openclaw-channel-setup`.

## Common pitfalls

- **`git clone` over SSM uses sudo correctly.** The clone must run as `ubuntu`, not as `root` (which is what SSM commands default to). Use `sudo -u ubuntu bash -c "..."` consistently.
- **Forgetting the chmod.** `cp` preserves permissions, but if the source isn't `+x`, the deployed script isn't either. Always `chmod +x` in the deploy command.
- **Stale workspace cache.** If a deploy doesn't seem to take effect, the gateway may still have the old session loaded. `systemctl --user restart openclaw-gateway.service` clears it.
- **Permissions on `~/.openclaw/`.** This directory must be owned by `ubuntu`. If a deploy ran as root once, fix with `sudo chown -R ubuntu:ubuntu /home/ubuntu/.openclaw`.
