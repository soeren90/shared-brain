# Architecture

The shared brain is a **substrate**, not an agent. It does nothing on its own.
Its whole job is to be one consistent, inspectable, version-controlled body of
knowledge that many narrow agents can stand on. Three properties make that work:
how bodies read it, how it stays in sync, and how it stays safe.

```mermaid
flowchart LR
    subgraph Local [Local bodies]
        D[Desktop workbench]
        X[Second workbench]
    end
    subgraph Remote [Remote bodies]
        C[Background routine]
        F[Front door]
    end
    subgraph Brain [shared-brain]
        M[memory/]
        SK[skills/]
        AG[agents/]
        TO[tools/]
        AN[agenda/]
    end
    D -->|symlink| Brain
    X -->|symlink| Brain
    Brain <-->|guarded sync| G[(GitHub · canonical)]
    C <-->|checkout per run| G
    F <-->|scoped token| G
```

## 1. How a body reads the brain

Bodies on my laptop symlink the brain's content into their own working
directory:

```
<body>/skills/<name>  ->  shared-brain/skills/<name>
<body>/memory         ->  shared-brain/memory
```

Bodies that run elsewhere (a scheduled routine in the cloud, the assistant on
my phone) check out the canonical repository at the start of each run and write
back to it. They never keep a private copy that could drift.

The symlink has a large consequence: **a change to the brain is live the
instant it's written**, before anything is committed and before it reaches any
other body. A local body isn't reading a snapshot; it's reading the working
tree.

That single fact drives the two models below. If edits are live immediately,
then:

- a "review before you commit" gate protects nothing (the change is already
  active locally), so review has to move *after* the change and be
  **guaranteed**, not optional (see the sync model);
- an agent that can edit the brain can edit *its own instructions*, so the
  automation that distributes changes needs **guardrails of its own** (see
  §4, Security by design).

## 2. What the brain stores

Five content areas, deliberately narrow:

| Area | Purpose | Written by |
| --- | --- | --- |
| `memory/` | Durable facts, one per file, plus an index | Bodies (after tasks), the operator |
| `skills/` | Reusable capability definitions | The operator, in dialog with a body |
| `agents/` | Specialist sub-agent definitions | The operator |
| `tools/` | Deterministic token-free scripts | The operator |
| `agenda/` | Open loops, deadlines, commitments | The background routine, voice notes, the operator |

Everything else (the wiring that connects bodies to the brain, host config,
the sync script itself) lives **outside** these paths. The automated sync is
scoped to content only; it can never rewrite the rules it runs under. That
boundary is enforced, not just documented (see §4, Security by design).

## 3. How it stays in sync

A small dependency-free script on the laptop commits, pushes, and pulls on a
timer, with no manual git, four guardians that **halt and notify** instead of
pushing anything surprising, and a heartbeat so a dead daemon is visible at a
glance. Remote bodies read and write the same canonical repository, so every
change from any body ends up in one history.

Full detail: **[docs/syncing-across-hosts.md](docs/syncing-across-hosts.md)**.

## 4. Security by design

This brain is read every day by agents that can run code, reach mail and chat,
and process untrusted inbound content. That combination is the **lethal
trifecta** (Simon Willison's term): private data, exposure to untrusted content,
and a way to reach the outside world. Any system with all three can be told, by
an attacker's text, to read something private and send it somewhere you don't
control.

The reason you can't prompt your way out of it: an LLM **cannot separate
instructions from data**. Everything it reads enters the context as tokens, and
it doesn't distinguish the sentences you wrote from the ones an attacker slipped
into an email. So prompt injection is a property of the architecture, not a bug
to patch, and a rule like "never touch the credentials" in the system prompt is
documentation, not a control.

The design response is structural, not textual: **guarantee that in any
execution path at least one leg of the trifecta is missing.**

- **Give every body only the access its role needs.** The background routine
  runs in a fresh, disposable cloud sandbox per run and may only *read* mail,
  calendar and chat, each tool granted one by one. The assistant on my phone
  reaches the brain through a token scoped to this one repository and nothing
  else.
- **Cut it again between agents.** Sub-agents get only the tools their one job
  needs. The component that reads the web carries no mail or document access, so
  the part touching untrusted content can't also reach private data.
- **Cut the exit leg for consequential actions.** Nothing is sent, posted,
  published, or purchased on its own. Outward actions are prepared as drafts and
  proposals; a human confirms. Where it matters most, the approval is enforced in
  code rather than in the prompt: a hook in front of the publishing tool blocks
  the call unless the operator's own latest message approves it. Untrusted
  content can shape a *suggestion*, never trigger an *action*.
- **Make the knowledge base reversible.** Every change is a version-controlled
  commit, distributed by an automation scoped to content only that can't rewrite
  its own rules. History is the quarantine and the rollback path
  ([docs/syncing-across-hosts.md](docs/syncing-across-hosts.md)).
- **Keep it observable.** Actions leave an audit trail; the sync leaves a
  heartbeat. Behavior can always be reconstructed.

Each layer is built to assume the ones around it can fail; no single boundary
is load-bearing alone. The same coverage also maps cleanly onto the OWASP Top 10
for Agentic Applications (2026); the trifecta is just the sharper way to reason
about *why* each boundary is there.

## Why this holds up

- **One source of truth.** Bodies that should agree, agree: they read the same
  files, whichever vendor runs them.
- **Inspectable and revertible.** The brain is text under git; any change can be
  read, diffed, and undone.
- **Small blast radius.** The sync touches content only; each body's access is
  scoped to its role; outward actions wait for a human.
- **Never silent.** Sync runs leave a heartbeat; tool calls leave an audit
  trail. "Why did it do that?" always has an answer.
