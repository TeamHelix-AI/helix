---
name: arena
description: Run a judged bakeoff on the Helix Arena board — N candidates answer the same question in their own workspaces, a judge scores them against a ratified rubric, one becomes the base, and the best parts of the losers are grafted in with the rejections recorded. Use when the user says "/helix-arena", "run an arena", "bake this off", or faces a one-way door where one attempt would lock in the wrong shape. Sibling of helix-swarm (slices merged) and helix-loop (build to a bar).
allowed-tools: Bash(${CLAUDE_PLUGIN_ROOT}/bin/helix *)
---

# Helix Arena

One question, answered N times, judged against a rubric the human ratified
before anything spawned. You orchestrate; the board shows every candidate
live; the plan and the gates belong to the human.

**The arena is expensive on purpose.** Four runners plus a judge for one
decision only pays when the decision is a one-way door — a shape every
downstream thing inherits and nobody can revise cheaply. Running one over a
settled shape is an expensive way to agree with yourself. Say so and use
`/helix-loop` or plain work instead.

## Setup

```sh
${CLAUDE_PLUGIN_ROOT}/bin/helix arena up   --arena <slug> --agent <name> --model <model-id> [--dir <projectDir>] [--no-open]
#   --agent: your product name, lowercase (claude, codex, …)
#   --model: your exact model id (e.g. claude-fable-5) — Fleet shows and filters by these
${CLAUDE_PLUGIN_ROOT}/bin/helix arena plan --arena <slug>   # plan JSON on stdin
${CLAUDE_PLUGIN_ROOT}/bin/helix arena push --arena <slug>   # event JSON (single or array) on stdin
${CLAUDE_PLUGIN_ROOT}/bin/helix arena wait --arena <slug> --since <cursor> [--timeout <s>] [--wake-on user]
```

Data: `~/.helix/arena/<project>/<slug>/` — one home for the whole machine,
never inside a repo.

- `up` also brings up Fleet (mission control) when it is down, and returns
  `fleet: {url, started}`. Hand the user both links in one line —
  `Board: <url> · Fleet: <fleet.url>` — so they know the overview is there.
  `up` never opens Fleet in the browser; the user follows the link.

Exit codes: 0 delta, 2 invalid, 3 timeout, 4 no-viewer, 5 down, **6 the plan
is not ratified**.

## Rails first

Before proposing anything, read the rails that bind this project on this surface:

```sh
${CLAUDE_PLUGIN_ROOT}/bin/helix rails for --surface arena
```

What comes back is not advice — it is the standard the work is held to. Cite
rail ids when you act on them; full text with `helix rails get <id>`. Details
in the helix-rails skill.

## The gate is real

The server refuses every work event until revision 1 of the plan is ratified
on the board. Exit 6 is not a hint you may work around — it means a person
has not yet agreed to what this will cost. There is no override.

## The run, step by step

1. **Open**: `up`. Nothing else yet.
2. **Propose the plan.** One object, one Ratify, six sections. Two of them
   cannot be blank:
   - `criteria` — 3 to 6 concrete, gradeable criteria derived from what
     success means for *this* task. Each carries `detail` (what it means
     here), `checks` (what a judge can actually test) and `evidence` (what
     the judge must produce). `Adds a --dry-run flag that skips writes` is a
     criterion; `code is correct` is not. **The candidates never see the
     rubric** — it is the picker's tool.
   - `fanout` — how many candidates, on what, how many at once. This is the
     cost driver.
   - `models` — one runner per entry. Prefer different model families:
     the adversarial signal comes from diverse blind spots, not from assigned
     personas. Same model N times only when the work is generation-bound.
   - `selection` — `first-pass`, `rank-all` or `best-of`, **declared now**.
     Choosing after the answers are in is the failure this prevents.
   - `budget`, and that is the plan.

   ```sh
   printf '%s' '{"changed":["criteria","budget","fanout","models","selection"],
     "reason":"one-way door: every context inherits this shape",
     "sections":{ ... }}' | helix arena plan --arena <slug>
   ```
3. **Wait for the ratification** — `wait … --wake-on user`, as in *Waiting
   on the board* below — then read the ratified revision's sections and run
   from those. On a decline, read the why, revise, propose
   the next revision. Sections version independently — a budget bump
   re-ratifies budget alone — and the plan's revision increments on any
   change, which is what every result cites.
4. **Fan out.** One `candidate.created` per runner, each with its own
   workspace (a git worktree where possible). N candidates writing to one
   path is the shared mutable state an arena exists to avoid.
   Spawn them in one message, all in parallel.
5. **Every candidate returns a rationale.** `candidate.finished` without one
   is rejected by the server, and rightly: without it there is no way to tell
   a principled shape from an accidental one, and the graft becomes guesswork.
   The rationale names what the candidate considered and rejected.
6. **Judge.** One read-only judge on a different model family from yours,
   scoring each criterion for each candidate. It sees the rubric and the
   candidates by path label. Run it alongside your own reading, never
   alongside the candidates — a judge spawned while they are still writing
   sees half-written files and reports them as dropouts.
7. **Pick the base.** Read every candidate end to end first; skimming
   surfaces the one whose style looks most familiar. Score criterion by
   criterion, then compare with the judge. `base.picked` requires a `why`.
   - Agreement confirms the pick.
   - Disagreement means one of you is biased or a criterion is ambiguous.
     Read both rationales. A clean disagreement usually means the rubric is
     missing something — say so, because that is the more valuable finding.
