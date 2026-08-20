---
name: shelf
description: Put finished artefacts on Helix Shelf and take them down again in any later session — design docs, reports, design systems, screenshots, bookmarks, bundles of files — each with a title, a description and the context it came from, scoped to one project or to every project. Use when the user says "/helix-shelf", "shelve this", "put this on the shelf", "is there a doc for X", "get the tokens from the shelf", when a deliverable is finished and worth finding again, or when a session needs something another session produced. Shelf keeps; it never binds and never decides.
allowed-tools: Bash(${CLAUDE_PLUGIN_ROOT}/bin/helix *)
---

# Helix Shelf

The cupboard, not the diary. Archive tells the story of a session; **Shelf
holds the thing itself** — the artefact, with enough context to know what it
is and when it applies — so any later session can take it down by id or by
search.

One item is one immutable folder: `item.md` (title, description, context,
tags, where it came from) and the files. A revision is a new item that
`supersedes` the old one; the old one stays readable, so a reference made
last month still points at what it saw.

## Setup

Call the binary by its full path every time:

```sh
${CLAUDE_PLUGIN_ROOT}/bin/helix shelf <verb> [flags]
```

Never put it in a shell variable or function. The skill pre-approves this
command, and that approval stops matching past a variable assignment — wrap it
and every call prompts instead. Prose below names commands as `shelf <verb>`;
run them with the full path.

**One store per machine**, at `~/.helix/shelf`, with two layers: **global**
and **project**, named by the repo's `.helix/project` file — or the directory
name when there is none. Both are searched. Nothing else is written into the
repo.

## The commands

```sh
# put something on the shelf → {"status":"ok","id":"shf_…"}
${CLAUDE_PLUGIN_ROOT}/bin/helix shelf put --title "…" --description "…" --context "…" [--tag t]… \
    [--scope project|global] [--supersedes shf_…] [--session <app/project/slug>] [--source <ref>] <files…>
${CLAUDE_PLUGIN_ROOT}/bin/helix shelf put --title "…" --description "…" --url https://…      # a bookmark

# take something down
${CLAUDE_PLUGIN_ROOT}/bin/helix shelf get <id> [--latest] [--out <dir>]     # --out writes the files to <dir>/<id>/
${CLAUDE_PLUGIN_ROOT}/bin/helix shelf find "<query>" [--tag t] [--scope L] [--limit n]
${CLAUDE_PLUGIN_ROOT}/bin/helix shelf list [--scope L]

${CLAUDE_PLUGIN_ROOT}/bin/helix shelf up [--no-open]                        # the board, for the person
```

Every verb takes `--dir <projectDir>` when the project is not the cwd. Pass
`--agent <name> --model <id>` on `put` so the item records who shelved it.

## Shelving well

Shelve **finished** things a later session would want: the ratified design
doc, the report the swarm merged, the design tokens, the screenshot set a
critic judged against. Not drafts, not scratch, not anything already in the
repo (reference the commit instead).

The three lines are what make an item findable a month on:

- **title** — what it is, in one line
- **description** — what it is for
- **context** — where it came from and when it applies: the session, the
  decision, the date. Write it for someone who has no other way to know.

```sh
${CLAUDE_PLUGIN_ROOT}/bin/helix shelf put \
  --title "Helix Rails and Shelf — design" \
  --description "Ratified design for the two shared services." \
  --context "From the rails-shelf canvas, 20 Aug 2026; four decisions ratified. Use as the spec until the apps ship." \
  --tag design --tag helix --scope project \
  --session canvas/skills/rails-shelf --agent claude --model <model-id> \
  brainstorming/helix-rails-shelf.md
```

Revising? Put the new version with `--supersedes <old id>`; never edit a
shelved file in place — there is no in place.

## Referencing

An item's id is its address everywhere: `shelf:shf_7c1e2a` in a canvas card, a
memory, a rail, an archive chapter. `shelf get <id>` hands you exactly what
was shelved; `--latest` follows the supersedes chain to the newest revision.

When a session needs something it does not have, `shelf find` before building
it again.

## What Shelf is not

- Not Archive. The story of a session, its chapters and decisions, is
  Archive's. Shelf keeps only the artefact and its context.
- Not Memory. A fact worth recalling automatically goes to Memory; Shelf is
  never injected, only fetched.
- Not a file system. Items are immutable and addressed by id; there are no
  folders to tidy.

## Sharp edges

- An item with a vague context is as good as lost. If you cannot say when it
  applies, you are not ready to shelve it.
- `get` without `--latest` is deliberate: it returns what the reference
  pointed at, superseded or not. Say so when you hand it over.
- Large bundles shelve slowly and are rarely what the next session wants;
  shelve the deliverable, link the rest.
