---
name: recall
description: "Two-tier retrieval from graphify memory and the knowledge graph. Returns summary-level answers first (cheap); only loads raw_context if the summary is insufficient or the user replies 'dig'. Trigger - $recall followed by a topic or question."
---

# recall

Answer the user's question using stashed memory (`$remember`), session compactions (`$compact`), and the graphify knowledge graph. Default to cheap retrieval; escalate to raw context only when necessary.

Dependencies: `jq` (required), `graphify` CLI (optional — graph query is skipped if missing).

## Tier 1 — summaries + graph only (always free)

1. **Check sources exist.** If both `./memory/` and `./notes/sessions/` are empty or missing, reply:
   `no memory yet — use $remember or $compact to start building it.`
   Stop.

2. **Load only light fields from JSON** (never `raw_context` in Tier 1). Filename is passed explicitly via `--arg fn` so each result carries a real `source_file`:
   ```bash
   find memory notes/sessions -type f -name "*.json" 2>/dev/null | while IFS= read -r f; do
     jq --arg t "<topic>" --arg fn "$f" \
       'select(
          (.summary // [] | join(" ") | test($t;"i"))
          or (.answer // "" | test($t;"i"))
          or (.topic // "" | test($t;"i"))
          or (.question // "" | test($t;"i"))
          or (.nodes_referenced // .nodes // [] | join(" ") | test($t;"i"))
        )
        | {session_id, timestamp, type, topic, question, summary, nodes: (.nodes_referenced // .nodes), source_file: $fn}' "$f" 2>/dev/null
   done
   ```

3. **Query the graph** (also free — no LLM). Guard both the CLI and the graph file:
   ```bash
   if command -v graphify >/dev/null 2>&1 && [ -f graphify-out/graph.json ]; then
     graphify query "<topic>" --graph graphify-out/graph.json
   fi
   ```
   If the CLI or graph is missing, skip silently — Tier 1 still runs on memory JSON alone.

4. **Synthesize a Tier-1 answer** in 3–6 bullets:
   - Cite each finding with `timestamp` / `session_id` / file path.
   - Pull directly from `summary` bullets and graph node labels. Do not open `raw_context`.
   - If a fact is partial or ambiguous, call that out — do not invent.

5. **Self-check** — did the summaries answer the question directly, or did you have to hedge?
   - **Sufficient:** stop. End with one line: `Need deeper context? Reply 'dig'.`
   - **Insufficient:** proceed to Tier 2 automatically, announcing `summary was thin, loading raw_context…`

## Tier 2 — open `raw_context` only if needed

Triggered when Tier 1 was insufficient, or the user replies `dig` to a previous `$recall`.

1. Pick the 1–3 most relevant sessions/memory files from Tier 1 results.
2. Load their `raw_context`:
   ```bash
   jq '{session_id, timestamp, topic, raw_context}' <path>.json
   ```
3. **Re-synthesize** with the extra detail:
   - Clearly mark which claims came from `raw_context` vs `summary` (e.g. `[raw]` / `[summary]` tags).
   - Quote the raw context directly when helpful.
4. **Report the cost honestly** at the end:
   `used raw_context from <N> session(s), ~<sum of raw_context_tokens_estimate> tokens loaded`.

## Rules

- **Do not load `raw_context` in Tier 1** — even if it looks convenient. The whole point of this split is that summaries are cheap.
- **Do not invent fields** that weren't in the JSON.
- **Do not offer `$remember` in the same turn** as a `$recall` — keep the flows separate.
- **Cite sources.** Every claim must trace to a `session_id` / timestamp / file path, or to a graph node label. If you can't cite, you can't include it.
- **Be terse.** Bullets, not paragraphs.
