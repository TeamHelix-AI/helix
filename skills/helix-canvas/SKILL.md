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

Three verbs, never more (ratified contract):

```sh
# start server, open browser; prints {url, port, cursor, viewers}
${CLAUDE_PLUGIN_ROOT}/bin/helix canvas up   --canvas <slug> [--no-open] [--brief "<the job>"]

# card JSON on stdin -> {cardId, cursor}; existing id -> update
${CLAUDE_PLUGIN_ROOT}/bin/helix canvas push --canvas <slug>

# blocks; exit 0 delta, 3 timeout, 4 no-viewer, 5 server down
# --wake-on user returns only when a human acted (see Draining)
${CLAUDE_PLUGIN_ROOT}/bin/helix canvas wait --canvas <slug> --since <cursor> [--timeout <s>] [--wake-on user]
```

- `up` opens the browser itself, from inside the binary. `--no-open` is for
  headless or scripted runs — never a way around a permission prompt. If a
  prompt is declined, hand the user `! open <url>` and carry on.

Canvas data lives in `~/.helix/canvas/<project>/<slug>/` (SQLite event log,
`server.json` with port/pid, uploads) — one home for the whole machine, never
inside a repo. The server watches files for `file-window`/`artifact` cards and
streams changes to the browser.

## Session flow

1. **Confirm the slug** with the user (new canvas or resume — list
   `~/.helix/canvas/<project>/` if unsure). Never guess. Canvases are
   per-project by scope: the cwd names the project; pass `--dir` to target
   another one.
2. `${CLAUDE_PLUGIN_ROOT}/bin/helix canvas up --canvas <slug> --brief "<the job, in your words>"`
   — reuses a live server or starts one. **Pass `--brief` whenever the user
   has already told you what this is about.** An empty board with a brief
   opens working, that line under the mark; without one it asks *What are we
   working on?* — which reads, to someone who just told you, as though you
   did not hear them. The brief is also your first status pill.
3. **Blank canvas** (no topic given): bare `up`, no `--brief`. The app shows
   the centered prompt; `up` and `wait`, and the user's opening prompt
   arrives as a `prompt` card in the delta — begin from it.
4. On resume, fetch state first:
   `curl -s http://127.0.0.1:<port>/api/state` — cards, responses, cursor.
   Summarize open items to the user before pushing anything.
5. Run **explicit rounds**: push the round's questions/options as cards
   (grouped by `threadId` — threads are the stages/columns), then wait.
6. Between rounds, **drain the delta continuously** — react to every user
   note/comment within one cycle, even mid-build ("built", "on it", or a
   thread reply). Never let user input sit unacknowledged.
7. **Keep the status pill honest**: `--brief` sets the first one; after
   that push `{"type":"status","payload":{"text":"…"}}` when starting a
   chunk of work, when going idle, and at hand-offs — it renders bottom-left and is
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
or sign-off is **not a checklist**: push a `decision` card (Ratify/Decline — a
Decline is what spawns the "why?" question, per the rejection protocol), or
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

What counts as a human acting — and so what `--wake-on user` returns on —
is user `response.added` (kind ≠ reaction), user `card.created`, user
`card.updated` (gate ticks, board moves, lane toggles), and `client.error`.
`card.updated` is in there deliberately: without it you would sit through
answered gates.

During an active round, drain in the foreground (one Bash call holds ≤9 min):

```bash
${CLAUDE_PLUGIN_ROOT}/bin/helix canvas wait --canvas <slug> --since <cursor> --timeout 540 --wake-on user
```

`--wake-on user` does the holding: viewer churn, your own cards and reactions
advance the cursor inside the call, and it returns only when a human actually
acted. Exit 0 delta on stdout, 3 quiet, 5 server down — restart on the same
port. Bake the slug and cursor in as literal values; the cursor is the one
`up` printed, or the one your last wait returned.

- Use `printf '%s'` when piping JSON in shell — **zsh `echo` mangles `\n`**.
- Heredocs (`<<'EOF'`) for push payloads, never `echo`.
- Call the binary by its full path — a shell variable or function holding
  it stops the skill's pre-approval from matching, and every call prompts.
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
${CLAUDE_PLUGIN_ROOT}/bin/helix canvas wait --canvas <slug> --since <cursor> --timeout 3600 --wake-on user
```

Exit 0 wakes you with the delta; 5 means the server died — restart it on the
same port and re-arm. 3 is a quiet hour, not an ending: re-arm and carry on.

Rules:

- Timeouts and retry counts bound a *foreground drain*, never the session.
  Quiet ≠ over. No-viewer ≠ over (the user reopens the tab and expects you
  live).
- The slug and cursor must be baked in as literal values (background calls
  share no shell state, and a shell variable stops this skill's
  pre-approval from matching); the cursor is the one from your **last
  drain**.
- The session ends only explicitly: the user says they're done (terminal
  or canvas), or ratifies a closing decision. Only then do the End-of-
  session steps and stop re-arming the wait.
- If a turn ends while you still have canvas work of your own, the same
  rule applies — push a status card first, do the work, and re-arm.

## The rejection protocol (ratified — never violate)

1. The **"why" question exists only once a Decline has landed**. It goes out
   in the same push as your reaction to that verdict, edged
   `declined — why?` from the decision — never minutes later. Pushing it
   alongside the decision card, before any verdict, is a violation: it asks
   the user why they declined something they have not decided yet.
2. Answered decision cards are **spent**. Re-deciding = push a *retake*
   decision, edge `superseded by` from the old one. Never edit a verdict.
3. While building, drain and acknowledge between edits.
4. Everything waiting on the user must be reachable from **your turn**;
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

- A stale server restarts on the same port when you run `${CLAUDE_PLUGIN_ROOT}/bin/helix canvas up` again;
  open tabs reconnect. Manual equivalent: kill the pid in `server.json`,
  relaunch with `--port <same port>`.
- The event log survives restarts; state is never in memory only.

## What the app gives the user (so you can point them at it)

Amber **N waiting on you** chip → **● your turn**: every card blocking the
session, oldest first, each headed by the action it needs (Answer, Choose,
Ratify or decline, Tick N items) and answerable in place; ⌖ jumps to the
card on the canvas (jumping auto-unfolds whatever hides the target). Empty
it reads *Nothing is waiting on you* — which is the answer when a person
asks why you are still waiting. **▦ index** → every card, findable:
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
