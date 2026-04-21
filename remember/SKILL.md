---
name: remember
description: "Stash one specific fact, decision, lesson, or note into graphify memory as structured JSON plus a markdown mirror. Trigger - $remember followed by the thing to remember."
---

# remember

The user just told you something worth keeping. Stash it as one structured record under `./memory/<type>s/` (the folder uses the pluralized type name: `decisions`, `lessons`, `facts`, `notes`).

Dependencies: `jq` (for `$recall`), `graphify` CLI (optional, for `$compact` to fold memory into the graph).

## Steps

1. **Classify the type** — pick exactly one:
   - `decision` — an architectural choice
   - `lesson` — a bug / gotcha + root cause + fix
   - `fact` — a non-obvious detail
   - `note` — anything else

2. **Infer a question** the record answers. Keep it one sentence.

3. **Pick 1–3 node labels** from the recent conversation. Use concrete names (class, function, concept). If unsure, read `graphify-out/GRAPH_REPORT.md` and choose from god nodes. Never invent labels.

4. **Build a slug**: first 4–6 words of the question, kebab-case, lowercase.

5. **Build identifiers** (portable across macOS BSD `date` and GNU `date`; the `sed` adds the colon in the TZ offset so output is `+08:00` rather than `+0800`):
   ```bash
   TS=$(date "+%Y-%m-%d-%H%M")                                                   # filename timestamp
   TS_ISO=$(date "+%Y-%m-%dT%H:%M:%S%z" | sed 's/\([0-9][0-9]\)$/:\1/')          # ISO-8601 for JSON
   ```

6. **Create the type folder** if missing:
   ```bash
   mkdir -p memory/<type>s
   ```

7. **Write JSON** to `memory/<type>s/<TS>-<slug>.json`:
   ```json
   {
     "type": "decision",
     "timestamp": "2026-04-17T16:50:00-07:00",
     "question": "Why did we choose httpx over requests?",
     "answer": "Needed async-native client, HTTP/2, and typed Response. Switched 2026-Q2.",
     "raw_context": "Short quote or excerpt from what the user actually said, plus any code or error referenced.",
     "nodes": ["Client", "AsyncClient", "Response"]
   }
   ```

8. **Write markdown mirror** (same path, `.md` extension) so `graphify update` can ingest it:
   ```markdown
   ---
   type: decision
   timestamp: 2026-04-17T16:50:00-07:00
   question: Why did we choose httpx over requests?
   nodes: [Client, AsyncClient, Response]
   ---

   # Q: Why did we choose httpx over requests?

   ## Answer
   Needed async-native client, HTTP/2, and typed Response. Switched 2026-Q2.

   ## Raw context
   Short quote or excerpt from what the user actually said, plus any code or error referenced.
   ```

9. **Confirm** in ONE line:
   `stashed <type> → memory/<type>s/<file>.json (+ .md)`

## Rules

- **Do not run `graphify update`** here. Batch is cheaper — `$compact` runs it.
- **Never invent node labels** that aren't in the graph or the user's text.
- **Never fabricate raw_context.** If the user's statement was short, keep raw_context short.
- **One record per invocation.** If the user packed several items into one message, stash the most important one and suggest they `$remember` the others separately.
- **Be terse.** One confirmation line, no report.
