---
name: dev-cycle
allowed-tools: Bash, Read, Grep, Glob, Agent, AskUserQuestion
description: "Run this skill after scope and any required architecture for one existing codebase feature or slice have been manually reviewed and accepted. Carry the accepted work through implementation, verification, testing, repair, review, documentation, sync, and completion. Never scope or architect work from this skill."
---

## Output style (plain words, no dashes, no hyphens)

<!-- OUTPUT-STYLE:START -->
Write everything this skill produces, files and messages alike, in plain simple language. Talk to the reader as `you`, warm and direct like a colleague, and present every step as a recommendation they may run or skip, never an order. Keep technical terms that carry real meaning; explain each in plain words. Never use a dash or a hyphen as punctuation: no em dash, no en dash, and no hyphenated compounds. Write `read only`, not `read-only`. Say it in simple words, or reword the sentence. Code, file paths, command flags, and values other skills match on keep their hyphens. Use short sentences, commas, or parentheses. Clear beats clever.
<!-- OUTPUT-STYLE:END -->

## What this skill does

Coordinates the execution part of one already designed feature or development slice in an existing codebase.

The engineer owns scope and architecture outside this skill. Before `/dev-cycle <target>` starts, the target must already be enrolled in scope and every required load bearing design decision must already be recorded and accepted. If either condition is missing, stop and hand control back to the engineer. Never dispatch `/scope` or `/architect` from this skill.

Once those design gates are satisfied, this skill coordinates implementation, verification, testing, repair, review, documentation, sync, and completion according to the effective workflow tier.

It owns no engineering phase and writes no project artifact. Each phase runs in a fresh subagent that loads and follows the installed skill for that phase. The coordinator only resolves durable state, chooses the next execution phase, verifies the handoff, carries allowed engineer decisions, and routes non design repair work.

The repository is the source of truth. A child report is evidence to inspect, not state to trust by itself.

This version is for existing codebases with accepted scope and design. Do not use it to bootstrap a new product, choose an initial stack, define feature scope, or make architecture decisions.

## Design gate

The design gate must pass before any execution stage runs.

The target is ready only when all of these are true:

1. a matching scope entry exists
2. the scope is specific enough to identify the accepted work
3. every required architecture or spec decision is recorded and accepted
4. the scope does not show an unresolved design step that must happen before implementation

If the target is not enrolled, stop and recommend that the engineer run `/scope <target>` manually.

If a load bearing design decision is still owed, stop and recommend that the engineer run `/architect <target>` manually.

If scope explicitly records that no architecture step is needed, that satisfies the architecture part of this gate.

If implementation, verification, testing, or review discovers a new scope or architecture question, stop the cycle and hand control back to the engineer. Do not answer that question inside `dev-cycle`, and do not dispatch a design skill automatically.

## Invocation consent

Running `/dev-cycle <target>` means the engineer has already reviewed the target's scope and required architecture and is choosing to carry that accepted work through the effective execution tier unless they steer otherwise.

That choice preapproves routine execution control only:

1. run the next non design stage in the tier
2. rerun a non design stage invalidated by a repair
3. accept the routine `mark done?` choice when the tier closing stage has passed

Do not ask the engineer again for those routine choices.

This consent does not cover product requirements, architecture choices, scope changes, optional discovery or critique that a child skill explicitly leaves to the engineer, contradictory durable state, privileged actions, or external irreversible actions.

Explicit steering always wins. If the engineer says to skip a stage, stop at a stage, or change the target, follow that instruction.

## Model policy

The normal authoring model for this skill is `gpt-6-luna`.

Use `gpt-6-luna` for implementation and repair work, including `/develop`, `/debug`, and any descendant agent that authors or changes code.

Use `gpt-6-sol` for code review and critique work, including `/check review` and any descendant agent whose job is to review code rather than author it.

For other non review stages, use `gpt-6-luna` unless the engineer explicitly requested another model before invoking this skill.

If any child asks which model should perform a review, answer `gpt-6-sol` without asking the engineer. The reason is already established: code is authored with `gpt-6-luna`, and reviews use `gpt-6-sol`.

Do not automatically escalate to `gpt-6.1-sol`, Astra, or another stronger or more expensive model. A stronger model may be used only when the engineer explicitly invokes or requests it outside the automatic model policy.

Astra is forbidden for this skill and every agent it starts, directly or indirectly. Never request `gpt-6-astra`, an Astra alias, or a model choice described as Astra.

Pass this model policy to every child and require each child to pass it to every descendant. If an installed child skill cannot operate without violating this policy, stop and report `BLOCKED`.

## Boundaries

Never perform phase work in the coordinator when an installed skill owns it.

Never edit code, scope, specs, tests, reviews, documentation, or `AGENTS.md` from the coordinator.

