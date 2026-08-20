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

Three verbs, never more:

```sh
# start server, open browser; auto-restarts stale servers on the same port
${CLAUDE_PLUGIN_ROOT}/bin/helix spark up   --spark <slug> --agent <name> --model <model-id> [--no-open]
#   --agent: your product name, lowercase (claude, codex, …)
#   --model: your exact model id (e.g. claude-fable-5) — Fleet shows and filters by these

# card JSON on stdin -> {cardId, cursor}; existing id -> update
${CLAUDE_PLUGIN_ROOT}/bin/helix spark push --spark <slug>   # card JSON (one object, or an array of them) on stdin

# exit 0 delta, 3 timeout, 4 no-viewer, 5 down
# --wake-on user returns only when a human acted
${CLAUDE_PLUGIN_ROOT}/bin/helix spark wait --spark <slug> --since <cursor> [--timeout <s>] [--card <id>] [--wake-on user]
```

- `up` opens the browser itself, from inside the binary. `--no-open` is for
  headless or scripted runs — never a way around a permission prompt. If a
  prompt is declined, hand the user `! open <url>` and carry on.

Sessions are **central**: data lives in `~/.helix/spark/<slug>/` — no
project, no cwd, no `--dir`. Pick a short topic slug; `up` with the same
slug resumes. **Warm resume:** when `up` resumes an existing shelf, open
with a recap `say` — survivor count, verdicts pending, the open question —
before riffing anew; the viewer shows its own resume digest, yours carries
the substance. Snapshot: `curl -s http://127.0.0.1:<port>/api/state`
(cards, responses, chat, cursor, `listening`, `lastEventAt`).

Shell discipline (same as siblings): always call the binary by its full
path — a shell variable or function holding it stops the skill's
pre-approval from matching, and every call then prompts. Heredocs
(`<<'EOF'`) for push payloads, never `echo`.

## Rails first

Before proposing anything, read the rails that bind this project on this surface:

```sh
${CLAUDE_PLUGIN_ROOT}/bin/helix rails for --surface spark
```

What comes back is not advice — it is the standard the work is held to. Cite
rail ids when you act on them; full text with `helix rails get <id>`. Details
in the helix-rails skill.

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

- A stale server restarts on the same port when you run `spark up` again.
  The event log survives restarts.
- App render failures arrive in your delta as `client.error` — fix them.
- **Kill by pid, never by name.** `pkill -f "helix spark serve"` matches every
  spark server on the machine — other projects, other people's sessions, runs
  in the middle of their work. It looks local and is not. Read the pid instead:
  `kill "$(sed -n 's/.*"pid": *\([0-9]*\).*/\1/p' ~/.helix/spark/<slug>/server.json)"`.

## Never end a turn without a wait running (ratified, family-wide)

A spark has no "probably done" — a 2am chat message must resume you. The
last tool call of every turn, for as long as the session is open, is an
endless wait launched with `run_in_background`. Your wait is also the
viewer's **listening dot**: while one is armed the composer shows
listening; while none is, it shows away and warns on send.

```bash
# LAST tool call of the turn — Bash with run_in_background: true
${CLAUDE_PLUGIN_ROOT}/bin/helix spark wait --spark <slug> --since <cursor> --timeout 3600 --wake-on user
```

`--wake-on user` holds through viewer churn and a closed tab, returning only
when a human acted or the app hit a render error. Exit 0 wakes you with the
delta; 5 means the server died — restart it on the same port and re-arm. 3 is
a quiet hour, not an ending.

Bake `<SLUG>` and the last drained `<CURSOR>` into the command — background
calls share no shell state. Cursor discipline is the family rule: only use
a cursor from your last drain or a fresh `/api/state`, never from a push
response. The session ends only when the user says so; only then stop
re-arming.
