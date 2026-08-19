---
name: spark
description: Run a chat-shaped ideation session on Helix Spark — a linear conversation where the agent riffs with the user and curates a shelf of living idea cards (create, update, archive), converging to a clean harvest instead of a sprawl. Hard rule — the agent writes no files during a spark; the event log is the whole record. Use when the user says "/helix-spark", "open a spark", "let's riff", "bounce ideas", or wants early divergent ideation before anything is scoped or decided. Sibling of helix-canvas (deciding) and helix-stream (watching); Spark is for diverging.
allowed-tools: Bash(${CLAUDE_PLUGIN_ROOT}/bin/helix *)
---

# Helix Spark

Diverge, then harvest. You riff in a chat feed and curate a shelf of living
idea cards; the user bounces back, drops verdicts, and exports the
survivors. Boundary (ratified): **Spark = diverge and harvest, no decisions
· Canvas = converge and ratify · Stream = watch.**

## The hard rule (ratified — never violate)

**You write no files during a spark session.** No Write, no Edit, no shell
redirects — the append-only event log under `~/.helix/spark/` is the whole
record. Reading files is fine (ideating about a codebase needs it). Value
leaves the session in exactly two ways: the user clicks **⇩ export** in the
browser, or the survivors graduate into a Canvas/Brief/doc session that
writes under its own rules. If asked to save something mid-spark, offer
those instead.

## Setup

```
HK() { ${CLAUDE_PLUGIN_ROOT}/bin/helix spark "$@"; }
```

Three verbs, never more:

```
HK up   --spark <slug> [--no-open]   # start server, open browser; auto-restarts stale servers on the same port
HK push --spark <slug>               # card JSON on stdin -> {cardId, cursor}; existing id -> update
HK wait --spark <slug> --since <cursor> [--timeout <s>] [--card <id>]
                                     # exit 0 delta, 3 timeout, 4 no-viewer, 5 down
```

Sessions are **central**: data lives in `~/.helix/spark/<slug>/` — no
project, no cwd, no `--dir`. Pick a short topic slug; `up` with the same
slug resumes. **Warm resume:** when `up` resumes an existing shelf, open
with a recap `say` — survivor count, verdicts pending, the open question —
before riffing anew; the viewer shows its own resume digest, yours carries
the substance. Snapshot: `curl -s http://127.0.0.1:<port>/api/state`
(cards, responses, chat, cursor, `listening`, `lastEventAt`).

Shell discipline (same as siblings): the CLI is wrapped in a shell
function above because zsh does NOT word-split unquoted `$VARS` — a bare
`$HK up` breaks; heredocs (`<<'EOF'`) for push payloads, never `echo`.

## What you push

| type | payload |
|---|---|
| `say` | `{markdown}` — your conversational turn, rendered as a chat bubble |
| `idea` | `{pitch, detail?, theme?, status?:"live"\|"archived", archivedReason?}` — a shelf card; `title` = its name |
| `status` | `{text}` — singleton pill; what you're doing right now |
| `question` / `options` | inline asks — the only cards that count as "waiting on you" |
| `delete` | `{cardId}` — for true mistakes only; a dead idea is **archived**, not deleted |

Core types (`markdown`, `code`, `image`, `table`, `chart`, `diagram`, …)
also render in the feed; unknown types fall back to raw payload. No
threadId anywhere. Re-pushing an existing `id` updates in place — that is
how ideas live: sharpen the pitch, move the theme, archive.

**Ideas are the anti-gate.** They never count as pending, never appear in
Fleet's queue, never hold attention. Push them freely and early.

## The bounce (rhythm)

1. `up`, push `status`, open with a `say` that frames the territory —
   then seed the shelf with 2–3 ideas so the first glance shows the shape.
2. **Every user chat message gets bounced back**: reply with a `say` that
   riffs (variants, inversions, adjacent angles — quantity over polish)
   and do the shelf work in the same breath — new `idea` cards for
   anything with a name, updates to ones the message sharpened.
