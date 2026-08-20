# Team Helix

Helix gives Claude Code a set of surfaces designed for different kinds of work.

Some jobs start with a conversation. Others need a whiteboard, parallel workers, structured reviews or a live dashboard. Helix chooses the right surface and provides the shared memory, secret handling and infrastructure behind it.

Everything runs on your machine.

```bash
/plugin marketplace add TeamHelix-AI/helix
/plugin install helix@team-helix
```

Start with:

```text
/helix:helix-start
```

Describe what you want to do and Helix will choose the right surface.

If you already know where you're going, you can open any surface directly.

---

# Surfaces

| Surface | Purpose | Description |
|----------|---------|-------------|
| **Start** | Entry point | Describe the work. Helix chooses the right surface or simply gets started. |
| **Spark** | Explore | Brainstorm with your agents. Good ideas become cards that you can keep, combine or discard. |
| **Canvas** | Decide | A shared board for questions, options, diagrams and decisions. |
| **Loop** | Build | Agents build, review and improve their work in rounds. You decide when it's ready to continue. |
| **Arena** | Compare | Generate several solutions, score them against your criteria and combine the strongest ideas into one result. |
| **Swarm** | Parallelise | Split a large job into independent pieces and let multiple agents work at the same time. The final report shows what was covered and where the gaps are. |
| **Stream** | Monitor | Watch agents work in real time. Pause them, redirect them or step in whenever you want. |
| **Archive** | Remember | A permanent record of a session, including its decisions, findings and outputs. |
| **Fleet** | Overview | See every active session across every project from one place, including the work waiting for your attention. |

---

# Shared infrastructure

These services are available from every surface.

| Component | Purpose | Description |
|-----------|---------|-------------|
| **Memory** | Long-term knowledge | Store facts once and reuse them across sessions. Memory suggests. It never overrides. |
| **Vault** | Secrets | Store credentials securely. Agents can use them without ever seeing the secret itself. |
| **Rails** | Team standards | Rules, playbooks, coding standards and checklists shared across your projects. Humans write them. Agents follow them. |
| **Shelf** | Shared artefacts | Save finished work together with its context so it can be found and reused later. |

---

# Commands

Every surface can be opened directly.

| Command | Description |
|---------|-------------|
| `/helix:helix-start` | Let Helix choose the right surface |
| `/helix:helix-spark` | Open Spark |
| `/helix:helix-canvas` | Open Canvas |
| `/helix:helix-loop` | Open Loop |
| `/helix:helix-arena` | Open Arena |
| `/helix:helix-swarm` | Open Swarm |
| `/helix:helix-stream` | Open Stream |
| `/helix:helix-archive` | Archive the current session |
| `/helix:helix-fleet` | Open Fleet |
| `/helix:helix-memory` | Read or update Memory |
| `/helix:helix-vault` | Use a stored credential |
| `/helix:helix-rails` | Read, follow or update Rails |
| `/helix:helix-shelf` | Save or retrieve an artefact |

---

# Requirements

Helix supports:

- macOS
- Linux
- arm64
- amd64

The first time you run a Helix command, it downloads the `helix` binary (about 23 MB) into:

```text
~/.helix/bin/
```

The download is verified against the bundled checksum before it runs.

Nothing else is installed.

The first time you use Vault, it creates a master key. On macOS the key is stored in the Keychain, so you may be asked to grant access once.

---

# Storage

Helix stores everything outside your repositories.

```text
~/.helix/
├── bin/
├── memory/
├── vault/
├── rails/
├── shelf/
├── sessions/
└── logs/
```

Set `HELIX_HOME` to move the entire directory.

Set `HELIX_BIN` to use your own Helix binary.

---

# Updating

```bash
/plugin marketplace update team-helix
/plugin update helix@team-helix
```

The next time you run a Helix command, the matching binary is downloaded automatically.

---

## License

Apache 2.0.

Published from the Team Helix repository. Issues, discussions and contributions are welcome.