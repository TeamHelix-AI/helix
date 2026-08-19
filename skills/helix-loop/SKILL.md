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

```
HL="${CLAUDE_PLUGIN_ROOT}/bin/helix loop"
```

```
$HL up   --loop <slug> [--dir <projectDir>] [--no-open]   # start server + board; auto-restarts stale servers on the same port
$HL push --loop <slug>                                     # loop EVENT JSON (single or array) on stdin
$HL wait --loop <slug> --since <cursor> [--timeout <s>]    # blocks for events; exit 3 timeout, 4 no-viewer, 5 down
```

Data: `~/.helix/loop/<project>/<slug>/` (event log + server.json) — one home
for the whole machine, never inside a repo. The board
is NOT a canvas — it is Helix Loop's own UI: bar pinned top, a lane per
piece (click for attempt history), budget meter, gate banners, agent row.

## Event vocabulary (what you push)

```
loop.init        {name, budget:{maxAttemptsPerPiece, maxTokens}}
audit            {agent, pieceId?, kind:"text"|"code"|"screenshot", text?, code?, language?, path?, label?}
metrics          {agent?, pieceId?, model?, tokens?, toolCalls?, durationS?}   # cumulative deltas; feeds the Stats tab
bar.proposed     {version, criteria:[{id, label, detail?, checks?:[string], evidence?}]}
piece.created    {pieceId, title, deps?:[pieceId]}      # re-push merges (updates title/deps, keeps attempts)
attempt.started  {pieceId, n, agent}
attempt.finished {pieceId, n, agent, summary?}
verdict.added    {pieceId, n, agent, criteria:{<id>:"pass"|"fail"}, notes?}
piece.passed     {pieceId}
budget.spent     {tokens}                               # cumulative deltas, best honest estimate
gate.raised      {gateId, kind:"budget", detail}
agent.status     {agent, text, pieceId?}                # include pieceId whenever the work is piece-scoped — it feeds the piece drawer's activity
```

User actions arrive in your `wait` delta with `origin:"user"`:
`bar.ratified`, `bar.declined`, `gate.resolved {action:"continue"|"stop"|"adjust"}`,
`note.added {text, pieceId?, critId?, bar?}` — `critId` targets one
criterion, `bar: true` targets the whole bar (e.g. "add a criterion for
X"). Bar/criterion notes are revision requests: answer them with the next
`bar.proposed` version (never edit a ratified bar in place); a note
arriving mid-run steers the next attempt or the next bar version.

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
   may start label-only. The board renders definitions as popovers on the
   criteria chips, marks undefined criteria dashed, and warns next to
   Ratify — ratification means the human approved the *definitions*, so
   proposing an under-defined bar on a serious loop is pushing the human
   to sign a blank check. Push `bar.proposed`. **Do not start work.**
3. **Wait for `bar.ratified`.** On `bar.declined`, ask why (terminal or a
   note), revise, propose the next version. The bar NEVER changes without
   a ratified new version — this is the anti-drift gate; there is no
   emergency exception. **Mid-run re-ratification** (the user asked for a
   revision after work started): the moment the new version is ratified,
   all future verdicts and the final integration judge against it.
   Pieces that already passed keep their pass — it is recorded against
   the version they cleared — but the lead must review them against the
   changed criteria and explicitly reopen (new attempt) any piece a new
   or tightened criterion invalidates, saying so in `agent.status`.
   Never silently re-grade history, and never keep judging against a
   superseded version.
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
   (loop slug, port, bar version, budget spent, outcome).

## Cursor discipline (same law as helix-canvas)

Wait from the cursor of your last drain or a fresh `/api/state` fetch —
and before waiting on something specific (a ratification, a gate), check
`/api/state` first: the answer may already be in the log. Waiting-for-the-
past and skipping-the-past are both real failure modes.

## Honesty rules

- Never ratify, resolve, or simulate a user action yourself — user events
  come only from the board.
- Never report a verdict the critic didn't produce, or a pass without
  every criterion cited.
- If the loop cannot meet the bar within budget, that is a finding, not a
  failure to hide: raise the gate and say so.
