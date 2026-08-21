---
name: fleet
description: Open or query Helix Fleet — mission control for Team Helix. One board showing every Canvas/Loop/Stream session across projects, a unified "needs you" attention queue, and embedded session tabs. Use when the user says "/helix-fleet", "open fleet", "mission control", "what's running", "what needs me", or wants an overview of all Helix sessions. Agents never drive Fleet — sessions register themselves; this skill is about starting the board and reading it.
allowed-tools: Bash(${CLAUDE_PLUGIN_ROOT}/bin/helix *)
---

# Helix Fleet

Mission control. Fleet is a **global singleton** (port 4890). Every
session `up` (spark, canvas, loop, arena, swarm, stream, archive) starts
it when it is down and returns `fleet: {url, started}`, so the board is
there whenever a session is; servers register with it on start, push
their needs-you items on change, and Fleet health-pings them for
liveness. Agents never push to Fleet — the only agent verbs are `up` and
`status`.

## CLI

```sh
# start/reuse THE fleet server (port 4890), open the board
${CLAUDE_PLUGIN_ROOT}/bin/helix fleet up [--no-open]

# registry as JSON: {sessions:[...], attentionTotal}
${CLAUDE_PLUGIN_ROOT}/bin/helix fleet status
```

- `up` opens the browser itself, from inside the binary. `--no-open` is for
  headless or scripted runs — never a way around a permission prompt. If a
  prompt is declined, hand the user `! open <url>` and carry on.
- Data: `~/.helix/fleet/` (registry.json + fleet.db). Port override:
  `HELIX_FLEET_PORT` env (must match in Fleet AND the app servers).
- `status` is the scriptable view — use it to answer "what's running /
  what needs the user" without opening anything. Session fields: `app`,
  `slug`, `projectDir`, `port`, `alive`, `title`,
  `attention:[{id,kind,label,since}]`, `viewers`, `startedAt`.
- Stale-server rule as siblings: `up` restarts an outdated fleet server
  on the same port.

## What the board shows (so you can point the user at it)

- **needs you** queue (left): every unanswered question/decision, open
  gate, and unratified bar across all sessions, newest first — click
  jumps into the session.
- Session rows grouped by project: attention badge, app chip, slug,
  title, live/dead, uptime, viewers. Click a row → the session opens as
  an embedded tab inside Fleet; ⧉ opens its native tab; ✕ prunes dead
  rows.

## The agent's obligations (this is the whole "integration")

1. Sessions register themselves — you do nothing to appear on the board.
2. **Keep your session's title honest** — Fleet displays it:
   - canvas → your latest `status` pill text
   - stream → your `status` tile text
   - loop → the `loop.init` name
   Stale status = a lying Fleet row.
3. A session `up` starts Fleet if needed, so rows rarely go missing. If
   the user says one is, rerun that app's `up` (same slug/port; open tabs
   reconnect) — it re-registers.

## When the user asks "what needs me?"

Run `${CLAUDE_PLUGIN_ROOT}/bin/helix fleet status`, and summarize the attention items grouped by session —
don't make them open the board if a sentence answers it.