3. **Curate as you go, not at the end.** When two ideas turn out to be one,
   merge: update the survivor, archive the other with an `archivedReason`.
   When a direction dies in conversation, archive it then — a clean shelf
   *is* the deliverable. Give related ideas a shared `theme` tag; the
   shelf groups by it. Never invent themes ahead of the ideas.
4. Keep `status` honest; drain the delta between pushes and react to every
   user event within one cycle.

## User events in your delta (origin:"user")

- `chat.added {text}` — the conversation's other half; the primary signal.
- `response.added` with `kind:"verdict"`, value `keep | spark | kill` —
  a verdict on an idea. **kill** → archive it; the verdict may carry an
  optional `reason` (from the viewer's sweep prompt) — use it as the
  `archivedReason`, plus their call from chat if said. **spark** → they
  want more of this: riff variants, deepen the detail. **keep** → it's a
  survivor; leave it be. Verdicts break your wait (reactions don't).
- **A burst of verdicts in one delta = the shelf was just swept.** React
  in one pass: archive the kills, riff the sparks, then a single `say`
  wrapping the sweep — not one bounce per verdict.
- `graduation.requested {}` — the user clicked **⇧ graduate**. Confirm in
  a `say`, then run the crossing (below) without waiting to be asked.
- `response.added` (other kinds) — answers to your inline questions.
- `card.updated` — user-side board changes, if you pushed any owned cards.

## Harvest & graduation

There is no ratification in Spark. When the shelf has settled (the user
says so, verdicts have landed on everything live — the viewer shows a
"Shelf settled" banner then — or a `graduation.requested` event arrives):

1. Push a closing `say` summarizing the survivors by theme, and point at
   **⇩ export** — the browser downloads the live shelf as markdown,
   judged survivors first, the unjudged tail flagged.
2. Offer the crossing: survivors feed a Canvas session (each theme's
   keepers become an `options` round), a Brief, or a brainstorm doc — that
   session does the writing, under its own rules. On `graduation.requested`
   the offer is already accepted: confirm the destination and go. Print the
   harvest in the terminal if the user wants it in hand immediately; still
   write nothing.

## Server operations

- A stale server restarts on the same port when you run `$HK up` again.
  The event log survives restarts.
- App render failures arrive in your delta as `client.error` — fix them.

## Never end a turn without a wait running (ratified, family-wide)

A spark has no "probably done" — a 2am chat message must resume you. The
last tool call of every turn, for as long as the session is open, is an
endless wait launched with `run_in_background`. Your wait is also the
viewer's **listening dot**: while one is armed the composer shows
listening; while none is, it shows away and warns on send.

```bash
# LAST tool call of the turn — Bash with run_in_background: true
HK() { ${CLAUDE_PLUGIN_ROOT}/bin/helix spark "$@"; }
SINCE=<CURSOR>
while true; do
  OUT=$(HK wait --spark <SLUG> --since $SINCE --timeout 3600); CODE=$?
  if [ $CODE -eq 0 ]; then
    # hold through viewer churn — wake only for user events and render errors
    V=$(printf '%s' "$OUT" | node -e "let d='';process.stdin.on('data',c=>d+=c).on('end',()=>{try{const s=JSON.parse(d);const real=(s.events||[]).some(e=>e.origin==='user'||e.type==='client.error');console.log(real?'wake':('hold '+s.cursor))}catch{console.log('wake')}})")
    case "$V" in
      hold\ *) SINCE=${V#hold }; continue;;
      *) printf '%s\n' "$OUT"; break;;             # user acted -> you wake with the delta
    esac
  fi
  if [ $CODE -eq 5 ]; then printf '%s\n' "$OUT"; break; fi  # server down -> wake, restart on the SAME port, re-wait
  if [ $CODE -eq 4 ]; then sleep 60; continue; fi           # tab closed -> hold; they may come back
  if [ $CODE -ne 3 ]; then sleep 30; fi                     # unknown exit -> never hot-loop
done                                                        # 3 = quiet hour -> keep holding
```

Bake `<SLUG>` and the last drained `<CURSOR>` into the command — background
calls share no shell state. Cursor discipline is the family rule: only use
a cursor from your last drain or a fresh `/api/state`, never from a push
response. The session ends only when the user says so; only then stop
re-arming.
