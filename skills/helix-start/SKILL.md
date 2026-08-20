---
name: start
description: Route a job to the right Helix surface, or to no surface at all, then launch it. Spark diverges on ideas, Canvas decides together, Loop builds against a ratified bar, Arena judges candidates at a one-way door, Swarm sweeps work too wide for one context, Stream watches autonomous work. Use when the user says "/helix-start", describes a job and asks which mode fits, asks "canvas or loop for this?", "how should we run this", or wants a session set up without naming a surface. Not for jobs already underway on a surface.
---

# Helix Start

The front door. The user describes a job; you pick the surface it deserves,
say why in one sentence, and launch it. The suite is only as good as the
routing into it, and the honest answer is sometimes "no surface at all".

## The seven answers

| Pick | When | The tell |
|---|---|---|
| **spark** | The job is *diverging*: early ideation, naming storms, "what could this even be". Nothing is scoped yet and nothing should be ratified | The user wants to riff, not decide; producing options matters more than picking one |
| **canvas** | The job is *shaping*: requirements, design, naming, trade-offs, a review full of judgment calls | The user will make many decisions as the work unfolds |
| **loop** | The job is *building to a bar*: a deliverable whose definition of done can be written down and ratified, then iterated against by builders and critics | "Done" is crisp; the user wants distance but control at the gates |
| **arena** | The job is a *one-way door*: a shape every downstream thing inherits, where one attempt would lock in the wrong answer and revising later is expensive | Several structurally different answers are possible and you cannot tell which is right by arguing |
| **swarm** | The job is *too wide for one context*: a sweep, an audit, a survey across many independent parts | The work splits into slices that need nothing from each other, and you want findings rather than a build |
| **stream** | The job is *long and autonomous*: refactors, test marathons, migrations, incident watch. The user watches, steers with notes, holds the brake | The user intends to walk away and check in |
| **none** | The job is small: one file, one question, minutes of work | Opening a surface would be ceremony. Do it in the terminal |

Fleet is not an answer here. It is the overview of many sessions, not a
way to run one. Archive is not one either; it records a session after the
fact rather than running it. If the job is "write up / archive / snapshot
what happened", skip routing and invoke `helix-archive` directly.

## Infer first; ask only what the description doesn't say

Read the job description and commit. When the pick is clear, state it with
its one-sentence why and launch. No interview. Ask only when the
description genuinely underdetermines the pick, at most three questions,
in a single round, drawn from the three axes that decide it:

1. **Who drives.** Will this need your calls as we go, or mostly my work
   after an initial brief?
2. **Is there a bar.** Can we write a crisp definition of done before
   starting?
3. **Will you stay.** At the keyboard throughout, checking in
   occasionally, or back at the end?

Riffing with no decisions yet is spark; deciding together is canvas; a
crisp bar to iterate against is loop; a one-way door with several possible
shapes is arena; work too wide for one context is swarm; walking away is
stream; minutes of work is none.

Arena and swarm both cost real money before they return anything, so both
earn a fourth question when the pick is close: **is this worth several agents
at once?** An arena over a settled shape, or a swarm over work one agent could
read in a single pass, is an expensive way to reach the answer you already
had.

## Jobs that span phases

Shaping and building are often the same job. Recommend the sequence:
spark to diverge if nothing is scoped, canvas to ratify the brief or the
quality bar, then loop or stream to build against it. Launch **only the
first surface**. The hand-off to the next one is that session's closing
decision, not something scheduled here.

## Launch

Invoke the chosen skill with the job as its topic: `helix-spark`,
`helix-canvas`, `helix-loop`, `helix-arena`, `helix-swarm`, or
`helix-stream`. Follow that skill's own rules from there, slug
confirmation included; never guess a slug here. For **none**, just do
the work; mention the routing only if the user asked about it.

## When the answer was none: offer the archive at the end

A session that ran in the terminal leaves nothing behind but the
transcript and the diff. When the work is done, look at what it produced.
If it holds **at least one thing worth finding again** — a decision and
its reason, a finding that changed the approach, a constraint discovered
the hard way, an artifact someone else will need — ask, once, in one line:

> Worth keeping? I can archive this session to Helix Archive — decisions,
> findings, artifacts — so it can be found later.

On yes, invoke `helix-archive` and follow its rules (it asks for a level,
then curates). On no, or no answer, drop it; do not ask again in the same
session.

Do **not** offer it when the session was only a lookup, a one-line fix, a
question answered from the code, or work with no decision in it. An
archive of nothing teaches the user to say no to the next one. The test
is not how long the session ran but whether a future reader would open
it: if you cannot name the one card that reader would want, there is
nothing to offer.

If the user overrides your pick, the correction is signal: note in memory
what kind of job it was and where they wanted it, so the next routing is
better.
