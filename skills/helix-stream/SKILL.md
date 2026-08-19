---
name: stream
description: Stream live updates and visualizations to the Helix Stream dashboard — a watch-first browser surface where the agent pushes status tiles and a feed of rich cards during autonomous sessions; the user watches, drops steering notes, holds the brake, and answers rare inline questions. Use when the user says "/helix-stream", "open a stream", "stream this session", or wants to watch long autonomous work (feature builds, test runs, incident monitoring, refactors, /loop sessions) from the browser. Sibling of helix-canvas (deciding) and helix-loop (gauntlet); Stream is for watching.
allowed-tools: Bash(${CLAUDE_PLUGIN_ROOT}/bin/helix *)
---

# Helix Stream

The work is autonomous; dialogue is the exception. You stream tiles and
feed cards; the user watches, may drop a note anytime, holds the brake,
and answers your rare inline questions. The append-only event log is the
record. Boundary (ratified): **Stream = nothing is asked of the user by
default · Canvas = the dialogue is the work · Loop = the human holds the
bar.**

## Setup

```
HS="${CLAUDE_PLUGIN_ROOT}/bin/helix stream"
```

Three verbs, never more:

```
$HS up   --stream <slug> [--dir <projectDir>] [--no-open]  # start server, open browser; auto-restarts stale servers on the same port
$HS push --stream <slug> [--dir]                            # card JSON on stdin -> {cardId, cursor}; existing id -> update
$HS wait --stream <slug> --since <cursor> [--timeout <s>] [--card <id>] [--dir]
                                                            # exit 0 delta, 3 timeout, 4 no-viewer, 5 down
```

