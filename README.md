# za-actor

**座 (za) の封じ込め済み実体。** 座 = 中世の職人・商人の自治結社。
座法という規約を持ち、座衆がそれに従って縄張りを侵さずに働く —— これは
`kotoba-fleet` の coordination substrate が agent に対してやっていることの語。
旧名 `fleet`（2026-08-07 rename: bare な `fleet` が二つの org に在り、三つの
repo が `fleet.*` namespace を主張していた）。

**FleetCoordinatorActor** — the deployed, sealed instance of the
[kotoba-fleet](https://github.com/kotoba-lang/kotoba-fleet) coordination
substrate. Same actor pattern as robotaxi-actor (AR1 ⊣ SafetyGovernor) /
ai-gftd-itonami (ops-LLM ⊣ CertGovernor): a contained intelligence node whose
output is always censored by an independent governor, on a langgraph-clj
StateGraph, with an append-only audit ledger.

- **Coordinator** (`za-actor.coordinator`) — the *contained intelligence node*.
  Proposes which pending agent-write to advance next. Proposal **only**; a mock
  today, swap in an LLM advisor (`langchain.model`) via `:advise` without
  touching the graph or the governor.
- **FleetGovernor** (`za-actor.governor`) — the independent censor. Enforces the
  single invariant: **a proposal materializes only if the proposing agent holds
  the lease on its work-unit and the gate passes; writes to protected paths
  (`manifest/`, `90-docs/adr/`) additionally require human sign-off.**
- **Actor** (`za-actor.actor`) — `observe → coordinate → govern → decide →
  materialize | hold | (human-signoff)` on a StateGraph. One run = one
  coordination tick (no unbounded inner loop). `interrupt-before
  #{:human-signoff}` is real human-in-the-loop. The injected `:materialize` hook
  is the single git writer, reached only for accepted proposals.
- **Runner** (`za-actor.runner`) — the kotoba-code seam: `coding-run` turns a bounded
  coding session into the fleet agent's `:run` fn, running it against a
  **capturing host** so the session's `write_file`s become proposal data (never a
  git write). The session driver is injected (`:session!`) — plug in kotoba-code's
  `build-agent`+`run-task` in production (needs `OR_KEY`/Murakumo gateway for a
  live model). Contract-tested with a mock session.
- **Worktree** (`za-actor.worktree`) — F3 per-agent isolation: `worktree-run` runs
  each coding session in its OWN detached git worktree, so reads and test-runs
  never collide across agents and the main tree is untouched. Captured writes are
  **gated** (test-cmd run in the worktree) before proposing — a red gate proposes
  nothing (never a broken edit). Worktrees live under `.fleet-wt/` and are removed
  after use.
- **Driver** (`za-actor.driver`) — the durable outer loop + multi-node roles.
  `run-node!` (single node) repeats *agents claim open work → run → propose →
  governor drains → close the unit*, bounded by `:budget`, crash-recoverable via
  lease TTL. For deployment scale (F4) it splits into **`agent-round!`** (runs on
  every node) and **`govern!`** (runs on exactly ONE node — the single git
  writer). Many nodes share one Datom graph; the optimistic lease resolves
  cross-node contention with no lock server. See [`docs/DEPLOY.md`](docs/DEPLOY.md)
  for the 2-PC × ~20-agent runbook. Tested: full-drain, idempotent re-run,
  crash-recovery-via-TTL, budget bound, two-nodes-one-governor, cross-node lease
  exclusion.
- **Node launcher** (`za-actor.node`) — env-driven entry point (`FLEET_ROLE` /
  `FLEET_AGENTS` / `FLEET_BUDGET` / `FLEET_GRAPH` / `OR_KEY`). `run-role!`
  dispatches `:agent`/`:governor`/`:both`; `-main` is a self-contained local
  smoke (in-memory store + echo run, seeded queue — `kbb -M:dev -m
  za-actor.node`). Production wires the kotoba-db store + a kotoba-code session and
  calls `run-role!` (see the ns doc + `docs/DEPLOY.md`).
- **kotoba-native residency** (`deploy/kotoba/`) — the ~20 agents run **through
  the kotoba mesh, not `sh`**: `fleet_agent.clj` is a WASM `on-tick` component
  (one agent round per firing, pure datom ops), deployed via `kotoba app deploy
  deploy/kotoba/fleet.app.edn --publish` — the mesh is the scheduler. The one
  governor (git writer) stays a host process (`deploy/install-node.sh --role
  governor`). See `docs/DEPLOY.md`.

The invariant that lets ~20 parallel coding agents share one repo without git
conflict: **agents only append proposals + hold leases; exactly one governed
coordinator decides what reaches git.** See ADR-2606302000 (design) and
ADR-2607010900 (this actor).

## Build

```bash
kbb -M:lint       # clj-kondo (errors fail)
kbb -M:dev:test   # contract + integration tests (langchain-clj pinned to local checkout)
kbb -M:dev:run    # capstone: 8 agents, 1 repo, one governed writer (za-actor.sim)
```

`za-actor.sim` is the end-to-end capstone: 8 agents across 2 PCs coordinate on one
repo — 5 disjoint edits materialize in parallel (no git conflict), a protected
write pauses for sign-off then materializes, an expired-lease proposal is held by
the govern-time invariant, and a racer loses the lock-free claim and never runs.
`za-actor.integration-test` asserts these properties.

## Identity (RAD)

- repo:    `github.com/etzhayyim/com-etzhayyim-fleet`
- did:web: `did:web:etzhayyim.github.io:com-etzhayyim-fleet`
- ledger:  `etzhayyim/root:80-data/kotoba-rad/fleet.identity.journal.edn`
