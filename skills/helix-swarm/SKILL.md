---
name: swarm
description: Run a coverage sweep on the Helix Swarm board — one job cut into disjoint slices, one worker each, every worker declaring how much of its slice it actually read, then merged into one report that names its own gaps. Use when the user says "/helix-swarm", "swarm this", "sweep the codebase", or a job is too wide for one context. Sibling of helix-arena (candidates judged) and helix-loop (build to a bar).
allowed-tools: Bash(${CLAUDE_PLUGIN_ROOT}/bin/helix *)
---

# Helix Swarm

One job, cut into slices nobody else touches, merged into one report. You
orchestrate; the board shows coverage first and every slice beside it; the
plan and the gates belong to the human.

**Coverage is the load-bearing idea.** A worker that read 96 of 1,390 files
and says "pass" produces a line indistinguishable from one that read
everything. The server rejects a report without `read` and `total` for
exactly that reason, and the board draws thin coverage in amber. An unstated
sample is how a survey becomes a claim nobody made.

## Setup

```sh
${CLAUDE_PLUGIN_ROOT}/bin/helix swarm up   --swarm <slug> --agent <name> --model <model-id> [--dir <projectDir>] [--no-open]
#   --agent: your product name, lowercase (claude, codex, …)
#   --model: your exact model id (e.g. claude-fable-5) — Fleet shows and filters by these
${CLAUDE_PLUGIN_ROOT}/bin/helix swarm plan --swarm <slug>   # plan JSON on stdin
${CLAUDE_PLUGIN_ROOT}/bin/helix swarm push --swarm <slug>   # event JSON (single or array) on stdin
${CLAUDE_PLUGIN_ROOT}/bin/helix swarm wait --swarm <slug> --since <cursor> [--timeout <s>] [--wake-on user]
```

Data: `~/.helix/swarm/<project>/<slug>/`.

Exit codes: 0 delta, 2 invalid, 3 timeout, 4 no-viewer, 5 down, **6 the plan
is not ratified**.

## Rails first

Before proposing anything, read the rails that bind this project on this surface:

```sh
${CLAUDE_PLUGIN_ROOT}/bin/helix rails for --surface swarm
```

What comes back is not advice — it is the standard the work is held to. Cite
rail ids when you act on them; full text with `helix rails get <id>`. Details
in the helix-rails skill.

## Narrow before you fan out

**The unit of work is the whole decision.** Ten thousand files is not ten
thousand workers — briefing, waiting for and filing an agent costs more than
reading one file, and that overhead is paid per worker. Before proposing the
plan:

1. Cut the tree down with a **script**, not agents. Counting files, graphing
   references and dating them from git is mechanical work a script does
   exactly and for nothing. Commit it: the census can then be re-run after
   each change and compared against this baseline.
2. Group what is left into slices a worker can actually hold — by seam
   (subsystem, bounded context, ownership), never by file.
3. Only then propose the fan-out.

A slice that needs something another slice holds is not a slice. If they
overlap, the partition is wrong.

## The run, step by step

1. **Open**: `up`.
2. **Propose the plan.** Two sections cannot be blank:
   - `fanout` — the unit, the slice list, how many at once. The cost driver.
   - `coverage` — whether sampling is allowed and what every worker must
     declare. The honest default is full coverage required.
   Plus `criteria` (what a slice report must establish), `budget` and
   `models`. Say in the reason why the job does not fit one context —
   if it does, a swarm is overhead plus a merge step.
3. **Wait for the ratification** — `wait … --wake-on user`, as in *Waiting
   on the board* below — then read the ratified revision's sections and run
   from those. Nothing spawns before it; the server refuses.
4. **Declare the selection rule in the plan and honour it.** For a coverage
   sweep it is always the same: **every slice must return.** There is no
   ranking and no best-of — a missing slice is a hole in the map, not a loser.
5. **Fan out**, one worker per slice, in one message. Each brief stands
   alone: the goal, its exact slice, how to verify, what to report, and the
   requirement to state coverage.