8. **Graft.** Walk each loser once more for the one or two things worth
   porting. Fold them in by hand; pasting produces something no single mental
   model holds. Push `graft.recorded` for each, and `reject.recorded` for
   what stayed out **with its reason** — the rejections are the half a future
   reader learns from.
9. **Two special outcomes**, and they lead opposite ways:
   - All candidates converge → `converged.noted`. Ship the consensus, skip
     the graft, and record the agreement.
   - Candidates wildly diverge → `diverged.noted`. The framing was too loose.
     Reframe and re-run; averaging a spread produces a design none of them
     would defend.
10. **Verify.** The synthesized result faces the same scrutiny as anything
    else — running an arena does not earn a pass. A fact you cannot prove
    cheaply is marked unproven with the test that would settle it, never
    described as safe.
11. **Report**: budget spent, the base and why, grafts with sources,
    rejections with reasons, dropouts, and whatever is still unproven.

## Plan sections — the shape the board reads

Every section carries `version` and the fields below. `budget.maxTokens`
and `budget.maxWallClockS` become the live ceilings on the meter; a budget
sent under another name is a budget nobody enforces.

`models.roles.judge` is required — set it, on a different family from the
candidates. `models.available` is the catalogue the board offers when a
person edits the plan: every model id you can actually run, with a label
where the id is not self-explanatory. Every section is editable on the
board; an edit becomes the next revision and waits for its own Ratify.

```
criteria:  {items:[{id, label, detail, checks, evidence, deleted?}]}   # deleted:true = retired on the board; judges skip it
budget:    {maxTokens:400000, maxWallClockS:900, maxAttempts?}
fanout:    {workers:3, unit:"candidate", maxConcurrent:3, task:"..."}
models:    {roles:{judge:{model:"…"}, "A":{model:"claude-fable-5",effort:"high"}, "B":{…}},
            available:[{id:"claude-fable-5",label:"Fable 5"}, {id:"claude-opus-5"}, …]}
selection: {rule:"first-pass"|"rank-all"|"best-of"}
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
   PORT=$(sed -n 's/.*"port": *\([0-9]*\).*/\1/p' ~/.helix/arena/<project>/<slug>/server.json)
   curl -s "http://127.0.0.1:$PORT/api/state"           # .plan, .revision, .ready, .gates, .notes
   curl -s "http://127.0.0.1:$PORT/api/plan?revision=N"  # one revision's exact sections
   ```
2. **Always `--wake-on user`.**
   ```sh
   ${CLAUDE_PLUGIN_ROOT}/bin/helix arena wait --arena <slug> --since <cursor> --timeout 600 --wake-on user
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
candidate.created  {candidateId, label, runner:{model, effort}, workspace, revision}
candidate.finished {candidateId, summary, rationale}      # rationale REQUIRED
candidate.dropped  {candidateId, why}                     # proceed with N-1, on the record
score.added        {candidateId, judge, scores:{<critId>:0..3}, notes?, revision}
base.picked        {candidateId, revision, judgePick?, agreed, why}   # why REQUIRED
graft.recorded     {into, from, what, why}
reject.recorded    {from, what, why}                      # why REQUIRED
converged.noted    {shape, why}
diverged.noted     {why}
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

## Runners and the judge report for themselves

Give every runner and the judge the CLI line and require, via Bash:

- `agent.status` on start and on finish — `{agent, text, unitId:
  "<candidateId>"}`. Always include `unitId`, or the board cannot attribute
  the work to a candidate.
- **One `audit` per candidate** carrying the thing it made: `kind:"code"`
  with the key block (≤40 lines) or `kind:"screenshot"` with an absolute
  `path`. The rationale goes in `candidate.finished`; the artifact goes here.
- **One `audit` per verdict from the judge** — `kind:"text"`, ≤300 chars,
  naming the criterion and what decided it. A score with no reasoning on the
  board is a number nobody can argue with.
- **One `metrics` event on finish** from each runner and the judge —
  `{agent, unitId, model, tokens, toolCalls, durationS}`, real model id,
  honest estimates where exact numbers are unavailable. Without it the Stats
  tab reports zero for a run that cost real money.
- The lead pushes `budget.spent` deltas as each runner completes.

Evidence over narration: one entry per step. A run whose Evidence and Stats
tabs are empty did not report — it did not do less work.

## Server operations

- A stale server restarts on the same port when you run `${CLAUDE_PLUGIN_ROOT}/bin/helix arena up`
  again; open tabs reconnect.
- **Kill by pid, never by name.** `pkill -f "helix arena serve"` matches every
  arena server on the machine — other projects, other people's sessions, runs
  in the middle of their work. It looks local and is not. Read the pid instead:
  `kill "$(sed -n 's/.*"pid": *\([0-9]*\).*/\1/p' ~/.helix/arena/<project>/<slug>/server.json)"`.
- The event log survives restarts; state is never in memory only. A server killed
  under a live session is survivable — the session sees exit 5 and `up` brings it
  back — but the work stops until someone notices.

## Honesty rules

- Never ratify or resolve a gate yourself — those come only from the board.
- Never report a score the judge did not produce.
- A dropped candidate is reported, not hidden. N-1 is a fact about the run.
- If the arena cannot separate the candidates, that is a finding about the
  rubric, not a failure to paper over.
