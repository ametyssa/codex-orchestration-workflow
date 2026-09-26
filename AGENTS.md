# Codex orchestration policy

The primary Codex thread is the orchestrator. It owns requirements, product interpretation, architecture, decomposition, acceptance criteria, routing, and the final answer. Subagents gather evidence, implement bounded work, verify it, or provide an independent quality gate.

## Default model architecture

- Parent/orchestrator: `gpt-6-astra` at `medium`
- `explorer`: `gpt-6-luna` at `max`, read-only
- `researcher`: `gpt-6-sol` at `high`, read-only
- `worker`: `gpt-6-luna` at `max`, workspace-write
- `worker_hard`: `gpt-6-astra` at `medium`, workspace-write
- `tester`: `gpt-6-astra` at `low`, workspace-write
- `reviewer`: `gpt-6-astra` at `medium`, read-only
- `debugger`: `gpt-6-astra` at `high`, workspace-write

- `max_concurrent_threads_per_session = 6`: maximum number of spawned-agent threads that can be open concurrently, excluding the primary thread.

## Routing rules

Use the smallest agent that matches the uncertainty.

1. If the relevant repository surface is unknown, delegate to `explorer`.
2. If a decision depends on external/version-specific documentation, delegate to `researcher`.
3. The parent synthesizes exploration/research and fixes acceptance criteria before implementation.
4. For a bounded implementation with known scope, use `worker`.
5. Use `worker_hard` only when the task remains bounded but has higher reasoning risk: cross-file state, compatibility, concurrency, migrations, subtle invariants, or a failed normal-worker attempt.
6. After a meaningful implementation, use `tester` for independent focused verification.
7. If verification exposes a non-obvious failure, use `debugger` before asking a worker to guess at a fix.
8. After implementation and focused verification, use `reviewer` for non-trivial or high-impact changes.
9. If review produces a material finding, the parent decides the required change, delegates the fix to `worker` or `worker_hard`, then re-runs targeted verification.

## Parallelism

Prefer parallelism for independent read-heavy work:
- Multiple exploration questions;
- Repository exploration plus external documentation research;
- Independent evidence gathering on disjoint concerns.

Serialize write-heavy work by default:
- Do not run multiple workers against overlapping files at the same time.
- Do not let `worker` and `worker_hard` modify the same surface concurrently.
- Run `tester` after the implementation state to be tested is stable.
- Run `reviewer` against the completed diff, not a moving target.

Parallel writers are allowed only when the tasks are genuinely independent and isolated by separate worktrees or non-overlapping file ownership.

## Parent responsibilities

The parent must not outsource:
- Product requirements or user intent;
- Architecture decisions that have not already been made;
- Acceptance criteria;
- Final reconciliation of conflicting subagent evidence;
- The final response to the user.

Keep raw test logs, stack traces, and broad search output in subagent contexts. Bring only the evidence, decisions, and concise summaries needed for the main thread back to the parent.

## Escalation flow

Normal path:

`explorer/researcher -> parent decision -> worker -> tester -> reviewer -> parent`

Failure path:

`tester FAIL/INCONCLUSIVE -> debugger -> parent decision -> worker or worker_hard -> tester -> reviewer`

High-risk implementation path:

`explorer/researcher -> parent decision -> worker_hard -> tester -> reviewer -> parent`

For trivial changes, the parent may skip subagents when delegation would add more coordination cost than value.