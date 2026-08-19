---
name: canvas
description: Run a visual brainstorm/collaboration session on Helix Canvas — the local canvas app where Claude pushes cards (questions, options, decisions, diagrams, artifacts) and the user answers in the browser. Use when the user says "/helix-canvas", "visual brainstorm", "open a canvas", or wants to collaborate on the canvas instead of the terminal. The classic `brainstorm` skill stays untouched for terminal/doc sessions.
allowed-tools: Bash(${CLAUDE_PLUGIN_ROOT}/bin/helix *)
---

# Helix Canvas

Drive a collaboration session through Helix Canvas (part of Team Helix). You
push typed cards over a CLI; the user answers, comments, authors notes, and
drags things in a browser. The append-only event log is the record; the
canvas is the surface; the terminal is the fallback channel.

## Setup

The CLI lives in the Helix Canvas repo:

```
HX="${CLAUDE_PLUGIN_ROOT}/bin/helix canvas"
```

Three verbs, never more (ratified contract):

```
$HX up   --canvas <slug> [--no-open]        # start server, open browser; prints {url, port, cursor, viewers}
$HX push --canvas <slug>                    # card JSON on stdin -> {cardId, cursor}; existing id -> update
$HX wait --canvas <slug> --since <cursor> [--timeout <s>]
                                            # blocks; exit 0 delta, 3 timeout, 4 no-viewer, 5 server down
```

Canvas data lives in `~/.helix/canvas/<project>/<slug>/` (SQLite event log,
`server.json` with port/pid, uploads) — one home for the whole machine, never
inside a repo. The server watches files for `file-window`/`artifact` cards and
streams changes to the browser.

## Session flow

1. **Confirm the slug** with the user (new canvas or resume — list
   `~/.helix/canvas/<project>/` if unsure). Never guess. Canvases are
   per-project by scope: the cwd names the project; pass `--dir` to target
   another one.
2. `$HX up --canvas <slug>` — reuses a live server or starts one.
3. **Blank canvas** (no topic given): the app shows a centered chat-style
   prompt. Just `up` and `wait` — the user's opening prompt arrives as a
   `prompt` card in the delta; begin from it.
4. On resume, fetch state first:
   `curl -s http://127.0.0.1:<port>/api/state` — cards, responses, cursor.
   Summarize open items to the user before pushing anything.
5. Run **explicit rounds**: push the round's questions/options as cards
   (grouped by `threadId` — threads are the stages/columns), then wait.
6. Between rounds, **drain the delta continuously** — react to every user
   note/comment within one cycle, even mid-build ("built", "on it", or a
   thread reply). Never let user input sit unacknowledged.
7. **Keep the status pill honest**: push
   `{"type":"status","payload":{"text":"…"}}` when starting a chunk of
   work, when going idle, and at hand-offs — it renders bottom-left and is
   how the user knows you're alive mid-build.
8. Fold ratified decisions into the session's doc/memory as usual.

## Ownership & gates (two-way cards)

`todo`, `kanban`, and `gantt` carry an optional `payload.owner`:

- **`claude`** (default) — a progress display; read-only for the user;
  update it as you work (push with the same `id`).
- **`user`** — a **gate**: rendered ochre and marked *yours*; the user
  clicks todo items (pending → active → done), drags kanban cards between
  columns, toggles their gantt lane. Every change arrives in your delta as
  `card.updated` with `origin: "user"` — treat these as the signals for
  when you may proceed. WATCH for updates on any card you gate on.

**Pick the right gate.** A user-owned checklist is for hand-offs where
**every item is genuinely required** — the session (and Fleet's attention
queue) treats the card as pending until all items are done. A confirmation
or sign-off is **not a checklist**: push a `decision` card (Ratify/Decline —
Decline already spawns its "why?" question per the rejection protocol), or
`options` when there are real alternatives. Never pad a checklist with
filler items like "one more thing (leave a note)" — notes are always
possible via the card's thread, and an untickable item keeps the session
flagged in Fleet forever. An item that is truly optional gets
`optional: true`; it stays clickable but never holds attention. When a gate
has served its purpose, update it (all required items done) or `delete` it;
user-owned kanban/gantt cards count as pending for as long as they exist,
so hand them back (`owner: "claude"`) or delete them when the hand-off ends.

The dual gantt uses `payload.lanes: {claude: [tasks], user: [tasks]}` — one
chart, two lanes, the user lane toggleable.

## The user's screenshots are yours to read

Attachments the user adds (📎 or clipboard paste) are stored in
`~/.helix/canvas/<project>/<slug>/uploads/` and referenced as `/api/upload/<file>`.
**Read the file with the Read tool and look at it** — users paste
screenshots of visual bugs; diagnosing from the actual pixels is the
highest-bandwidth feedback loop this system has. Reference the same
uploads in `gallery` cards to show them back.

## The wait loop (cursor discipline — CRITICAL)

Two symmetric failure modes, both real:

- **Skipping the past**: `--since` from a push response silently drops
  events that arrived between your last drain and that push. Only use the
  cursor from your *last drain* or a fresh `/api/state` fetch.
