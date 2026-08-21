---
name: archive
description: Curate a session's story into Helix Archive — the read-only record where the agent distills what happened (chapters, decisions, findings, artifacts, memory candidates) into an append-only log the user reads in the browser. Use when the user says "/helix-archive", "archive this session", "snapshot what we did", "write up what happened", or at the end of exploratory work worth keeping. Works for any session, especially ones that ran outside a Helix surface. Sibling of helix-stream (watching live); Archive is the story written afterwards.
allowed-tools: Bash(${CLAUDE_PLUGIN_ROOT}/bin/helix *)
---

# Helix Archive

The session already happened — here, in this conversation. Your job is to
curate it into a story someone can read in five minutes: what was tried,
what was decided and why, what came out of it. The log is **append-only**:
an archive is a record, not a workspace. Nobody answers anything in it;
the one thing a viewer can do is promote a memory candidate.

Boundary (ratified): **Stream = watching work as it runs · Archive = the
story written afterwards.** If the user wants to watch live, that is a
stream; archive it when it's over.

## Setup

```sh
# what level is this archive at, and where did the last snapshot stop?
# reads the log directly; works with no server
${CLAUDE_PLUGIN_ROOT}/bin/helix archive info --archive <slug> [--dir <projectDir>]

# start server, open browser
${CLAUDE_PLUGIN_ROOT}/bin/helix archive up   --archive <slug> --agent <name> --model <model-id> [--dir <projectDir>] [--no-open]
#   --agent: your product name, lowercase (claude, codex, …)
#   --model: your exact model id (e.g. claude-fable-5) — Fleet shows and filters by these

# card JSON on stdin -> {cardId, cursor}
${CLAUDE_PLUGIN_ROOT}/bin/helix archive push --archive <slug> [--dir]   # card JSON (one object, or an array of them) on stdin

# promote clicks arrive here; exit 0 delta, 3 timeout, 4 no-viewer, 5 down
${CLAUDE_PLUGIN_ROOT}/bin/helix archive wait --archive <slug> --since <cursor> [--timeout <s>] [--wake-on user]
```

Data: `~/.helix/archive/<project>/<slug>/` — one home for the whole
machine, never inside a repo. One archived session, one slug: name it
after the session's short topic (e.g. `vault-flake-hunt`). **Always pass
`--dir`** with the project root in scripts (cwd is not reliable).

- `up` also brings up Fleet (mission control) when it is down, and returns
  `fleet: {url, started}`. Hand the user both links in one line —
  `Board: <url> · Fleet: <fleet.url>` — so they know the overview is there.
  `up` never opens Fleet in the browser; the user follows the link.

Shell discipline (same as siblings): always call the binary by its full
path; heredocs (`<<'EOF'`) for push payloads, never `echo`.

## The flow — every invocation starts with `info`

1. **`info` first.** `status: "empty"` means a fresh archive: ask the
   user for a level if they didn't give one, then push the `session`
   card before anything else. `status: "ok"` means you are appending:
   the level is already set (never ask again, never mix levels) and
   `watermark` tells you where the last snapshot stopped — curate only
   what happened after it.
2. **Curate from the record, not from memory.** Reread the conversation;
   if it is long and early parts were summarized away, read the session
   transcript from disk (`~/.claude/projects/<project>/<uuid>.jsonl`)
   rather than trusting your recollection of hour one.
3. **Open a chapter** (`chapter` card) for the span you are archiving,
   then push its cards in story order. One append run opens at least one
   chapter; a long span may deserve two or three.
4. **Close by updating the watermark**: re-push the `session` card with
   `watermark` set to now (ISO timestamp). The session card is the one
   card that updates; everything else is refused if the id exists.
5. `up` to open it for the user, and say the archive stays reopenable
   with the same slug.

## Levels — the altitude of the whole archive

Set once on the `session` card; appends inherit it. Rough budgets per
chapter:

| level | what survives | budget |
|---|---|---|
| `compact` | decisions and final artifacts only | ≤ 5 cards |
| `standard` | + key findings and turning points | ≤ 10 cards |
| `detailed` | + the narrative connective tissue | ≤ 20 cards |

Selectivity is the product. Five decision cards beat forty cards of
play-by-play; git history and the transcript already keep the rest.

## What you push

**`session`** — the singleton describing the archived session; the server
forces `id: "session"` and it is the only card that may be re-pushed:

```json
{"type":"session","payload":{"title":"...", "prompt":"<the user's original ask, verbatim>",
 "source":"claude-code terminal session", "level":"standard",
 "startedAt":"<ISO>", "watermark":"<ISO>"}}
```

`prompt` is the message that started the session, quoted verbatim — not
your paraphrase. It opens the story ("the ask"), and the whole archive is
read against it. Trim only what the user would trim (pasted logs, huge
snippets), and mark the cut with `[…]`.

**`chapter`** `{label, summary?}` — a divider; every card after it
belongs to it until the next one.

**Story cards** — payloads identical to Helix Canvas/Stream types:
`markdown` (narrative beats), `decision`, `code`, `diff`, `terminal`,
`test-report`, `table`, `diagram`, `image`, `chart`, `artifact`
(`{path}` — absolute path to a file the session produced). For
decisions, add what Canvas gets from the human loop: `outcome`
(`adopted` | `declined`) and `rationale` in the payload — archived
decisions were already made.

**`memory-candidate`** — a distilled fact in Helix Memory's shape,
awaiting the human's verdict:

```json
{"type":"memory-candidate","payload":{"name":"kebab-slug",
 "description":"one-line summary","factType":"project","markdown":"the fact"}}
```

Push one for anything worth keeping beyond this session — a lesson, a
constraint, a decision with lasting force. The user promotes it in the
browser; drain promote clicks with `wait`, and store each promoted fact
through `/helix-memory`. Don't write memories the user hasn't promoted.

**Append-only rules** — the properties that make it a record: never
re-push an existing card id (the server refuses with 409), no `delete`,
no `edge`, no `comment`. Made a mistake? Push a correction card; the
archive keeps both, like a ledger.

## The views

The browser renders four read-only views from the same log — story
(chapters in order), decisions (the register), artifacts, memory
(candidates with promote buttons). You never manage views; push good
cards and they populate themselves.

## Server operations

- A stale server restarts on the same port when you run `${CLAUDE_PLUGIN_ROOT}/bin/helix archive up`
  again; open tabs reconnect.
- **Kill by pid, never by name.** `pkill -f "helix archive serve"` matches every
  archive server on the machine — other projects, other people's sessions, runs
  in the middle of their work. It looks local and is not. Read the pid instead:
  `kill "$(sed -n 's/.*"pid": *\([0-9]*\).*/\1/p' ~/.helix/archive/<project>/<slug>/server.json)"`.
- The event log survives restarts; state is never in memory only. A server killed
  under a live session is survivable — the session sees exit 5 and `up` brings it
  back — but the work stops until someone notices.

## End of an append run

After `up`, arm one background wait so a promote click still reaches you:

```bash
# LAST tool call of the turn — Bash with run_in_background: true
${CLAUDE_PLUGIN_ROOT}/bin/helix archive wait --archive <slug> --dir <projectDir> --since <cursor> --timeout 3600 --wake-on user
```

When it wakes with promote responses, store those facts through
`/helix-memory` and confirm in the terminal. Exit 3 (quiet hour) or 4
(no viewer) end the matter — an archive owes nobody a vigil.
