# subagent-delivery

[中文](./README.md) | [English](./README_EN.md)

A Codex skill for **development delivery after requirements are confirmed**. It isolates implementation details in fresh subagents, creates local unpushed stage checkpoints, schedules sequential and parallel work conservatively, and closes testing, real HTTP verification, independent review, and fix loops.

## Scope

- Inputs should be a confirmed OpenSpec change, tickets, PRD tasks, or an equally explicit development request.
- This skill does not perform requirements discovery, solution discussion, or scope clarification. First use the strongest current reasoning tier with **OpenSpec** to refine requirements; for large work, use **Wayfinder** to define stages and dependencies before entering this skill.
- It stops for confirmation when an unresolved choice affects business behavior, data, permissions, security, API contracts, or scope.

## Guarantees

- Functional closure comes first, required safety is built in, and extra hardening is risk-proportionate: complete the minimum production-ready main flow and necessary failure paths without expanding scope, stacking abstractions, or extending review indefinitely in pursuit of absolute safety or theoretical zero bugs.
- Dev, Integrator, and Reviewer use fresh subagents without full-history inheritance. The main conversation retains only key decisions, states, SHAs, and evidence summaries.
- Confirmed requirements are the write-scope ceiling. Implement only accepted behavior and inseparable minimal support; do not add adjacent features, contracts, migrations, or unrelated refactors without confirmation.
- Adjacent sequential stages in the same requirement, call chain, and ownership scope reuse the current Dev by default. Rotate to a fresh Dev only for a domain shift, a distinct high-risk boundary, or a context-health failure.
- The main task always keeps its current model, while subtasks choose between that model and an available one-tier-lower model according to complexity. Reasoning defaults to `medium`; Integrator, independent Reviewer, and complex or high-risk work require `high`. If `high` is unavailable, the affected node pauses for confirmation instead of silently degrading or upgrading models. Only an explicitly named reasoning level overrides these defaults.
- The skill selects one or more development subtasks and automatically dispatches the next ready node after a node completes.
- Only the main task creates and schedules direct subagents. Subagents do not spawn grandchildren; they return requests for additional roles to the main task. Direct subagents that satisfy the parallel whitelist may still run in parallel.
- Code-writing work is sequential by default. It runs in parallel only when dependencies, files, contracts, database/fixture/port resources, integration, and rollback are all proven independent. Shared files, unfrozen contracts, or shared data resources force sequential execution.
- Exactly one active Dev writes the target worktree at a time during sequential work. Isolated worktrees and a single Integrator are used only for parallel batches.
- Every stage creates a local checkpoint commit by default, without pushing, opening a PR, deploying, or archiving. If the user or project forbids commits, the skill uses a sequential snapshot mode.
- TDD is conditional. The Dev simplifies and self-reviews the implementation; once a sequential chain is fixed, it gets one final independent Review node that checks Standards and Spec sequentially in the same session. P0-P3 findings introduced by the change, blocking acceptance, having a real failure path in the current scope, or affecting the current call chain and required safety are fixed and re-reviewed; only historical issues proven unrelated to the delivery and theoretical suggestions become residual risks.
- New or changed HTTP APIs are called through the real route with sanitized, representative parameters. Mocks and schema checks do not replace endpoint acceptance.
- Only one CodeGraph service is reused per repository: the main task records the instance and consumer leases, subagents reuse it instead of starting duplicates, and a finishing subagent closes it only when it started the instance and no consumers remain.
- A node cleans up only services, ports, fixtures, temporary directories, and worktrees proven to be node-owned, non-shared, and unused by later consumers. It never terminates shared Codex, MCP, CodeGraph, Docker, or other-task resources.
- Users may set a timebox or choose no time limit. By default, each implementation-node closeout, each parallel batch loop, and the final overall closeout receive 60 minutes. Rotating Devs within a stage does not reset that clock; expiry produces `STOPPED_INCOMPLETE`, never a false completion claim.

## Installation and invocation

```bash
mkdir -p ~/.agents/skills
git clone https://github.com/lcw363/subagent-delivery.git \
  ~/.agents/skills/subagent-delivery
```

Explicit invocation is the most reliable:

```text
Use $subagent-delivery to complete openspec/changes/add-order-refund, and keep only key conclusions in the main conversation.
```

After installation, Codex may also select the skill automatically when a task matches its description.

## Workflow

1. Lock requirements, acceptance criteria, tests, and release boundaries.
2. Build a dependency DAG from independently verifiable stages. Reuse a Dev across adjacent sequential stages in the same requirement, call chain, and ownership scope; rotate only for a domain shift, a high-risk boundary, or a context-health failure.
3. The Dev performs the minimal implementation, conditional TDD, focused tests, code simplification, self-review, and a stage checkpoint.
4. At a planned checkpoint in a long stage, evaluate context health and keep reusing the Dev while healthy. Prepare a handoff on the first observable compaction; rotate only on the second observable compaction, a failed health check, a domain shift, or a distinct high-risk boundary. The fresh Dev remains read-only until verification; the original node and deadline remain unchanged.
5. Run applicable regression/compile/build checks at sequential-stage or integrated-batch closeout. Add real HTTP parameter testing for API changes.
6. After the sequential chain is fixed, the main task creates one final independent Reviewer. Findings return to the single writer for fixes, a new checkpoint, and re-review until pass or timebox closeout. Node closeout records owned-resource cleanup or retention.

For example, even with checkpoints for stages C/D, shared files, an unfrozen contract, or non-isolated databases or fixtures force sequential execution. A worktree or integration preflight failure also falls back to sequential execution while preserving recovery evidence.

## Example

```text
Use $subagent-delivery to deliver the 10 confirmed Wayfinder development stages:
- infer dependencies and keep dispatching the next task; stay sequential unless modules are fully independent;
- create an unpushed local checkpoint commit after each stage;
- call new endpoints on the local service with a test account and real order parameters;
- run a final independent review and close actionable P0-P3 findings in the current delivery; list out-of-scope issues as residual risks and unfinished work if the timebox expires.
```

See [SKILL.md](./SKILL.md) for the complete rules, [context-rotation.md](./references/context-rotation.md) for context rotation, [checkpoint-and-recovery.md](./references/checkpoint-and-recovery.md) for recovery, [model-routing.md](./references/model-routing.md) for model policy, and [evidence-and-briefs.md](./references/evidence-and-briefs.md) for briefs and test evidence.