- **Waiting for the past**: a fresh state cursor sits *after* everything
  already logged — if the user answered while you were building, the wait
  never sees it. So when waiting on a **specific card**, always pass
  `--card <id>`: the server returns `already-answered` with the responses
  instantly if the log already holds them. When waiting generally, first
  inspect the `/api/state` snapshot for answers you haven't processed.

Stop signals for a general drain loop: user `response.added` (kind ≠
reaction), user `card.created`, **user `card.updated`** (gate ticks, board
moves, lane toggles), and `client.error`. Forgetting `card.updated` means
sitting through answered gates.

During an active round, drain in the foreground (one Bash call holds ≤9 min):

```bash
CURSOR=$(curl -s http://127.0.0.1:$PORT/api/state | node -e "let s='';process.stdin.on('data',d=>s+=d).on('end',()=>console.log(JSON.parse(s).cursor))")
for i in $(seq 1 12); do
  OUT=$($HX wait --canvas $SLUG --since $CURSOR --timeout 45); CODE=$?
  # break on: user response.added (kind != reaction), user card.created, client.error
  # otherwise update CURSOR from $OUT and continue
  if [ $CODE -eq 4 ]; then break; fi   # no-viewer -> the endless wait below covers it
  if [ $CODE -eq 5 ]; then break; fi   # server down -> restart on the SAME port
done
```

- Use `printf '%s'` when piping JSON in shell — **zsh `echo` mangles `\n`**.
- Heredocs (`<<'EOF'`) for push payloads, never `echo`.
- zsh does **not** word-split unquoted `$VARS` — wrap the CLI in a shell
  function (`hx() { ${CLAUDE_PLUGIN_ROOT}/bin/helix canvas "$@"; }`), not a string variable.
- `client.error` events in the delta are app-side render failures — fix
  them; don't wait for the user to report.

## Never end a turn without a wait running (ratified)

A canvas session has no "probably done". Gates the user hasn't ticked,
threads they might reply in, notes they might drop at 2am — all of it must
resume you. So **the last tool call of every turn, for as long as the
session is open, is an endless wait launched with `run_in_background`** —
even when nothing appears to be pending. When it exits you are re-invoked
with the delta on stdout; drain, respond, and end the turn the same way.

```bash
# LAST tool call of the turn — Bash with run_in_background: true
HX="${CLAUDE_PLUGIN_ROOT}/bin/helix canvas"
while true; do
  OUT=$($HX wait --canvas $SLUG --since $CURSOR --timeout 3600); CODE=$?
  if [ $CODE -eq 0 ]; then printf '%s\n' "$OUT"; break; fi  # user acted -> you wake with the delta
  if [ $CODE -eq 5 ]; then printf '%s\n' "$OUT"; break; fi  # server down -> wake, restart on the SAME port, re-wait
  if [ $CODE -eq 4 ]; then sleep 60; fi                     # tab closed -> hold; they may come back
done                                                        # 3 = quiet hour -> keep holding
```

Rules:

- Timeouts and retry counts bound a *foreground drain*, never the session.
  Quiet ≠ over. No-viewer ≠ over (the user reopens the tab and expects you
  live).
- `$SLUG` and `$CURSOR` must be baked into the command (background calls
  share no shell state); `$CURSOR` is the cursor from your **last drain**.
- The session ends only explicitly: the user says they're done (terminal
  or canvas), or ratifies a closing decision. Only then do the End-of-
  session steps and stop re-arming the wait.
- If a turn ends while you still have canvas work of your own, the same
  rule applies — push a status card first, do the work, and re-arm.

## The rejection protocol (ratified — never violate)

1. A **Decline on a decision card spawns its linked "why" question in the
   same push** (edge labeled `declined — why?`). Never minutes later.
2. Answered decision cards are **spent**. Re-deciding = push a *retake*
   decision, edge `superseded by` from the old one. Never edit a verdict.
3. While building, drain and acknowledge between edits.
4. Everything waiting on the user must be reachable via the blockers panel;
   anything long-running on your side gets an in-progress card.

## Card reference (push JSON on stdin)

Common: `{"type", "threadId", "title"?, "payload"}`. Unknown types render as
a raw-payload fallback, so new types may be pushed before renderers exist.