Data: `~/.helix/stream/<project>/<slug>/` (SQLite event log + `server.json`)
— one home for the whole machine, never inside a repo.
Streams are **per-session**: one task, one stream — pick a fresh slug per
session (e.g. the task's short name). **Always pass `--dir`** with the
project root in scripts/background sessions (cwd is not reliable).
Snapshot: `curl -s http://127.0.0.1:<port>/api/state`.

Shell discipline (same as siblings): wrap the CLI in a shell function —
zsh does NOT word-split unquoted `$VARS`; heredocs (`<<'EOF'`) for push
payloads, never `echo`.

## What you push

Re-pushing an existing `id` updates in place. `status` is a singleton
(the server forces `id: "status"`).

**Tiles** — pinned dashboard, update in place, keep them honest all session:

| type | payload |
|---|---|
| `status` | `{text, phase?}` — what you are doing RIGHT NOW |
| `progress` | `{label, done, total}` |
| `metric` | `{tiles:[{label, value, unit?, delta?, tone?:"good"\|"bad"\|"neutral"}]}` |
| `tasks` | `{items:[{label, state?:"pending"\|"active"\|"done"}]}` — read-only plan |
| `chart` | canvas chart payload + `"zone":"tile"` to pin (charts default to feed) |

**Feed** — chronological cards; payloads identical to Helix Canvas types:
`update` (`{markdown}`), `code`, `diff`, `terminal`, `test-report`, `api`,
`commit`, `image`, `chart`, `table`, `diagram`, plus:

- `milestone` `{label}` — section break; push one when a phase completes.
- `question` / `options` — the escape hatch (see below).
- `artifact` `{path, status?:"forming"|"stable"|"done"}` — a deliverable
  (report, doc, generated file). Write the file with Write/Edit; the
  stream watches it live. Shows in the feed AND collects under the
  **artifacts tab**. Advance status with update pushes.
- Any feed card may carry `payload.tone: "good"|"bad"|"neutral"` —
  use `bad` for failures, `good` for wins; it colors the card edge.
- `delete` `{cardId}` works; there are no threads and no edges.

**Actions** — native toolbar/tabs, never a bare link in prose:

- `action` `{label, url, preview?:boolean}` — with `preview: true` it
  becomes a **tab** embedding the URL (dev servers, dashboards); without,
  a **↗ toolbar button** opening a new browser tab (docs, PRs). Push one
  the moment a dev server is up. Re-push same id to update; `delete`
  removes the button/tab when the server goes away.

No threadId anywhere. Unknown types render as a raw-payload fallback.

## The rhythm

1. `up` with a fresh slug; push `status` + `tasks` (the plan) immediately —
   the user's first glance must answer "what is it doing and what's the plan".
2. Work. Push an `update`/`code`/`test-report`/... at every meaningful
   step — a feed the user reads after 20 minutes away should tell the
   story without them asking. Push `milestone` between phases. Update
   `status`, `progress`, and `metric` tiles as you go; a stale status
   tile is a bug.
3. **Drain between steps**: run `wait` with a short timeout between work
   chunks (or check after each push — the push response cursor tells you
   nothing about user events; always `wait --since` your last drained
   cursor). React to every user event within one cycle.
4. Exit 4 (no-viewer) is fine — Stream is ambient; keep working and
   keep pushing. Do NOT stop streaming because nobody is watching; the
   feed is also the record they read later.

## User events in your delta (origin:"user")

- `note.added {text, cardId?}` — a steering note; acknowledge it in the
  feed (an `update` referencing it) and obey it. Notes are the primary
  steering channel.
- `brake {action:"pause"|"resume"|"stop"}` — **architecture, not a
  suggestion**: on `pause`, finish the current atomic step, push a status
  update saying you're holding, and `wait` until `resume` or `stop`; on
  `stop`, wind down cleanly and summarize.
- `response.added` — an answer to an inline question.

## Inline questions (the escape hatch — use sparingly)

Only when genuinely blocked. Push a `question`/`options` card with
`payload.default` (what you'll do if unanswered) and `payload.windowS`.
Then KEEP WORKING on other parts; check with
`wait --card <id> --timeout <s>` between chunks. When the window closes
unanswered, proceed with the default and say so in an `update`. Never
push rounds of questions — that's a Canvas session; if real collaboration
is needed, say so and suggest switching.

## Server operations

- A stale server restarts on the same port when you run `$HS up` again.
  The event log survives restarts.
- App render failures arrive in your delta as `client.error` — fix them.

## End of session — linger, don't vanish

1. Push a final `milestone` ("done") + closing `update` with the summary,
   set `status` to idle, mark all `tasks` done/skipped. Outcomes become
   surfaces: `action` cards for anything running (preview tab for the dev
   server), `artifact` cards for anything written (report, doc).
2. **Then keep a wait armed — indefinitely.** The user often reads the
   finished feed and reacts — a note arriving after "done" is a request,
   not noise. Quiet cycles and no-viewer results do NOT end the session:
   the last tool call of every turn, "done" included, is an endless wait
   launched with `run_in_background`, so any late note, brake, or answer
   re-invokes you. NEVER end the session with an undrained delta.

```bash
# LAST tool call of the turn — Bash with run_in_background: true
HS="${CLAUDE_PLUGIN_ROOT}/bin/helix stream"
while true; do
  OUT=$($HS wait --stream $SLUG --dir $PROJECT --since $CURSOR --timeout 3600); CODE=$?
  if [ $CODE -eq 0 ]; then printf '%s\n' "$OUT"; break; fi  # user acted -> you wake with the delta
  if [ $CODE -eq 5 ]; then printf '%s\n' "$OUT"; break; fi  # server down -> wake, restart on the SAME port, re-wait
  if [ $CODE -eq 4 ]; then sleep 60; fi                     # no viewer -> hold; they may come back
done                                                        # 3 = quiet hour -> keep holding
```

   Bake `$SLUG`, `$PROJECT`, and the last drained `$CURSOR` into the
   command — background calls share no shell state. The session ends only
   when the user says so (a `stop` brake or an explicit done in the
   terminal); only then stop re-arming.
3. The stream stays reopenable via `up` with the same slug; say so in
   the terminal summary.
