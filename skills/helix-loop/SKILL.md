---
name: loop
description: Run a visual, human-gated gauntlet loop on the Helix Loop board — a lead splits work against a ratified quality bar, builders and critics iterate via the Workflow engine, and the human ratifies the bar and resolves budget gates on the board. Use when the user says "/helix-loop", "run a gauntlet", "start a loop", or wants iterative multi-agent building with a visible quality bar. Sibling of helix-canvas (different app, different server).
allowed-tools: Bash(${CLAUDE_PLUGIN_ROOT}/bin/helix *)
---

# Helix Loop

A gauntlet loop the human can see and steer. You orchestrate; the board
shows every piece live; the bar and the budget belong to the human. The
loop event log is the record. "You are the brake" is architecture here —
never bypass a gate.

## Setup

```sh
# start server + board; auto-restarts stale servers on the same port
${CLAUDE_PLUGIN_ROOT}/bin/helix loop up   --loop <slug> --agent <name> --model <model-id> [--dir <projectDir>] [--no-open]
#   --agent: your product name, lowercase (claude, codex, …)
#   --model: your exact model id (e.g. claude-fable-5) — Fleet shows and filters by these

# the bar: plan JSON {changed, reason, sections} on stdin -> a revision awaiting Ratify
${CLAUDE_PLUGIN_ROOT}/bin/helix loop plan --loop <slug>

# loop EVENT JSON (single or array) on stdin; exit 6 = the bar is not ratified yet
${CLAUDE_PLUGIN_ROOT}/bin/helix loop push --loop <slug>

# blocks for events; exit 3 timeout, 4 no-viewer, 5 down
# --wake-on user holds through your own pushes and viewer churn and returns on a human act
${CLAUDE_PLUGIN_ROOT}/bin/helix loop wait --loop <slug> --since <cursor> [--timeout <s>] [--wake-on user]
```

- `up` opens the browser itself, from inside the binary. `--no-open` is for
  headless or scripted runs — never a way around a permission prompt. If a
  prompt is declined, hand the user `! open <url>` and carry on.
- `up` also brings up Fleet (mission control) when it is down, and returns
  `fleet: {url, started}`. Hand the user both links in one line —
  `Board: <url> · Fleet: <fleet.url>` — so they know the overview is there.
  `up` never opens Fleet in the browser; the user follows the link.

Data: `~/.helix/loop/<project>/<slug>/` (event log + server.json) — one home
for the whole machine, never inside a repo. The board
is NOT a canvas — it is Helix Loop's own UI: bar pinned top, a lane per
piece (click for attempt history), budget meter, gate banners, agent row.

## Rails first

Before proposing anything, read the rails that bind this project on this surface:

```sh
${CLAUDE_PLUGIN_ROOT}/bin/helix rails for --surface loop
```

What comes back is not advice — it is the standard the work is held to. Cite
rail ids when you act on them; full text with `helix rails get <id>`. Details
in the helix-rails skill.

## Event vocabulary (what you push)

Every event on stdin is `{"type": "<name>", "payload": {…}}` — one object
or an array of them. The fields listed below are the payload.

```
loop.init        {name, budget:{maxAttemptsPerPiece, maxTokens}}   # defaults only; the plan's budget section is what counts
audit            {agent, pieceId?, kind:"text"|"code"|"screenshot", text?, code?, language?, path?, label?}
metrics          {agent?, pieceId?, model?, tokens?, toolCalls?, durationS?}   # cumulative deltas; feeds the Stats tab
piece.created    {pieceId, title, deps?:[pieceId]}      # re-push merges (updates title/deps, keeps attempts)
attempt.started  {pieceId, n, agent}
attempt.finished {pieceId, n, agent, summary?}
verdict.added    {pieceId, n, agent, criteria:{<id>:"pass"|"fail"}, notes?}
piece.passed     {pieceId}
budget.spent     {tokens}                               # cumulative deltas, best honest estimate
gate.raised      {gateId, kind:"budget", detail}
agent.status     {agent, text, pieceId?}                # include pieceId whenever the work is piece-scoped — it feeds the piece drawer's activity
```

**The server refuses every work event until the bar is ratified** — pieces,
attempts, verdicts, gates, evidence all come back `409 not-ratified` (exit 6).
Only `loop.init`, `agent.status`, `metrics` and `budget.spent` land before.

### The bar is the plan

The bar goes through `helix loop plan`, never through `push`. It is the same
plan every Helix run carries (Arena and Swarm use it too), in three sections:

```
criteria   {items:[{id, label, detail?, checks?:[string], evidence?}]}   # required
budget     {maxTokens, maxAttemptsPerPiece}                                # defaults apply if absent, and the board says so
models     {roles:{lead, builder, critic}, available?:[…]}                # every role inherits the session model if absent
```

