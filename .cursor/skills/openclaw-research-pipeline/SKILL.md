---
name: openclaw-research-pipeline
description: Set up the optional deep-research pipeline — Parallel AI for the research engine, a separate GitHub repo as the published research library, and GitHub Actions to rebuild the index on push. Skip this skill entirely if research is not a wanted capability.
---

# Research Pipeline (Optional)

The research pipeline gives the agent a `/research` capability — submit a query, get back a structured markdown report committed to a GitHub research library, and have a librarian index that the agent can use to recall past research.

This skill is **optional**. Skip it if the user doesn't want research capabilities.

## 1. Confirm the prerequisites

Before starting:

- Parallel AI account and API key (`PARALLEL_API_KEY`)
- A GitHub personal access token with `repo` scope
- A GitHub username/org where the research repo will live

If any of these are missing, send the user back to the prerequisites skill and add them.

## 2. Create the research repo

This is a **separate** GitHub repo from the private workspace repo. It can be public or private; public is reasonable since published research is generally not personally identifying.

```bash
# Locally — create the repo
mkdir -p ~/projects/my-research && cd ~/projects/my-research
git init -b main
mkdir -p _templates research voice-notes
```

**Add a research template:**
```markdown
<!-- _templates/research-template.md -->
---
title: "<query>"
date: "<YYYY-MM-DD>"
processor: "<lite-fast|core-fast|pro-fast|ultra-fast>"
run_id: "<parallel-run-id>"
tags: []
summary: "<one-sentence takeaway>"
---

# <query>

<full report from Parallel AI>
```

**Add a basic README and `.gitignore`:**
```bash
echo "# Research Library" > README.md
cat > .gitignore <<'EOF'
.DS_Store
*.tmp
EOF

git add . && git commit -m "init"
```

Push to GitHub:
```bash
git remote add origin https://github.com/<user>/<repo>.git
git push -u origin main
```

## 3. Add an index-rebuild script

The agent needs an `index.json` of all published research to do "what do we have on X" lookups. Build one with a small Python script that reads frontmatter from every markdown file:

```python
# scripts/build-index.py
import json, sys, pathlib, re
import yaml

corpus = []
for path in pathlib.Path("research").rglob("*.md"):
    text = path.read_text()
    match = re.match(r"^---\n(.*?)\n---", text, re.DOTALL)
    if not match:
        continue
    fm = yaml.safe_load(match.group(1))
    corpus.append({
        "url": f"https://github.com/<user>/<repo>/blob/main/{path}",
        "raw_url": f"https://raw.githubusercontent.com/<user>/<repo>/main/{path}",
        "title": fm.get("title", ""),
        "date": fm.get("date", ""),
        "tags": fm.get("tags", []),
        "summary": fm.get("summary", ""),
        "path": str(path),
    })

corpus.sort(key=lambda r: r["date"], reverse=True)
index = {
    "version": "1.0.0",
    "last_synchronized": __import__("datetime").datetime.utcnow().isoformat() + "Z",
    "corpus": corpus,
}
pathlib.Path("index.json").write_text(json.dumps(index, indent=2))
print(f"Indexed {len(corpus)} research docs")
```

Run it once locally to test:
```bash
pip install pyyaml
python scripts/build-index.py
git add scripts/ index.json
git commit -m "add index builder"
git push
```

## 4. Add a GitHub Actions workflow

Auto-rebuild the index every time research is pushed:

```yaml
# .github/workflows/index.yml
name: Rebuild research index

on:
  push:
    branches: [main]
    paths:
      - 'research/**'
      - 'voice-notes/**'
      - '_templates/**'
      - 'scripts/build-index.py'

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.12' }
      - run: pip install pyyaml
      - run: python scripts/build-index.py
      - name: Commit index if changed
        run: |
          if ! git diff --quiet index.json; then
            git config user.name "github-actions[bot]"
            git config user.email "github-actions[bot]@users.noreply.github.com"
            git add index.json
            git commit -m "ci: rebuild research index"
            git push
          fi
```

Push the workflow:
```bash
git add .github/
git commit -m "add index rebuild workflow"
git push
```

## 5. Build the publish tool

This script runs **on the EC2 instance** and is what the agent invokes when the user asks for research:

