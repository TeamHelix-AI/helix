---
name: vault
description: Use credentials through Helix Vault without ever seeing them — declare a need and block until a person supplies it, run commands with secrets injected into their environment, render config files from templates of names. Use when a task needs a password, API key, token or connection string, when a command fails for want of a credential, or when the user says "/helix-vault", "put this in the vault", "which agent used that key". Never ask a user to paste a secret into the conversation; ask Vault for it by name instead.
allowed-tools: Bash(${CLAUDE_PLUGIN_ROOT}/bin/helix *)
---

# Helix Vault

> **You reference secrets. You never read them.**

Vault is not a place to fetch secrets from — it is a broker that *uses* them on
your behalf. A value goes into one command's environment at the moment it runs,
and nowhere else. The command line, the tool output and the transcript never
hold a plaintext value; the audit log holds the *name* and the fact it was used.

There is **no reveal**. Not for you, not for anyone. If you find yourself
wanting the value itself, the answer is one of the two paths below, or the task
changes.

## Setup

Call the binary by its full path every time:

```sh
${CLAUDE_PLUGIN_ROOT}/bin/helix vault <verb> [flags]
```

Never put it in a shell variable or function. The skill pre-approves this
command, and that approval stops matching past a variable assignment — wrap it
and every call prompts instead. Prose below names commands as `vault <verb>`;
run them with the full path.

First use creates a master key in the OS keychain. On a machine where the
keychain prompts, the person will see a dialog — tell them it is coming rather
than letting it surprise them.

`--session` is not needed: the session comes from the host, so a grant that
covers "this session" genuinely does.

## The one thing to do when you need a credential

```sh
${CLAUDE_PLUGIN_ROOT}/bin/helix vault need db/staging --reason "to run the pending migrations" --wait 300
```

Then **stop**. That is the point: you hit a wall, a request is now waiting for
a person, and you are blocked behind it. The reason is in your own words and it
is what the person reads when deciding.

Three answers come back:

| | |
|---|---|
| `db/staging is available for this session` | Proceed. That is all you are told — not the value, not which layer answered, not how long the grant lasts |
| `db/staging was declined: <why>` | **Final.** Report it and stop. Do not ask again, do not work around it, do not look for the credential elsewhere |
| still waiting | Say so and stop. Asking louder returns the same request |

**Never ask the person to paste a secret into the conversation.** That is the
exact thing this exists to prevent — a value in the transcript is there forever.
Point them at `vault pending`, or at the browser surface.

## Using it — two paths, no third

**Run a command with it:**

```sh
${CLAUDE_PLUGIN_ROOT}/bin/helix vault run --with DATABASE_URL=db/staging -- ./migrate.sh
```

The value is in that process's environment and nowhere else. If the command
prints it, the output is masked on the way back to you.

**Get it into a file:**

```
# write a template of names, never values
printf 'DATABASE_URL=${vault:db/staging}\nAPP_ENV=staging\n' > .env.template
${CLAUDE_PLUGIN_ROOT}/bin/helix vault render .env.template --out .env
```

The CLI does the substitution; the value never passes through you. It refuses
to write anywhere git would pick up, and the file is destroyed when the session
ends unless `--keep` is passed — which is itself recorded.

Anything neither path reaches is a case to raise with the person, not to route
around.

## What the person does

```sh
${CLAUDE_PLUGIN_ROOT}/bin/helix vault pending                     # the queue of blocked agents
printf %s '<value>' | ${CLAUDE_PLUGIN_ROOT}/bin/helix vault provide <req>    # answer one
${CLAUDE_PLUGIN_ROOT}/bin/helix vault decline <req> --reason "<why>"         # a real answer
${CLAUDE_PLUGIN_ROOT}/bin/helix vault list · show <name> · audit             # never a value, always a shape
```

Or in the browser, which is where this is meant to happen:

```
${CLAUDE_PLUGIN_ROOT}/bin/helix vault serve --port 4860
```

**A value never goes on a command line** — it is read from stdin, and `--value`
is refused with the reason. If you are writing a command for a person to run,
write the `printf %s '…' |` form.

## Layers and grants

**One vault per machine**, at `~/.helix/vault` — the same store and the same
server whichever project you are in. The project is a *scope inside* the vault,
named after the repo directory; nothing vault-related is ever written into a
repo.

Secrets live at **project**, **team** or **organisation**; most specific wins.
**Inheritance is opt-in** — a secret above your scope is invisible unless it has
been granted to it. Nothing is reachable by accident, and specifically: a
project asking for a name never resolves upward to a production credential it
was never meant to touch.

A grant covers **one session** by default. Anything wider is a deliberate choice
with a visible expiry. Revocation stops resolution immediately.

## The audit log

Every resolution captures **which agent, in which session, running which
command, resolving which layer, granted by whom**. `vault audit` reads it back.
This is what makes "which agent touched the production database key, and when"
answerable — it is the product, not a by-product, so do not paper over it.

Actions by a person are attributed to the account running them. Nobody is
asked to type their own name on a machine with one person on it.

## Hooks

`vault hook install` prints the `settings.json` fragment:

- **PreToolUse** rewrites shell commands to run inside a wrapper that masks this
  session's secrets out of their output. It rewrites nothing when the session
  holds no secrets, so ordinary sessions are untouched. Masking cannot be done
  afterwards — a post-tool hook can observe or block, but it cannot rewrite what
  you have already been shown.
- **SessionEnd** destroys the files this session rendered.

## Sharp edges

- **Only shell commands are shielded.** Reading a rendered file directly reaches
  you unmasked. Those files are gitignored, mode 0600 and destroyed at session
  end, but the window exists — prefer `run --with` over rendering when you have
  the choice.
- **A declined request stays declined for that session.** Asking again returns
  the same refusal by design.
- **A lost master key means the secrets are gone.** There is no recovery and no
  backdoor. Say this before someone stores something they cannot afford to lose.
- **Support has no view here.** That belongs to the hosted console, where there
  are customers and employees. On this machine there is one person and their own
  key.