Never dispatch `/scope` or `/architect` from this skill.

Never make or silently infer a new product, scope, or architecture decision. If execution reveals one, stop at the design gate and return control to the engineer.

Never commit, push, merge, publish, release, deploy, or perform another irreversible external action unless the engineer explicitly asked for that action outside this skill.

Never rerun work durable state already records as complete unless later execution work invalidated it.

Do not copy the parent conversation into a child. Pass only the target, the skill, explicit steering, and any engineer answer the child needs.

## Subagents

Subagent support is required. If the client cannot delegate, stop and say this skill needs subagent support.

Use one fresh subagent per top level stage. A stage may use its own subagents exactly as its installed skill instructs, except for choices that conflict with the Design gate or Model policy.

All stage agents work in the same repository and current working directory as the coordinator.

Use this compact prompt:

```text
Run the installed <skill> skill for <target>.

Read and follow that skill's SKILL.md exactly. It is authoritative for this stage.
Use durable repo artifacts and the current working tree as the source of truth.

DESIGN BOUNDARY: /dev-cycle begins only after scope and required architecture are manually accepted. Do not run /scope or /architect, and do not make a new scope or architecture decision. If this stage discovers one, return NEEDS_USER and say whether the engineer should return to scope or architecture.

MODEL POLICY: Code authoring and repair use gpt-6-luna. Code review and critique use gpt-6-sol. If asked which model should review, use gpt-6-sol without asking the engineer. Do not automatically use gpt-6.1-sol, Astra, or another stronger model. Pass this policy unchanged to every descendant.

The engineer invoked /dev-cycle. Routine continuation through the effective execution tier, routine reruns after repair, and the routine mark done choice at the tier closing stage are already approved. Do not use that approval for product decisions, architecture choices, scope changes, optional discovery or critique, privileged actions, or irreversible external actions.

Steering from the coordinator:
<only information this stage needs, or "none">

Before asking the engineer about a non design execution question, check whether the governing spec, matching feature scope, relevant AGENTS.md, or explicit steering already answers it unambiguously. If so, use that recorded answer and continue.

Do not ask the engineer to choose a review model. The review model is gpt-6-sol.

If a real non design engineer decision remains, ask directly when the client supports it. Otherwise return NEEDS_USER with the exact question, recommendation, and options. If the unresolved question changes scope or architecture, return NEEDS_USER and identify that design boundary instead of choosing for the engineer.

When finished, report:
STATUS: COMPLETE | BLOCKED | NEEDS_USER
ARTIFACTS: files or durable state changed
NEXT: the next skill or action you recommend, if any
SUMMARY: a compact result
```

Prefer a fresh child without inherited chat turns when the client supports it.

## Resolve current state

For `/dev-cycle <target>`:

1. Read root `AGENTS.md` if present.
2. Locate only the scope entry that contains the target under `docs/scope/` or `.workflow/scope/`.
3. Follow that entry to its governing spec when one exists.
4. Confirm the Design gate is satisfied.
5. Read `git status`.
6. Resolve the effective workflow tier from the feature override, else the project default.
7. Carry any explicit engineer steering for this run.

If the target is genuinely ambiguous, ask one short question.

If an existing codebase has no usable root `AGENTS.md`, stop and recommend that the engineer run `/audit` manually before starting `dev-cycle`.

If the target is not enrolled, stop and recommend `/scope <target>`. Do not dispatch it.

If a required architecture decision is missing or unresolved, stop and recommend `/architect <target>`. Do not dispatch it.

## Dispatch loop

After every successful stage, resolve current state again from the repo, recheck the Design gate, and choose the first execution stage still needed.

| Durable state | Action |
|---|---|
| project context missing | stop, recommend manual `/audit` |
| target not enrolled | stop, recommend manual `/scope <target>` |
| load bearing decision owed | stop, recommend manual `/architect <target>` |
| build incomplete | `/develop <target>` |
| `Alpha`, `Beta`, or `GA`, verify not complete | `/check verify <target>` |
| `Beta` or `GA`, tests not complete | `/test <target>` |
| `GA`, review not complete | `/check review <target>` |
| `GA`, documentation not complete | `/document <target>` |
| selected execution stages satisfied | `/sync` |
| sync complete, no blocking item | complete |

`Prototype` stops its normal execution sequence after `/develop`. `Alpha` adds verify. `Beta` adds tests. `GA` adds review and documentation.

The tier selects the normal automated execution sequence for this invocation. It does not make the workflow compulsory outside this invocation, and the engineer may skip or add non design stages at any time.

Treat a child's `NEXT` as a routing hint. Recheck durable state and the Design gate before following it.

## Evidence gate

A child saying `STATUS: COMPLETE` is not enough. Before advancing, verify the smallest durable evidence that the stage owns.

