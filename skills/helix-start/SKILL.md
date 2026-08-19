---
name: start
description: Route a job to the right Helix surface — Spark (diverging on ideas), Canvas (deciding together), Loop (building against a ratified bar), Stream (watching autonomous work) — or to no surface at all, then launch it. Use when the user says "/helix-start", describes a job and asks which mode fits, asks "canvas or loop for this?", "how should we run this", or wants a session set up without naming a surface. Not for jobs already underway on a surface.
---

# Helix Start

The front door. The user describes a job; you pick the surface it deserves,
say why in one sentence, and launch it. The suite is only as good as the
routing into it — and the honest answer is sometimes "no surface at all".

## The five answers

| Pick | When | The tell |
|---|---|---|
| **spark** | The job is *diverging*: early ideation, naming storms, "what could this even be" — nothing is scoped yet and nothing should be ratified | The user wants to riff, not decide; producing options matters more than picking one |
| **canvas** | The job is *shaping*: requirements, design, naming, trade-offs, a review full of judgment calls | The user will make many decisions as the work unfolds |
| **loop** | The job is *building to a bar*: a deliverable whose definition of done can be written down and ratified, then iterated against by builders and critics | "Done" is crisp; the user wants distance but control at the gates |
| **stream** | The job is *long and autonomous*: refactors, test marathons, migrations, incident watch — the user watches, steers with notes, holds the brake | The user intends to walk away and check in |
| **none** | The job is small: one file, one question, minutes of work | Opening a surface would be ceremony. Do it in the terminal |

Fleet is not an answer here — it is the overview of many sessions, not a
way to run one.

## Infer first; ask only what the description doesn't say

Read the job description and commit. When the pick is clear, state it with
its one-sentence why and launch — no interview. Ask only when the
description genuinely underdetermines the pick, at most three questions,
in a single round, drawn from the three axes that decide it:

1. **Who drives** — will this need your calls as we go, or mostly my work
   after an initial brief?
2. **Is there a bar** — can we write a crisp definition of done before
   starting?
3. **Will you stay** — at the keyboard throughout, checking in
   occasionally, or back at the end?

Riff, no decisions yet → spark. Decide-together → canvas. Crisp bar +
iterate → loop. Walk away → stream. Minutes of work → none.

## Jobs that span phases

Shaping and building are often the same job: recommend the sequence
(spark to diverge if nothing is scoped, canvas to ratify the brief or the
quality bar, then loop or stream to build against it), but launch **only
the first surface**. The hand-off to
the next one is that session's closing decision, not something scheduled
here.

## Launch

Invoke the chosen skill — `helix-spark`, `helix-canvas`, `helix-loop`, or
`helix-stream` — with the job as its topic, and follow that skill's own rules from there
(slug confirmation included; never guess one here). For **none**, just do
the work; mention the routing only if the user asked about it.

If the user overrides your pick, the correction is signal: note in memory
what kind of job it was and where they wanted it, so the next routing is
better.