```sh
printf '%s' '{"changed":["criteria","budget","models"],
  "reason":"pre-flight",
  "sections":{"criteria":{"items":[…]},"budget":{"maxTokens":500000,"maxAttemptsPerPiece":5},"models":{"roles":{…}}}}' \
  | ${CLAUDE_PLUGIN_ROOT}/bin/helix loop plan --loop <slug>
```

The person can **edit every section on the board** before ratifying — add,
reword, reorder or remove criteria, change the budget, swap models. An edit
supersedes your proposal and becomes the next revision, awaiting its own
Ratify. So always run from the ratified revision's sections (`/api/state`
`.plan.sections`), never from what you proposed.

User actions arrive in your `wait` delta with `origin:"user"`:
`plan.ratified {revision}`, `plan.declined {revision, why}`, a user-origin
`plan.proposed` (an edit — keep waiting), `gate.resolved {action:"continue"|"stop"|"adjust"}`,
`note.added {text, pieceId?, critId?, bar?}` — `critId` targets one
criterion, `bar: true` targets the whole bar. Bar notes are revision
requests: answer them with the next `plan` revision (never edit a ratified
bar in place); a note arriving mid-run steers the next attempt.

## The loop, step by step

1. **Open**: `up`, then `loop.init` with the budget (defaults ratified:
   5 attempts/piece, 500k tokens/run — honor user overrides).
2. **Propose the bar**: decompose the user's quality target into criteria
   with short ids, **each fully defined**: `detail` (what this criterion
   means for THIS loop), `checks` (concrete, verifiable definition-of-pass
   items — the things a critic can actually test), and `evidence` (what
   the critic must produce to claim a verdict). This matters most on heavy
   loops (app builds, code conversion, legacy modernization — where
   criteria carry non-functional requirements); only a trivial quick loop
   may start label-only. The board lists the criteria, marks undefined
   ones, and warns next to Ratify — ratification means the human approved
   the *definitions*, so proposing an under-defined bar on a serious loop
   is pushing the human to sign a blank check. Propose with
   `helix loop plan` — criteria, budget and models in one revision.
   **Do not start work**; the server will not let you anyway.
3. **Wait for `plan.ratified`** with `wait … --wake-on user` (see *Cursor
   discipline*), then read the ratified revision's sections and run from
   those — the person may have edited them. On `plan.declined`, read the
   why, revise, propose the next revision. The bar NEVER changes without
   a ratified new revision — this is the anti-drift gate; there is no
   emergency exception. **Mid-run re-ratification** (the user asked for a
   revision after work started): the moment the new revision is ratified,
   all future verdicts and the final integration judge against it.
   Pieces that already passed keep their pass — it is recorded against
   the revision they cleared — but the lead must review them against the
   changed criteria and explicitly reopen (new attempt) any piece a new
   or tightened criterion invalidates, saying so in `agent.status`.
   Never silently re-grade history, and never keep judging against a
   superseded revision.
4. **Split**: `piece.created` per independently judgeable piece — **as a
   DAG**: declare `deps` where one piece genuinely cannot be built or
   judged before another exists (design tokens before components, schema
   before migration). Deps are judgeability constraints, never
   preferences: every edge is wall-clock serialization, so justify each
   one in the piece's creation status. The board derives a `waiting`
   state for unmet deps and draws the dependency rail; no deps = flat
   parallel gauntlet, which stays the default.
