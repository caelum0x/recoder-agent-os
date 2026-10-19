# Recoder Agent OS

The open-source agent orchestration layer behind [Recoder](https://recoder.xyz) — a
Cursor-like coding agent product where you run Claude Code, Codex, Cursor, pi,
and opencode in parallel isolated git worktrees, then review and merge.

This repository is a filtered extraction of the `packages/agent-*` (and
closely-related) workspace packages from the private Recoder monorepo. They
were already designed as decoupled, runtime-agnostic contract and SDK
packages inside that monorepo — "do not couple core packages to specific
runtimes, UI apps, sandboxes, or model providers; runtimes and OSS systems are
adapters around Recoder state, not owners of Recoder state" — so pulling them
out as a standalone harness required no rewrite, only a path filter that
preserves real commit history (via `git filter-repo`).

## What's here vs. what stays private

**Here (MIT, this repo):** the agent-loop orchestration contracts — domain
state machines, policy gates, instruction compilation, memory-candidate
lifecycle, A2A wire protocol, MCP tool registry, workflow/runtime adapter
contracts. This is "the harness": the shape of a task/run/approval/memory
system that any agent runtime (Claude Code, Codex, a custom CLI, etc.) can be
adapted onto.

**Stays private (the hosted `recoder.xyz` product):** multi-tenant identity
and org provisioning, billing (Dodo integration), the governance secrets
vault, social/channel connectors, the desktop app (Wave-fork Electron shell),
production deploy configuration, and the concrete backend that wires these
contracts to a real Postgres-backed control plane. Those pieces are the
hosted service, not the harness.

## Packages

| Package | What it does |
|---|---|
| [`@recoder/agent-os`](packages/agent-os) | Domain core: the `TaskSpec` / run / approval state machine and its legal transitions. The spine every other package builds on. |
| [`@recoder/a2a-sdk`](packages/a2a-sdk) | Agent2Agent (A2A) wire-level interoperability: task-state mapping, error taxonomy, envelope builders, audit event emitters. |
| [`@recoder/agent-instructions`](packages/agent-instructions) | Compiles human task intent (goal, scope, constraints, verification, approvals) into runtime-specific instruction bundles, with provenance. |
| [`@recoder/agent-policy`](packages/agent-policy) | Deterministic TypeScript policy gates — the decision layer that approves or blocks risky actions before they execute. |
| [`@recoder/agent-events`](packages/agent-events) | CloudEvents-shaped events and an outbox pattern for reliable cross-domain event delivery. |
| [`@recoder/agent-memory`](packages/agent-memory) | Reviewed memory-candidate lifecycle: propose with evidence, review, promote to durable — never silent writes. |
| [`@recoder/agent-runtimes`](packages/agent-runtimes) | Runtime adapter contracts, including a clean-room TypeScript client for the herdr/Loom terminal-manager wire protocol. |
| [`@recoder/agent-workflows`](packages/agent-workflows) | Workflow definition and driver contracts for multi-step, resumable agent work. |
| [`@recoder/agent-mcp`](packages/agent-mcp) | MCP (Model Context Protocol) registry and tool-permission contracts — what an agent may call, and under whose identity. |
| [`@recoder/agent-schemas`](packages/agent-schemas) | Runtime-safe schema guards for the above, with zero provider SDK dependencies. |
| [`@recoder/beads-adapter`](packages/beads-adapter) | Adapter around the [`bd`](https://github.com/steveyegge/beads) CLI (beads issue-graph store) used as cross-agent shared memory. |
| [`@recoder/recodergit`](packages/recodergit) | Contracts for repo intelligence / context-pack generation handed to an agent before it starts work. |
| [`@recoder/oss-registry`](packages/oss-registry) | A registry and health-descriptor format for composing OSS engines/modules into a product (Recoder's own "module, not wrapper" pattern). |
| [`@recoder/pal`](packages/pal) | Planner and agent-team routing: runtime/risk/approval decisions, model routing, multi-model consensus/escalation. |

All 14 packages are MIT-licensed, have no dependency on `backend/` or
`apps/desktop/` from the private monorepo, and contain no secrets, internal
hostnames, or `.env` values (verified with `gitleaks detect` across full
history before this repo went public).

## Design principles (carried over from the monorepo)

- **Memory is proposed, never written directly.** Every memory update is an
  evidence-backed candidate; only a review step promotes it to durable.
- **Policy is enforced below the model.** Agent output, tool descriptions, and
  runtime-reported success are never trusted as authorization; a deterministic
  policy gate decides.
- **Health must be truthful.** Adapters report `up`/`down` for what they
  actually checked — never a hardcoded "ok".
- **Runtimes are adapters, not owners.** Claude Code, Codex, Cursor, a custom
  CLI — all sit behind the same `agent-runtimes` contract; none of them own
  canonical task/approval/memory state.

## Install

```bash
pnpm install
pnpm build
pnpm test
```

Each package builds independently with `tsup` and type-checks with `tsc
--noEmit`. Tests run with Node's built-in test runner via `tsx`.

## Relationship to recoder.xyz

Recoder (the hosted product) imports these same packages as workspace
dependencies in its private backend, where it adds the proprietary pieces
listed above (identity, billing, governance vault, connectors) and a real
Postgres-backed implementation of the ports these packages define. If you're
building your own agent orchestration product, you can depend on these
packages directly, or fork them, without any of that hosted-service layer.

## License

MIT — see [LICENSE](LICENSE). See [NOTICE](NOTICE) for third-party protocol
and tooling attribution.
