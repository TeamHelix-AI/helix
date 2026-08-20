---
name: rails
description: Read, follow and propose Helix Rails — the rules, standards, playbooks, checklists, templates, policies and safeguards an organisation binds its agents to, scoped to one project or to every project. Use when the user says "/helix-rails", "what are the rules here", "add a rule", "follow the release playbook", "is this allowed", at the start of any Helix session (every skill opens by reading the rails in scope), or when a correction keeps recurring and deserves to become a rail. Rails bind where Memory informs — a rail in scope is not advice, it is the standard the work is held to.
allowed-tools: Bash(${CLAUDE_PLUGIN_ROOT}/bin/helix *)
---

# Helix Rails

Rails are how a team tells its agents *how we do things here*, graded by how
hard each one binds. Memory informs and blocks nothing; **Rails bind** — a
rail in scope is the standard the work is judged against, and where hooks are
installed the platform refuses on its behalf.

One file, one rail. Eight kinds, four modes:

| kind | what it is | usual mode |
|---|---|---|
| rule | a hard constraint — *never force-push main* | refuse |
| safeguard | an act that needs a person first — *confirm before deleting data* | gate |
| standard | how a thing is done — commit format, naming, copy | check |
| playbook | a procedure, in steps — release, incident, key rotation | guide |
| checklist | what "done" means — ticked, with evidence | guide (Stop hook) |
| template | the shape of a document — ADR, postmortem, PR | guide |
| policy | what an agent may touch — tools, paths, data classes | refuse |
| waiver | a time-boxed exception to another rail | — |

`refuse` stops the act. `gate` asks a person. `check` warns and records.
`guide` informs and is cited.

## Setup

Call the binary by its full path every time:

```sh
${CLAUDE_PLUGIN_ROOT}/bin/helix rails <verb> [flags]
```

Never put it in a shell variable or function. The skill pre-approves this
command, and that approval stops matching past a variable assignment — wrap it
and every call prompts instead. Prose below names commands as `rails <verb>`;
run them with the full path.

**One store per machine**, at `~/.helix/rails`, with two layers: **global**
(every project on this machine) and **project**, named by the repo's
`.helix/project` file — or the directory name when there is none. Project
overrides global. Nothing else is written into the repo.

## The first thing every session does

```sh
${CLAUDE_PLUGIN_ROOT}/bin/helix rails for --surface <loop|arena|swarm|canvas|stream|spark|all>
```

Read what comes back as binding. It lists every ratified rail that applies to
this project and this surface, first paragraph each, with its id, kind and
mode — and any waiver that relaxes one. Cite rail ids when you act on them
(`per rail no-force-push`), and when a playbook applies, follow its steps and
report progress against them in your status.

Need the full text — a playbook's steps, a template's body?

```sh
${CLAUDE_PLUGIN_ROOT}/bin/helix rails get <id>
```

## What you may and may not do

| You may | You may not |
|---|---|
| Read every rail in scope, on demand and at session start | Write a rail that binds |
| **Propose** a rail when a correction keeps recurring | Ratify, retire, or change the mode of any rail |
| Report compliance, with evidence | Report a check you did not run |
| Ask for a waiver, in chat | Grant one |

A proposal is a draft on the board, marked *proposed by you*, binding nothing
until a person ratifies it. Humans author rails; you may suggest them.

## The commands

```sh
${CLAUDE_PLUGIN_ROOT}/bin/helix rails for --surface <name> [--tool Bash] [--path src/…] [--format json]
${CLAUDE_PLUGIN_ROOT}/bin/helix rails get <id>
${CLAUDE_PLUGIN_ROOT}/bin/helix rails list [--kind K] [--status S] [--scope global|project]
${CLAUDE_PLUGIN_ROOT}/bin/helix rails propose      # a rail file on stdin → status: proposed
${CLAUDE_PLUGIN_ROOT}/bin/helix rails report <id> --ok|--failed --evidence "…"
${CLAUDE_PLUGIN_ROOT}/bin/helix rails seed <nestjs|react> [--layer global|project]   # a starter pack; never overwrites an edited rail
${CLAUDE_PLUGIN_ROOT}/bin/helix rails up [--no-open]   # the board, for the person
```

Seeding is the one write you may do on a person's say-so: when they ask for
the NestJS or React starter rails, run `seed` and list what it wrote. Each
seed is an ordinary rail afterwards — theirs to edit, retire or restore.

Every verb takes `--dir <projectDir>` when the project is not the cwd.

## Proposing a rail

Propose only what a person would otherwise have to say twice. The file is
frontmatter and a body:

```markdown
---
id: commit-format
kind: standard
mode: check
scope: project
applies:
  surfaces: [all]
  tools: [Bash]
  match: "git commit"
owner:
---

Commit subjects are imperative, under 72 characters, no trailing period.

## When it fires
Rewrite the subject before committing; do not amend history to fix it later.
```

```sh
printf '%s' "$RAIL" | ${CLAUDE_PLUGIN_ROOT}/bin/helix rails propose --agent claude --model <model-id>
```

Say in chat that you proposed it and why. The person ratifies or declines on
the board; you never do.

## Checklists and evidence

A checklist rail lists what "done" means. When its items hold for your work,
report it with the evidence a critic could check:

```sh
${CLAUDE_PLUGIN_ROOT}/bin/helix rails report definition-of-done --ok --evidence "pnpm test: 84 passed; screenshots in audit"
```

Where the Stop hook is installed, an unreported checklist keeps the turn from
ending. A report without evidence is the one thing worse than no report.

## Hooks

Rails work without hooks — `rails for` is the contract every skill keeps. With
`rails install` a person wires the same store into Claude Code so the platform
enforces it: `refuse` rails block a tool call before it runs, `gate` rails
turn it into a question for the person, `check` rails warn after, and open
checklists hold the Stop. Installing, uninstalling and what is mandatory for
an organisation are the person's decisions, not yours.

## What Rails is not

- Not Memory. A fact goes to Memory; a rule goes to Rails. Never express a
  rule as a memory — it silently will not apply.
- Not a place to record what you did. That is the session's log and Archive.
- Not yours to edit. A rail you disagree with is a conversation with the
  person, then a proposal.

## Sharp edges

- Read `rails for` at the start, not when something goes wrong. A refusal you
  meet mid-task was almost always a rail you could have read.
- A waiver has an expiry. Past it the rail binds again; do not rely on one
  you read an hour ago.
- `--surface all` is for session start; inside a surface name it, so
  surface-specific rails reach you.
- Exit 3 means the rail does not exist in any layer; check `rails list`
  before proposing a duplicate.
