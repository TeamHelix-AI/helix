# Team Helix

Visual surfaces for working with Claude Code agents, plus the shared store and
secret broker beneath them. Everything runs on your machine.

```
/plugin marketplace add TeamHelix-AI/helix
/plugin install helix@team-helix
```

Then `/helix:start` in any project, and Helix picks the surface the job
deserves.

## The surfaces

| | | |
|---|---|---|
| **Spark** | Diverging | A chat feed and a shelf of living idea cards. Riff, then harvest the survivors. |
| **Canvas** | Deciding | Cards on a canvas — questions, options, decisions, diagrams. You answer in the browser. |
| **Loop** | Building | A human-gated gauntlet. Builders and critics iterate against a bar you ratified. |
| **Stream** | Watching | A dashboard for autonomous work. Watch, steer, hold the brake. |
| **Fleet** | Mission control | Every session across every project on one board, with a "needs you" queue. |

## The infrastructure

| | | |
|---|---|---|
| **Memory** | What we know | One file, one fact. Distilled knowledge, recalled under every session. Advice, never enforcement. |
| **Vault** | Secrets | Agents reference credentials by name and never read them. A value enters one command's environment and nowhere else. |

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
~/.helix/<app>/…      event logs, session state, memories, the vault
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