Use stage owned evidence when it exists:

1. `develop`: the claimed code change exists and the target's build progress changed as the skill owns
2. `check verify`: the claimed verification result is reflected in its owned scope or `verify.md` state
3. `test`: the claimed tests exist, and `Test it` changed when the skill says it passed
4. `check review`: the claimed review artifact exists
5. `document`: the claimed document exists, and `Document it` changed when applicable
6. `sync`: the reconciliations it claims are visible in the owned durable files

Do not demand evidence a stage does not own.

If the report says complete but its claimed durable evidence is missing, do not advance. Rerun that stage once with the exact mismatch. If the same mismatch remains, stop and report the blocker.

## Resolve before escalating

A child `NEEDS_USER` report is provisional for non design questions. Before asking the engineer, try to resolve the question from authoritative durable state.

Check only the sources that can govern this target, in this order:

1. the governing spec
2. the matching feature scope
3. the relevant `AGENTS.md`
4. explicit steering already given for this invocation

Use a recorded answer only when it resolves the child's exact question unambiguously and does not conflict with another governing artifact.

If the question is which model should perform code review, answer `gpt-6-sol` and continue. Do not escalate that question to the engineer.

If the answer to another non design question is recorded, resume the same child when possible, otherwise start a fresh replacement child with that answer. Tell the child the answer came from durable state.

Do not treat prior chat summaries, guesses, conventions, or a recommendation alone as an engineer decision.

If the unresolved question changes product requirements, scope, or architecture, stop the cycle. Tell the engineer whether the work needs manual `/scope` or manual `/architect` before `dev-cycle` can resume.

Escalate other questions when the governing artifacts are silent, contradictory, or the owning non design skill explicitly requires fresh consent from the engineer.

A deterministic mismatch is not a human decision. If the correct state is objectively established by current repo evidence, such as a stale count, status, pointer, or generated record, route the mismatch to the installed non design skill that owns that artifact or reconciliation. Do not edit it in the coordinator. After the owner runs, apply the evidence gate again.

If ownership is unclear, inspect the installed skills' ownership rules and route to the non design owner. If resolving ownership requires a scope or architecture choice, stop and return control to the engineer.

## Repair routing

Follow the classification from the skill that found the problem, then return to the dispatch loop.

Typical routes:

1. a scope change stops the cycle and returns to manual `/scope`
2. an undecided load bearing choice stops the cycle and returns to manual `/architect`
3. missing or incomplete implementation goes to `/develop`
4. broken implemented behavior goes to `/debug`
5. review findings that need code changes go to `/develop` or `/debug` according to the finding
6. deterministic durable drift goes to the non design skill that owns the artifact or reconciliation

After code changes during this invocation, treat downstream verification already run in this invocation as invalidated and rerun the affected non design stages selected by the tier.

Do not restart scope or architecture from inside this skill. Do not restart earlier execution phases that the repair did not invalidate.

If the same gate fails twice for materially the same reason with no meaningful new repo evidence, stop for the engineer.

## Human decisions

The goal is to remove message carrying from execution, not engineer judgment from design.

A product, scope, or architecture decision always exits the cycle. Report the exact issue and recommend the appropriate manual design skill. Resume `dev-cycle` only after the resulting durable artifacts are accepted.

For non design work, pause only after the Resolve before escalating gate leaves a real engineer choice or blocker, such as:

1. optional discovery, research, or critique that the owning skill requires separate consent for
2. an irreversible or unusually privileged action
3. contradictory durable artifacts that cannot be reconciled mechanically
4. a repeated blocker after the repair limit

Review model selection is not an engineer choice inside this skill. Use `gpt-6-sol`.

If a child cannot ask a permitted non design question directly, relay its exact question, recommendation, and options. Once answered, resume that child when possible, otherwise start a fresh replacement child with the answer.

Do not pause just because a stage completed.

## Completion

The cycle is complete when the tier selected non design stages have either completed or been explicitly skipped, final `/sync` completes, the Design gate still holds, no blocking engineer decision remains, and no known deterministic artifact mismatch remains that an installed non design skill can reconcile.

Decision debt may remain only when it genuinely belongs to later scope or architecture and does not invalidate the completed feature. Report it briefly and do not resolve it from this skill.

Do not declare `/dev-cycle complete` while also handing the engineer a mechanical cleanup that the non design workflow can resolve itself. Route that cleanup, verify the durable result, then complete.

Return only the useful summary. Durable files hold the detail.

```text
## /dev-cycle complete

<target> completed through <tier>.

Completed: <non design stages actually run>
Skipped: <non design stages explicitly skipped, or none>
Resumed from: <first execution stage this invocation needed>
Needs you: <nothing, or one remaining nonblocking design debt item>
```