| type | payload |
|---|---|
| `question` | `{prompt}` — free text, accepts multiple replies |
| `options` | `{prompt, options:[{id,label,description?}], mode?:"single"\|"multi"}` — user may attach a note |
| `decision` | `{topic, proposal}` — explicit Ratify/Decline |
| `markdown` | `{markdown}` (GFM: tables, fenced code) |
| `note` | `{markdown, attachments?}` — user-authored; you may push them too |
| `prompt` | `{text}` — the user's opening ask from a blank canvas (user-side) |
| `diagram` | `{mermaid}` — renders serialized; errors report back as `client.error` |
| `mindmap` | `{tree:{label,children?[]}}` — interactive: balanced two-side layout, click nodes to fold/unfold (+N badge); `{mermaid}` also accepted |
| `gantt` | `{sections:[{name,tasks:[{label,start,duration?\|end?,done?,active?}]}]}` or `{lanes:{claude,user}}` or `{mermaid}`; `owner?` |
| `timeline` | `{entries:[{period, items:[…]}]}` or `{mermaid}` |
| `kanban` | `{columns:[{id,title,cards:[{id,label}]}], owner?}` — user-owned = drag & drop |
| `todo` | `{items:[{id,label,state?:"pending"\|"active"\|"done",optional?}], owner?}` — `optional` items never hold attention |
| `chart` | `{kind, labels?, datasets:[…], height?}` — kinds: line, area, bar, hbar, stacked-bar, scatter, bubble, pie, doughnut, polar, radar; palette is built-in and CVD-validated |
| `metric` | `{tiles:[{label, value, unit?, delta?, tone?:"good"\|"bad"\|"neutral"}]}` — KPI tile row |
| `table` | `{columns:[...], rows:[[...]]}` |
| `code` | `{code, language?}` |
| `image` | `{src}` (http/data) or `{path}` (absolute file) |
| `gallery` | `{images:[{src\|path, caption?}]}` — first image is the hero |
| `video` | `{src\|path, poster?}` — local files served with range support |
| `html` | `{html, height?}` — sandboxed iframe, scripts allowed |
| `file-window` | `{path}` — live view, line comments, md preview/raw |
| `artifact` | `{path, status?:"forming"\|"stable"\|"done"}` — the deliverable; write the file with Write/Edit, the canvas streams it; modes preview/raw/meta + download |
| `weave` | `{text?}` — renders this canvas's event log as the double helix; Claude's reflection card |
| `diff` | `{diff, path?}` — unified-diff string, GitHub-style render; lines clickable for review comments (kind `comment` + line index in your delta). Review = diff card + line comments + decision card |
| `terminal` | `{output, command?, exitCode?}` — console-styled command transcript |
| `test-report` | `{suites:[{name,tests:[{name,status:"pass"\|"fail"\|"skip",message?,duration?}]}]}` |
| `api` | `{method, url, status?, duration?, requestBody?, responseBody?}` — request/response pair |
| `commit` | `{message, hash?, author?, stats?:{additions,deletions,files}, files?:[{path,additions?,deletions?}]}` |
| `compare` | `{left:{title}, right:{title}, rows:[{label,left,right,winner?:"left"\|"right"}]}` |
| `comment` | `{cardId, text, attachments?}` — reply in a card's thread (no threadId) |
| `edge` | `{from, to, label?}` — arrows; **always link follow-up cards to what prompted them** |
| `status` | `{text}` — not a card: updates the agent-status pill |
| `delete` | `{cardId}` — removes a card |

Update = push the same JSON with the existing `id`.

## Artifacts & graduation

Content starts as ordinary cards; when a decision stabilizes it, **promote**:
push an `artifact` card bound to the repo path, edges from the source
decisions (`written into`), keep the file written through with Write/Edit,
and advance `status` forming → stable → done via update pushes. Graduation =
every artifact card `done`; announce it and update the session's memory.

## Server operations

- A stale server restarts on the same port when you run `$HX up` again;
  open tabs reconnect. Manual equivalent: kill the pid in `server.json`,
  relaunch with `--port <same port>`.
- The event log survives restarts; state is never in memory only.

## What the app gives the user (so you can point them at it)

Amber **N waiting** chip → blockers panel, click-to-jump (jumping auto-
unfolds whatever hides the target). **▦ index** → every card, findable:
tabs by type group plus artifacts and attachments, search, stage/status
filters, pagination; row click opens the card's full preview, ⌖ jumps back
to the canvas focused on it. **◷ timeline** → the session in order: cards
added, answers, ratifications, replies and status changes, live-updating,
filterable by actor, searchable; row click previews the card, ⌖ jumps to
the canvas. **✓ decisions** → the register: every decision card as a
numbered entry (D1…) with verdict and timestamp, filterable by
open/ratified/declined, searchable; retakes nest their superseded
ancestors, declined entries link their "why" question. **☐ work** → who
is blocking whom: two columns — everything waiting on the user (open
questions/options/decisions, gates with required items left) and
everything Claude has committed to (open todos with progress, boards in
play, artifacts still forming, the current status); the toolbar filters
pending / done / all. **▣ stages** →
per-thread show/hide,
collapse (collapsed stages become summary cards that re-anchor crossing
edges), and ⤢ solo view. **compact** folds answered cards; **arrange**
re-stacks columns; theme toggle; ⤢ maximize on every card (full mode +
thread sidebar); 💬 threads (resizable panels); ⊘ hide + master reveal;
double-click empty canvas → note editor (📎 + clipboard paste); reactions;
activity toasts bottom-center; drag anything — user positions always win.

## End of session

Update the project's memory file with: canvas slug, port, cursor reached,
open blockers, ratified decisions, artifact statuses, and gate states.
Before stopping, clear spent gates — every user-owned todo/kanban/gantt
that is no longer waiting on the user gets updated to done or deleted, so
the session drops out of Fleet's attention queue.