```bash
#!/usr/bin/env bash
# tools/research-and-publish.sh
# Usage: research-and-publish.sh "<query>" [processor] [filename]
set -euo pipefail

QUERY="$1"
PROCESSOR="${2:-pro-fast}"
FILENAME="${3:-}"

# Submit to Parallel AI
RUN_ID=$(curl -s -X POST https://api.parallel.ai/v1/tasks/runs \
  -H "x-api-key: $PARALLEL_API_KEY" \
  -H "content-type: application/json" \
  -d "$(jq -n --arg q "$QUERY" --arg p "$PROCESSOR" '{
    input: $q,
    processor: $p,
    output_schema: { type: "text" }
  }')" | jq -r '.run_id')

# Poll for completion (with backoff)
while :; do
  STATUS=$(curl -s -H "x-api-key: $PARALLEL_API_KEY" \
    "https://api.parallel.ai/v1/tasks/runs/$RUN_ID" | jq -r '.status')
  case "$STATUS" in
    completed) break ;;
    failed) echo "Parallel AI run failed" >&2; exit 1 ;;
    *) sleep 5 ;;
  esac
done

# Get the report
REPORT=$(curl -s -H "x-api-key: $PARALLEL_API_KEY" \
  "https://api.parallel.ai/v1/tasks/runs/$RUN_ID/result" | jq -r '.output')

# Generate filename if not provided
if [ -z "$FILENAME" ]; then
  SLUG=$(echo "$QUERY" | tr '[:upper:]' '[:lower:]' | tr -c 'a-z0-9' '-' | tr -s '-' | head -c 60)
  FILENAME="research/$(date -u +%Y-%m-%d)-${SLUG}.md"
fi

# Write the file
mkdir -p "$(dirname "$RESEARCH_DIR/$FILENAME")"
cat > "$RESEARCH_DIR/$FILENAME" <<EOF
---
title: "$QUERY"
date: "$(date -u +%Y-%m-%d)"
processor: "$PROCESSOR"
run_id: "$RUN_ID"
tags: []
summary: ""
---

# $QUERY

$REPORT
EOF

# Commit and push
cd "$RESEARCH_DIR"
git add "$FILENAME"
git commit -m "research: $QUERY"
git push origin main

# Print the GitHub URL for the agent to return
echo "https://github.com/<user>/<repo>/blob/main/$FILENAME"
```

This is a sketch. The actual published version handles errors more carefully — see the original author's `tools/research-and-publish.sh` for a battle-tested version.

**Required env vars on the instance:**
```
PARALLEL_API_KEY=...
GITHUB_TOKEN=...
RESEARCH_DIR=/home/ubuntu/openclaw-research  # local clone of the research repo
```

Set these in `/etc/environment` (system-wide) so they're available to the gateway service:
```bash
# On the instance
sudo bash -c 'cat >> /etc/environment <<EOF
PARALLEL_API_KEY=<key>
GITHUB_TOKEN=<token>
RESEARCH_DIR=/home/ubuntu/openclaw-research
EOF'
```

Restart the gateway so it picks up the env vars: `systemctl --user restart openclaw-gateway.service`

## 6. Clone the research repo on the instance

```bash
# On the instance
sudo -u ubuntu git clone "https://${GITHUB_TOKEN}@github.com/<user>/<repo>.git" "$RESEARCH_DIR"
```

## 7. Document the tool in TOOLS.md

Add a section to the deployed `~/.openclaw/workspace/TOOLS.md` documenting the new tool:

```markdown
## Research & Publish

Runs deep research via Parallel AI and publishes the result as a markdown file in the research repo.

**Command:** `~/.openclaw/tools/research-and-publish.sh "<query>" [processor] [filename]`

**Processors:** lite-fast, base-fast, core-fast, pro-fast, ultra-fast

**Use when:** the user says "research X", "look into X and save it", "write up a report on X"
```

Push the change to the workspace repo and re-deploy.

## 8. Verify end to end

In Telegram:

```
/research What are recent advancements in serverless architectures?
```

Expected behavior:

1. Agent acknowledges immediately ("acknowledged. running pro-fast research, ~30s.")
2. After ~30 seconds, agent posts a GitHub URL of the published report
3. The research repo's GitHub Actions runs and updates `index.json`

If the third step fails, check the Actions tab on the research repo and look at the workflow run.

---

## Output of this skill

- A research repo with the index-rebuild workflow
- An on-instance research tool (`~/.openclaw/tools/research-and-publish.sh`)
- Required env vars set in `/etc/environment`
- The local research repo cloned to `$RESEARCH_DIR`
- TOOLS.md updated with documentation
- Verified end-to-end research → published markdown → indexed → URL returned

## Common pitfalls

- **`output_schema: { type: "auto" }`** returns JSON, not markdown. Use `{ type: "text" }`. (This was bug 12 in the original build narrative — one parameter, one week of broken reports.)
- **GitHub Actions can't push** without `permissions: contents: write` in the workflow.
- **`git pull --rebase origin HEAD`** is wrong syntax. Use `git pull --rebase origin main`. (Bug 11 in the build narrative.)
- **Multi-line GitHub Actions outputs** with `echo "summary=..." >> $GITHUB_OUTPUT` get truncated. Use heredoc delimiter syntax for multi-line values.
- **Forgetting to restart** the gateway after adding env vars to `/etc/environment` — systemd doesn't pick them up otherwise.
