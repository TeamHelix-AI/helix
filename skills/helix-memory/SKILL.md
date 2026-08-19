---
name: memory
description: Read and write Helix Memory — the shared store of distilled knowledge that sits under every session, scoped to a project, a team or the whole organisation. Use when the user says "/helix-memory", "remember this", "what do we know about X", "why did the agent think that", when a decision or lesson is worth keeping beyond this session, or when a recalled memory turns out to be wrong and needs finding and fixing. Memory is advice, never enforcement — it shapes what an agent does and blocks nothing.
allowed-tools: Bash(${CLAUDE_PLUGIN_ROOT}/bin/helix *)
---

# Helix Memory

One file, one fact. Memory is not a session surface — it sits underneath
Canvas, Loop and Stream, shared across sessions, projects and people. The
session apps produce events; **Memory is where distilled knowledge lives and
gets recalled from.**

The failure the whole design is built against:

> A wrong memory is worse than no memory, and a wrong *shared* memory is worse
> still.

Recall is automatic and nobody asks for it, so one bad organisation-wide entry
misleads every agent in the company, silently, until somebody traces a bad
decision back to it. That is why provenance, conflict detection and human-gated
promotion all exist — and why you have less power here than the person does.

## Setup

Call the binary by its full path every time:

```sh
${CLAUDE_PLUGIN_ROOT}/bin/helix memory <verb> [flags]
```

Never put it in a shell variable or function. The skill pre-approves this
command, and that approval stops matching past a variable assignment — wrap it
and every call prompts instead. Prose below names commands as `memory <verb>`;
run them with the full path.

First run in a repo needs nothing. **One store per machine**, at
`~/.helix/memory` — the same store and the same server whichever project you
are in. The project layer is a scope inside it, named after the repo
directory, created on first write; nothing is ever written into the repo.

## What you may and may not do

| You may | You may not |
|---|---|
| Write memories at the **project** layer | Write at team or organisation |
| Read everything, automatically and on demand | Put your own memory into the automatic core |
| Report that you acted on a memory | Confirm a memory is still true — that is the person's |
| Propose that a memory move up a layer | Decide that move |

**An agent cannot enlarge what every future agent reads.** Write freely; a
memory earns its way into the core through being used.

## The commands

```sh
${CLAUDE_PLUGIN_ROOT}/bin/helix memory write --name <name> --body "<the fact>" [--type --areas --paths]
${CLAUDE_PLUGIN_ROOT}/bin/helix memory search <query> [--limit 10]   # keyword, widened one hop along typed edges
${CLAUDE_PLUGIN_ROOT}/bin/helix memory recall <path>                 # what this file brings with it
${CLAUDE_PLUGIN_ROOT}/bin/helix memory core [--explain]              # what you would be given here, and why
${CLAUDE_PLUGIN_ROOT}/bin/helix memory used <id>                     # you acted on it — this is the control loop
${CLAUDE_PLUGIN_ROOT}/bin/helix memory get <id|name> · list · doctor
```

Add `--json` to anything. `used` and `confirm` answer with one human line and
nothing machine-readable — deliberately.

**`--session` is not needed.** The session is taken from the host, so a
memory you write carries the session it came out of, and usage is attributed
correctly when two sessions are open on one project.

## Writing a memory worth keeping

```sh
${CLAUDE_PLUGIN_ROOT}/bin/helix memory write --name pnpm-not-npm --type feedback \
  --paths '**/package.json' \
  --body 'Always prefer pnpm to npm.

**Why:** npm rewrites the lockfile and breaks workspace resolution.

**How to apply:** use pnpm in every command, script and doc example.'
```

- **One fact per memory.** If the body has two unrelated claims, it is two
  memories. Atomicity is what lets one be promoted, contradicted or decayed
  without dragging the other with it.
- **The `--why` matters more than the rule.** A memory that says what without
  why cannot be judged when it stops being true.
- **`--type`** is `user` · `feedback` · `project` · `reference`, and it is a
  real ranking signal — `feedback` belongs in the core, `reference` rarely does.
- **Bind it** with `--paths` or `--areas` when it applies somewhere specific.
  Unbound means the whole layer, which is what an org-wide standard looks like.
  Bindings **widen** reach; they never narrow each other.
- **Link it** with typed edges — `supersedes`, `contradicts`, `depends_on`,
  `evidence_for`, `narrows`. An edge may point at an artifact
  (`commit:afd81b3`), which is recorded and never followed.

