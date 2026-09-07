---
name: superpowers-with-matt-tdd
description: Use only when explicitly invoked by the human to write implementation plans or execute approved work with Superpowers and Matt's tdd skill.
---

# Superpowers with Matt TDD

## Overview

Preserve Superpowers' Spec -> Plan -> Execute -> Review -> Verify -> Finish discipline while adapting implementation planning and TDD:

1. Implementation plans use larger semantic tasks.
2. Execution uses Matt's `tdd` strategy.

Everything not explicitly overridden here follows the original Superpowers skill.

## Writing Plans: Detailed Content, Larger Tasks

**REQUIRED SUB-SKILL:** Use `superpowers:writing-plans`.

Do not split work mechanically into 2-5 minute actions. Each task should be a semantically complete, verifiable implementation increment suitable for one commit. A strong coding agent may complete several naturally related steps inside one task.

Plans still specify goals and constraints, affected files, interfaces and dependencies, key implementation requirements, verification, and commit boundaries.

Do not create separate tasks merely for:

- writing a failing test;
- verifying RED;
- writing the minimal implementation; or
- verifying GREEN.

Those are internal steps of one implementation task. Use 5-15 minutes only as a rough scale; semantic, dependency, verification, and commit boundaries take precedence over time.

### Test Seams

For tasks requiring TDD, identify the valuable test seams: which public interfaces verify which behaviors. Do not plan tests for every internal function or implementation detail.

Approval of the full plan also approves its listed test seams.

## Execution: Use Matt TDD

Whether execution uses `superpowers:executing-plans` or `superpowers:subagent-driven-development`, preserve that executor's workflow.

**REQUIRED SUB-SKILL:** Use `tdd`, replacing `superpowers:test-driven-development` for implementation tasks.

Within each task:

- test public behavior rather than implementation details;
- test only at approved seams;
- proceed in vertical slices: one behavior test -> minimal implementation -> next behavior;
- observe RED before GREEN;
- avoid duplicate tests for private helpers, internal collaborators, or trivial layers merely for coverage; and
- leave broader refactoring to review rather than expanding the RED -> GREEN loop.

Do not re-confirm test seams already approved in the plan. If implementation requires a material new seam not present in the approved plan, confirm that seam before adding it.

## Common Mistakes

- Splitting RED and GREEN into separate plan tasks.
- Planning tests for every internal function instead of valuable public behaviors.
- Re-confirming test seams already approved in the plan.
- Writing all tests before implementing any behavior.
- Replacing an executor's review or verification workflow rather than only its TDD strategy.

## Boundaries

This skill changes implementation task granularity and TDD strategy. It does not change brainstorming's question cadence or design approval behavior, and it does not choose between single-agent execution and subagent-driven development.

It does not replace writing-plans, executing-plans / subagent-driven-development, requesting-code-review, systematic-debugging, verification-before-completion, or finishing-a-development-branch. Outside the overrides above, follow Superpowers unchanged.

This skill can be used independently or together with `superpowers-with-checkpoint-brainstorming`.