6. **Workers report**: `pass`, `issues` or `blocked`, with `read`, `total`,
   and findings that carry `evidence` and a `basis` of `counted` or
   `estimated`. A number extrapolated from a sample is marked `estimated`;
   the board renders those differently, because a plan built on an
   extrapolation is a different thing from one built on a count.
7. **A worker that drops out** gets `slice.dropped` with a reason. Proceed
   with the rest and say so. Its units stay in the denominator — a blocked
   worker must never make coverage look better.
8. **Merge.** `merge.recorded` with the report, the coverage as merged, the
   gaps, and the dropouts named. Do not paste raw worker output; each finding
   is one line with the evidence that proves it, ranked by what it costs to
   leave.
9. **Report** in chat: the coverage line first, findings ranked, what was not
   covered, and the next move. If the sweep found the same shape of problem
   three times, the recommendation is a lint, not a habit — encoding it is
   what stops the sweep being needed again.

## Plan sections — the shape the board reads

Every section carries `version` and the fields below. `budget.maxTokens`
and `budget.maxWallClockS` become the live ceilings on the meter; a budget
sent under another name is a budget nobody enforces.

`models.roles.worker` is required. `models.available` is the catalogue the
board offers when a person edits the plan: every model id you can actually
run, with a label where the id is not self-explanatory. Every section is
editable on the board; an edit becomes the next revision and waits for its
own Ratify.

```
criteria:  {items:[{id, label, detail, checks, evidence, deleted?}]}   # deleted:true = retired on the board; judges skip it
budget:    {maxTokens:400000, maxWallClockS:900, maxAttempts?}
fanout:    {workers:5, unit:"slice", maxConcurrent:3, slices:["api","web",…]}
models:    {roles:{worker:{model:"claude-sonnet-5"}, lead:{model:"claude-fable-5"}},
            available:[{id:"claude-sonnet-5",label:"Sonnet 5"}, {id:"claude-fable-5"}, …]}
coverage:  {samplingAllowed:false, declare:["read","total"]}
```

## Waiting on the board

Every user act — Ratify, Decline, an edit, a gate, a note — is an event with
`origin:"user"`. You see it only by waiting from the right cursor with the
right flag. This is where runs go wrong: a lead that waits without
`--wake-on user` wakes on its own pushes and viewer churn, reads "no
ratification yet", and either loops forever or gives up.

1. **Cursor.** Every `plan` and `push` reply carries `cursor`; wait from the
   last one you saw. Before waiting for something specific, read the state —
   the answer may already be in the log:
   ```sh
   PORT=$(sed -n 's/.*"port": *\([0-9]*\).*/\1/p' ~/.helix/swarm/<project>/<slug>/server.json)
   curl -s "http://127.0.0.1:$PORT/api/state"           # .plan, .revision, .ready, .gates, .notes
   curl -s "http://127.0.0.1:$PORT/api/plan?revision=N"  # one revision's exact sections
   ```
2. **Always `--wake-on user`.**
   ```sh
   ${CLAUDE_PLUGIN_ROOT}/bin/helix swarm wait --swarm <slug> --since <cursor> --timeout 600 --wake-on user
   ```
   Exit 0: the delta holds a human act — read it. Exit 3: nothing yet;
   wait again from the returned cursor (and say so in chat every few rounds).
   Exit 5: the server is down; `up`, then wait again. Exit 4 only happens
   without `--wake-on user`.
3. **What arrives, and what each means.**
   - `plan.ratified {revision}` — the gate is open. **Read the ratified
     revision and run from its sections, never from what you proposed**: the
     person may have changed models, fan-out, criteria or the selection rule
     on the board before ratifying.
   - `plan.declined {revision, why}` — read `why`, revise, propose again.
   - `plan.superseded` + a user-origin `plan.proposed` — the person edited
     the plan; it is a new revision awaiting Ratify. Keep waiting.
   - `note.added {text}` — steering. Read it, act on it, acknowledge with
     `agent.status`.
   - `gate.resolved {gateId, action}` — `continue` or `stop`; honour it.