5. **Iterate** — this is where the Workflow engine does the fan-out
   (the user opted into multi-agent by invoking this skill):
   - **MAXIMUM CONCURRENCY WITHIN THE DAG — this is a law, not a
     preference.** Every piece launches the moment its deps have passed;
     nothing ever waits on a piece it does not depend on. The first
     proving run serialized pieces and turned ~90 minutes of work into
     6+ hours of wall clock. The clean Workflow shape is a memoized
     promise per piece:
     ```js
     const done = {};
     const run = (p) => done[p.id] ??= (async () => {
       await Promise.all((p.deps ?? []).map((d) => run(byId[d])));
       // build → judge → retry loop for this piece, pushing events as it goes
     })();
     await Promise.all(pieces.map(run));
     ```
     The only global serial points are the ratified bar (before
     everything) and final integration (after everything).
   - per piece: `attempt.started` → builder agent produces the real
     artifact in the repo → `attempt.finished` → critic agent (fresh
     context, sees only artifact + bar) → `verdict.added` citing every
     criterion id → all pass → `piece.passed`; any fail → next attempt
     with the critic's notes.
   - critics judge the REAL output (render it, screenshot it, read it) —
     never the builder's description of it.
   - **Critique can be a panel.** When a piece's criteria span distinct
     concerns (performance trace vs accessibility vs responsive vs visual
     craft), split the critique across parallel single-concern critics:
     each gets only the artifact plus its criteria subset with their
     `checks` and `evidence` definitions, runs concurrently, and pushes
     its own `verdict.added` citing only the criteria it judged. The
     board merges the union; the piece passes only when every criterion
     is covered and passing. Keep one whole-piece coherence check (a
     generalist critic or the integration pass) — specialists can each
     pass their lane while the piece is incoherent as a whole.
   - **Subagents report for themselves, frequently.** Give every builder
     and critic the CLI line and require, via Bash, at each step:
     `agent.status` on start, after each major step, and on finish; plus
     `audit` events for evidence — the builder pushes a screenshot
     (`kind:"screenshot"`, absolute `path`) after rendering and ONE key
     code block (≤40 lines); the critic pushes the screenshot(s) it judged
     and one `text` audit per verdict (≤300 chars). Not a running
     commentary — one entry per step, evidence over narration. The board's
     Audit tab and per-piece drawer render these.
   - keep `agent.status` truthful; report `budget.spent` deltas as each
     agent completes (workflow token telemetry or honest estimates).
   - **Telemetry**: each builder/critic pushes ONE `metrics` event per
     attempt on finish — `{agent, pieceId, model, tokens, toolCalls,
     durationS}` (deltas; honest estimates when exact numbers are
     unavailable, model = the actual model id it ran on). The lead pushes
     loop-level metrics for its own work. This feeds the board's Stats tab
     (overall tiles + per-agent and per-piece tables); `budget.spent`
     remains the authoritative ledger for the gate.
6. **Gates**: crossing max attempts on a piece or the token budget →
   `gate.raised` + pause that work + `wait` for `gate.resolved`.
   `continue` resumes; `stop` ends the run cleanly (report state);
   `adjust` means read the detail/notes for new limits.
7. **Drain continuously**: user `note.added` during the run is steering —
   acknowledge it in `agent.status` and apply it to the next attempt.
8. **Finish**: all pieces passed → assemble the final artifact, push a
   final `agent.status`, summarize in the terminal, update memory
   (loop slug, port, plan revision, budget spent, outcome). A finished
   loop's plan is a record: the board stops offering edits once every
   piece has passed.

## Server operations

- A stale server restarts on the same port when you run `${CLAUDE_PLUGIN_ROOT}/bin/helix loop up`
  again; open tabs reconnect.
- **Kill by pid, never by name.** `pkill -f "helix loop serve"` matches every
  loop server on the machine — other projects, other people's sessions, runs
  in the middle of their work. It looks local and is not. Read the pid instead:
  `kill "$(sed -n 's/.*"pid": *\([0-9]*\).*/\1/p' ~/.helix/loop/<project>/<slug>/server.json)"`.
- The event log survives restarts; state is never in memory only. A server killed
  under a live session is survivable — the session sees exit 5 and `up` brings it
  back — but the work stops until someone notices.

## Cursor discipline (same law as helix-canvas)

Wait from the cursor of your last drain or a fresh `/api/state` fetch —
and before waiting on something specific (a ratification, a gate), check
`/api/state` first: the answer may already be in the log. Waiting-for-the-
past and skipping-the-past are both real failure modes.

When you wait **for a person** — `plan.ratified`, `gate.resolved` — always
pass `--wake-on user`:

```sh
${CLAUDE_PLUGIN_ROOT}/bin/helix loop wait --loop <slug> --since <cursor> --timeout 600 --wake-on user
```

Without it the first delta of any origin returns — your own `piece.created`,
a viewer opening the tab — and a lead that reads "not ratified yet" from
such a delta and stops waiting has missed the ratification that comes next.
Exit 3 means nothing yet: wait again from the returned cursor. Exit 5 means
the server is down: `up`, then wait again. When draining steering between
attempts, a short plain `wait` (60–120s) is right; it should return on
anything.

```sh
PORT=$(sed -n 's/.*"port": *\([0-9]*\).*/\1/p' ~/.helix/loop/<project>/<slug>/server.json)
curl -s "http://127.0.0.1:$PORT/api/state"   # .plan.status, .ready, .gates, .notes — check before you wait
```

## Honesty rules

- Never ratify, resolve, or simulate a user action yourself — user events
  come only from the board.
- Never report a verdict the critic didn't produce, or a pass without
  every criterion cited.
- If the loop cannot meet the bar within budget, that is a finding, not a
  failure to hide: raise the gate and say so.