**Write when a fact will still matter next month and is not derivable from the
code.** Not what the repo already says; not what only matters in this
conversation.

## How memories reach you

Three channels, one ranking, one store:

1. **At session start** — the automatic core, under a hard token budget. You
   receive it; you do not ask for it.
2. **On what you are working on** — memories bound to files in play arrive at
   the next prompt. Not at the moment a file opens: the hook that observes
   cannot add to the conversation, so observation and delivery are split.
3. **When you search** — `memory search`, widened one hop so an answer arrives
   with its caveats attached.

**Say when you used one.** `memory used <id>` is not bookkeeping — it is the whole
of the control loop. A memory acted on is promoted toward the core; one shown
repeatedly and ignored is demoted; one never recalled at all decays to archive.
If nothing ever reports use, everything decays and the store slowly empties of
the things that were working.

## When a memory is wrong

Do not edit it quietly and move on. Either:

- **`memory search`** to find it, then tell the person what it says, where it came
  from and who wrote it — provenance is the point;
- or record the disagreement as an edge (`contradicts`) so it surfaces as a
  conflict rather than a silent overwrite.

`memory doctor` reports broken files, dead edges and conflicts in two shapes: a
declared contradiction, and the same name asserted at more than one layer.

## Promotion is the person's

You may propose; you never decide. A memory reaching team or organisation
needs a named human, and it happens in the browser:

```
${CLAUDE_PLUGIN_ROOT}/bin/helix memory serve --port 4840
```

Point the person there when something deserves to be true for everyone.

## Hooks

`memory hook install` prints the `settings.json` fragment. Three hooks: the core
at session start, path recording after a tool runs, and delivery at the next
prompt. They never fail loudly — a broken install looks like nothing happening,
never like an error on every turn.

## State that survives between runs

A long job needs somewhere to put a cursor, a checkpoint, a partial result —
and that is **not** a memory. Memory is distilled knowledge that gets handed to
agents automatically; a resumption cursor handed to every future session would
fill the recall budget with bookkeeping inside a week. So state is a separate
collection, and **nothing in it ever reaches recall**.

```sh
${CLAUDE_PLUGIN_ROOT}/bin/helix memory state ns <name> --routine <r> [--visibility private|shared|scope]
${CLAUDE_PLUGIN_ROOT}/bin/helix memory state set <ns> <key> --routine <r>   # value on stdin, or --value
${CLAUDE_PLUGIN_ROOT}/bin/helix memory state get <ns> <key> --routine <r>
${CLAUDE_PLUGIN_ROOT}/bin/helix memory state list [<ns>] · rm · prune
${CLAUDE_PLUGIN_ROOT}/bin/helix memory state cadence                        # what ran, and what stopped
${CLAUDE_PLUGIN_ROOT}/bin/helix memory state promote <ns> <key> --body "…"  # turn a result into a memory
```

**State belongs to a routine, not a session** — surviving from one run to the
next is the entire point. Pass `--routine <name>` or set `HELIX_ROUTINE`.

**Private by default.** `shared` reaches a named set of routines, `scope`
reaches anything in the scope. **Reading is shareable; writing never is** —
only the owning routine may write, because shared state anyone can overwrite is
a race rather than state.

**Entries expire** — 30 days by default, per namespace. Bookkeeping should
expire; nothing here is evidence.

**When a result outlives its run, promote it:** `memory state promote` writes a
real memory whose provenance points back at the state entry and the session
that produced it. That is the only path from state into recall, and it is
deliberately something you decide rather than something that happens.

**Nothing here schedules anything.** `memory state cadence` reports what a routine
has been doing — *last ran, usually every day, overdue* — because a routine
that writes state on every run reveals its own rhythm. It can tell you a
nightly job did not run last night. It cannot start one.

## What Memory is not

**Memory is advice.** It shapes what you do and blocks nothing; breaking one
produces no violation and no event. If something must actually be prevented,
that is a policy, and a memory expressed as a rule is a rule that silently does
not apply.

## Sharp edges

- **Nothing schedules the promotion pass.** Usage figures are as fresh as the
  last `memory sweep`. Run it, or offer to.
- **A store has a team layer only if someone configured one.** Unconfigured
  means project and organisation and nothing between.
- **Reading records that memories were shown**, which feeds demotion. Pass
  `--no-track` to look through the store without your looking counting.
- **There is no history.** A hand edit replaces a memory with no trace of what
  it said before. Say so before editing one on someone's behalf.
