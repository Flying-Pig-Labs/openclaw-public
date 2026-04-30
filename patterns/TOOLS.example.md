# Tools

## Deep Research (Parallel AI)

You have access to a deep web research tool. Use it when you need to research a topic thoroughly with real-time web data — current events, company info, government data, market analysis, etc.

**Command:** `~/.openclaw/tools/parallel-research.sh "<query>" [processor]`

**Processors (pick based on depth needed):**
- `lite-fast` — quick fact lookups (10–20s)
- `base-fast` — standard enrichment, 2–5 fields (15–50s)
- `core-fast` — cross-referenced, moderate complexity (15s–100s)
- `pro-fast` — exploratory web research, default choice (30s–5min)
- `ultra-fast` — multi-source deep research (1–10min)

**When to use which:**
- Simple facts → `lite-fast`
- Company/person background → `core-fast`
- General research questions → `pro-fast`
- Comprehensive analysis → `ultra-fast`

## Research & Publish to GitHub

Runs deep research AND publishes the results as a markdown file to a GitHub research repo.

**Command:** `~/.openclaw/tools/research-and-publish.sh "<query>" [processor] [filename]`

- If filename is omitted, one is auto-generated from the query
- Results are committed and pushed to GitHub automatically
- Each file includes YAML frontmatter with the query, processor, run ID, and date

**Use this tool when asked to:**
- "Research X and save it"
- "Write up a report on X"
- "Look into X and publish it"
- Any research task where the output should be preserved

## Research Index (Librarian Lookup)

A structured JSON manifest of all published research. Fetch this first when asked about past research — it's fast, small, and always current.

**Index URL:** `https://raw.githubusercontent.com/<your-org>/<your-research-repo>/main/index.json`

**Command:** `web_fetch <index_url>`

**Schema:**
```json
{
  "version": "1.0.0",
  "last_synchronized": "2026-04-09T08:30:00Z",
  "corpus": [
    {
      "query": "the original question",
      "summary": "one-sentence takeaway",
      "tags": ["topic", "tags"],
      "url": "https://github.com/.../path-to-doc.md",
      "date": "2026-04-08"
    }
  ]
}
```

## Voice Memo Transcription

Telegram/WhatsApp voice memos are processed automatically by the gateway — you don't need to invoke this tool yourself. The pipeline:

1. User sends voice memo
2. Gateway annotates the message with `[media attached: <local_path>]`
3. Tool uploads to S3, calls AWS Transcribe (async)
4. Transcript saved as a structured markdown note to the research library
5. Confirmation sent back to user with a preview and a GitHub link

If you see a voice memo annotation, **acknowledge first** ("got it, transcribing — usually 60–120 seconds"), then let the pipeline complete.

## GitHub Issue Filing

**Command:** `~/.openclaw/tools/create-github-issue.sh "<title>" "<body>" [labels...]`

- Default repo is configured in environment
- Always confirm title and body with the user before filing
- Returns the issue URL on success

---

## Why this file exists

`TOOLS.md` is the tool catalog the agent reads at prompt assembly. Each tool has:

1. A natural-language description of when to use it
2. The exact shell command to invoke
3. Examples
4. Failure modes and what to do about them

Without explicit tool documentation, the agent either (a) doesn't know the tool exists, or (b) hallucinates command syntax. Both are common failure modes. The fix is documentation, not better prompting.

Keep tool descriptions short. Half a screen each is plenty.
