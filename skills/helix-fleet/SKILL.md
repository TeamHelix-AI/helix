---
name: fleet
description: Open or query Helix Fleet — mission control for Team Helix. One board showing every Canvas/Loop/Stream session across projects, a unified "needs you" attention queue, and embedded session tabs. Use when the user says "/helix-fleet", "open fleet", "mission control", "what's running", "what needs me", or wants an overview of all Helix sessions. Agents never drive Fleet — sessions register themselves; this skill is about starting the board and reading it.
allowed-tools: Bash(${CLAUDE_PLUGIN_ROOT}/bin/helix *)
---

# Helix Fleet

Mission control. Fleet is a **global singleton** (port 4890) that Canvas,
Loop, and Stream servers register with automatically on start; they push
their needs-you items on change, and Fleet health-pings them for
liveness. Agents never push to Fleet — the only agent verbs are `up` and
`status`.

## CLI

```
HF="${CLAUDE_PLUGIN_ROOT}/bin/helix fleet"

$HF up [--no-open]   # start/reuse THE fleet server (port 4890), open the board
$HF status           # registry as JSON: {sessions:[...], attentionTotal}
```

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
3. Sessions started while Fleet was down appear on their next server
   (re)start — if the user says a session is missing, rerun that app's
   `up` (same slug/port; open tabs reconnect).

## When the user asks "what needs me?"

Run `$HF status`, and summarize the attention items grouped by session —
don't make them open the board if a sentence answers it.
