---
name: compact
description: "Summarize the current Codex session into a structured JSON (session_id, summary, raw_context, decisions, lessons) plus markdown mirror, then fold it into the graphify knowledge graph. Trigger - $compact."
---

# compact

The user wants this session's knowledge captured in one structured record and folded into the knowledge graph.

Dependencies: `jq` (for `$recall`), `graphify` CLI (optional — skipped gracefully if not installed).

## Steps

1. **Infer a topic** — 2–4 words, lowercase, kebab-case. Examples: `auth-redesign`, `cache-debugging`, `transport-refactor`.

2. **Build timestamps** (portable across macOS BSD `date` and GNU `date`; the `sed` adds the colon in the TZ offset so output is `+08:00` rather than `+0800`):
   ```bash
   TS_FILE=$(date "+%Y-%m-%d-%H%M")
   TS_ISO=$(date "+%Y-%m-%dT%H:%M:%S%z" | sed 's/\([0-9][0-9]\)$/:\1/')
   ```

3. **Resolve the session ID.** Use the first that yields a value:
   - `$CODEX_SESSION_ID` env var, if set
   - Basename of newest Codex session file:
     ```bash
     ls -t ~/.codex/sessions/*.jsonl 2>/dev/null | head -1 | xargs -n1 basename | sed 's/\.jsonl$//'
     ```
   - Fallback: `local-<TS_FILE>-<4-char-random>`

   Also capture the full path to the Codex session file if found.

4. **Write JSON** to `./notes/sessions/<TS_FILE>-<topic>.json` (create dir with `mkdir -p ./notes/sessions`):
   ```json
   {
     "type": "session_compact",
     "session_id": "01JFQE...",
     "codex_session_path": "~/.codex/sessions/01JFQE...jsonl",
     "timestamp": "2026-04-17T16:52:00-07:00",
     "topic": "auth-redesign",
     "summary": [
       "Chose DigestAuth over BearerAuth because legacy API requires it",
       "Nonce handling must survive retries — added test case",
       "Transport abstraction stays so sync/async share auth code"
     ],
     "summary_tokens_estimate": 42,
     "raw_context": "Quotes and key excerpts from the conversation. Include specific user statements, code snippets touched, decision points. Not a verbatim transcript — the material a future agent would need to reconstruct the reasoning.",
     "raw_context_tokens_estimate": 820,
     "decisions": [
       {"what": "Use DigestAuth", "why": "Legacy API requires it"}
     ],
     "lessons": [
       {"symptom": "Auth retry looped", "root_cause": "Nonce not reset on 401", "fix": "Reset nonce on WWW-Authenticate"}
     ],
     "nodes_referenced": ["DigestAuth", "Response", "Client"],
     "files_touched": ["./auth.py", "./client.py"]
   }
   ```

   Field rules:
   - `summary` — 3–6 bullets. Decisions, bugs fixed, gotchas, non-obvious facts. No filler.
   - `raw_context` — the substrate the summary is derived from. Quote actual user statements. This is the expensive field — `$recall` loads it only when summary is insufficient.
   - Estimate tokens as `word_count * 1.3`.
   - `decisions` / `lessons` — omit if empty.
   - `nodes_referenced` — only labels that actually exist in `graphify-out/graph.json`.
   - If Codex session file not found, set `session_id` to the fallback and omit `codex_session_path`.

5. **Write markdown mirror** to `./notes/sessions/<TS_FILE>-<topic>.md` (same folder as the JSON):
   ```markdown
   ---
   type: session
   session_id: 01JFQE...
   timestamp: 2026-04-17T16:52:00-07:00
   topic: auth-redesign
   nodes: [DigestAuth, Response, Client]
   ---

   # Session — auth-redesign

   ## Summary
   - <bullet 1>
   - <bullet 2>

   ## Decisions
   - **Use DigestAuth** — legacy API requires it

   ## Lessons
   - Auth retry looped because nonce wasn't reset on 401 → reset on `WWW-Authenticate`

   ## Raw context
   <compact paragraph of raw_context>
   ```

6. **Fold into the graph** — only if `graphify-out/` exists AND the `graphify` CLI is installed:
   ```bash
   if [ -d graphify-out ] && command -v graphify >/dev/null 2>&1; then
     graphify update .
   fi
   ```
   If `graphify-out/` is missing, tell the user to run `$graphify .` first (memory file is still saved; it will be folded in on the next `$graphify .` or `graphify update .`). If the CLI is missing, tell the user `graphify CLI not installed — record saved to disk but not indexed into the graph`. Do not attempt a fresh build from inside this skill.

7. **Confirm** in 2 lines:
   - `session <session_id> → ./notes/sessions/<file>.json`
   - `graph updated (<N> new nodes)` or `graph not present — run $graphify . first`

## Rules

- **Never invent node labels** — only use ones present in `graph.json` or explicitly named in the conversation.
- **raw_context must be grounded** — quote what was actually said, don't paraphrase into something more polished.
- **Append-only, one file per invocation.** Multiple `$compact` calls in one session are expected and useful — they capture distinct phases without over-distilling. Never overwrite or delete a prior session file. Filenames disambiguate via timestamp + topic; `session_id` inside the JSON ties them together for later retrieval.
- **Be terse.** Two confirmation lines, no report.