4. **Mid-run edits.** A user-origin `plan.proposed` while work is running
   means a change is coming: finish in-flight units, start nothing new until
   it is ratified or declined. Once ratified, every later event cites the new
   revision, and the work that follows runs under its sections.
5. **Drain while working.** Between fan-out and merge, wait with a short
   timeout (60–120s) each time you would otherwise idle, so a note or a gate
   reaches you before the next expensive step, not after.

## Events

Every event on stdin is `{"type": "<name>", "payload": {…}}` — one object
or an array of them. The fields listed below are the payload.

```
slice.created   {sliceId, label, scope, unitCount, revision}
slice.started   {sliceId, worker, model}
slice.reported  {sliceId, verdict:"pass"|"issues"|"blocked",
                 read, total,                    # REQUIRED — no exceptions
                 findings:[{summary, evidence, basis:"counted"|"estimated"}]}
slice.dropped   {sliceId, why}
merge.recorded  {report, coverage:{read,total}, gaps:[], dropouts:[], revision}
```

Plus the shared kit: `run.init`, `agent.status`, `audit`, `metrics`,
`budget.spent`, `gate.raised`.

`audit` is a read-only evidence card. Pick the kind that fits what you have —
the board renders each properly, and text is markdown (GFM tables work):

```
audit {agent, unitId?, label?, kind:"markdown"|"text", text}
audit {agent, unitId?, label?, kind:"code", code, lang?, text?}
audit {agent, unitId?, label?, kind:"table", columns:[…], rows:[[…]|{…}], text?}
audit {agent, unitId?, label?, kind:"terminal", command?, output}
audit {agent, unitId?, label?, kind:"diff", diff}
audit {agent, unitId?, label?, kind:"screenshot"|"image", path, caption?}   # absolute path
```

## Workers report for themselves

**A slice verdict is a claim; the audit trail is what makes it checkable.**
Give every worker the CLI line and require, via Bash:

- `agent.status` on start and on finish — `{agent, text, unitId: "<sliceId>"}`.
  Always include `unitId`, or the board cannot attribute the work to a slice.
- **One `audit` per finding**, carrying the evidence rather than describing
  it: `kind:"code"` with the offending lines (≤40), or `kind:"text"` with the
  `file:line` and what it shows (≤300 chars). A finding whose evidence never
  reaches the board is a claim the reader has to take on trust.
- **One `metrics` event on finish** — `{agent, unitId, model, tokens,
  toolCalls, durationS}`, with the real model id and honest estimates where
  exact numbers are unavailable. This is what fills the Stats tab; without it
  a run that cost real money reports zero.
- The lead pushes `budget.spent` deltas as each worker completes, and its own
  `metrics` for the merge.

Evidence over narration: one entry per finding, not a commentary. A run whose
Evidence and Stats tabs are empty did not report — it did not do less work.

## Server operations

- A stale server restarts on the same port when you run `${CLAUDE_PLUGIN_ROOT}/bin/helix swarm up`
  again; open tabs reconnect.
- **Kill by pid, never by name.** `pkill -f "helix swarm serve"` matches every
  swarm server on the machine — other projects, other people's sessions, runs
  in the middle of their work. It looks local and is not. Read the pid instead:
  `kill "$(sed -n 's/.*"pid": *\([0-9]*\).*/\1/p' ~/.helix/swarm/<project>/<slug>/server.json)"`.
- The event log survives restarts; state is never in memory only. A server killed
  under a live session is survivable — the session sees exit 5 and `up` brings it
  back — but the work stops until someone notices.

## Honesty rules

- Never ratify or resolve a gate yourself.
- Never report coverage a worker did not declare.
- Never quietly drop an unreported slice from the denominator.
- A sweep that read 15% of the tree says 15%, in the first line, before any
  finding. It is a reasonable basis for an order of work and a poor basis for
  an estimate of effort — say which you are giving.
