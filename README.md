# Team Helix

Visual surfaces for working with Claude Code agents, plus the shared store and
secret broker beneath them. Everything runs on your machine.

```
/plugin marketplace add TeamHelix-AI/helix
/plugin install helix@team-helix
```

Then `/helix:helix-start` in any project, and Helix picks the surface the job
deserves.

## The surfaces

| | | |
|---|---|---|
| **Start** | Routing | Describe the job; Helix picks the surface it deserves, or none at all. |
| **Spark** | Diverging | A chat feed and a shelf of living idea cards. Riff, then harvest the survivors. |
| **Canvas** | Deciding | Cards on a canvas — questions, options, decisions, diagrams. You answer in the browser. |
| **Loop** | Building | A human-gated gauntlet. Builders and critics iterate against a bar you ratified. |
| **Arena** | Judging | One question, N candidates, one judge scoring against a rubric you ratified. The winner becomes the base; the best of the losers is grafted in. |
| **Swarm** | Sweeping | One job cut into disjoint slices, one worker each. Every worker says how much it actually read; the merged report names its own gaps. |
| **Stream** | Watching | A dashboard for autonomous work. Watch, steer, hold the brake. |
| **Archive** | Remembering | A session's story — chapters, decisions, findings, artefacts — written once, read-only, kept. |
| **Fleet** | Mission control | Every session across every project on one board, with a "needs you" queue. |

## The infrastructure

| | | |
|---|---|---|
| **Memory** | What we know | One file, one fact. Distilled knowledge, recalled under every session. Advice, never enforcement. |
| **Vault** | Secrets | Agents reference credentials by name and never read them. A value enters one command's environment and nowhere else. |
| **Rails** | What binds | Rules, standards, playbooks, checklists, safeguards. Humans author, agents propose; every skill reads the rails in scope first. Optional Claude Code hooks make refusals real. |
| **Shelf** | What is kept | Finished artefacts with a title, a description and their context, immutable and addressed by id from any later session. |

## Commands

| | |
|---|---|
| `/helix:helix-start` | Route a job to a surface |
| `/helix:helix-spark` | Open a spark |
| `/helix:helix-canvas` | Open a canvas |
| `/helix:helix-loop` | Run a loop |
| `/helix:helix-arena` | Run an arena |
| `/helix:helix-swarm` | Run a swarm |
| `/helix:helix-stream` | Open a stream |
| `/helix:helix-archive` | Archive this session |
| `/helix:helix-fleet` | Open Fleet |
| `/helix:helix-memory` | Read or write Memory |
| `/helix:helix-vault` | Use a credential through Vault |
| `/helix:helix-rails` | Read, follow or propose Rails |
| `/helix:helix-shelf` | Put an artefact on the Shelf, or take one down |

## Requirements

macOS or Linux, on arm64 or amd64.

The first Helix command downloads the `helix` binary (~23 MB) from this
repository's releases and caches it at `~/.helix/bin/`. The download is
verified against `bin/checksums.txt`, which ships with the plugin — a
mismatch refuses to run. Nothing else is installed.

Vault creates a master key on first use, in the macOS keychain by default. On
a machine where the keychain prompts, expect a dialog that first time.

## Where things live

One machine-wide home, never inside your repositories:

```
~/.helix/<app>/…      event logs, session state, memories, the vault, rails, the shelf
~/.helix/bin/         the cached binary
```

`HELIX_HOME` moves all of it. `HELIX_BIN` points at a binary of your own.

## Updating

```
/plugin marketplace update team-helix
/plugin update helix@team-helix
```

A new plugin version fetches its matching binary on the next command.

---

Apache 2.0. Published from the Team Helix source repository; issues and
discussion here.
