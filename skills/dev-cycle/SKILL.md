---
name: dev-cycle
allowed-tools: Bash, Read, Grep, Glob, Agent, AskUserQuestion
description: "Run this skill when you want one existing codebase feature or slice carried through its effective engineering workflow. Resume from durable repo state, delegate each phase to the installed skill in a fresh subagent, and stop only for real engineer decisions, blockers, or explicit steering."
---

## Output style (plain words, no dashes, no hyphens)

<!-- OUTPUT-STYLE:START -->
Write everything this skill produces, files and messages alike, in plain simple language. Talk to the reader as `you`, warm and direct like a colleague, and present every step as a recommendation they may run or skip, never an order. Keep technical terms that carry real meaning; explain each in plain words. Never use a dash or a hyphen as punctuation: no em dash, no en dash, and no hyphenated compounds. Write `read only`, not `read-only`. Say it in simple words, or reword the sentence. Code, file paths, command flags, and values other skills match on keep their hyphens. Use short sentences, commas, or parentheses. Clear beats clever.
<!-- OUTPUT-STYLE:END -->

## What this skill does

Coordinates one feature or development slice in an existing codebase.

It owns no engineering phase and writes no project artifact. Each phase runs in a fresh subagent that loads and follows the installed skill for that phase. The coordinator only resolves state, chooses the next phase, verifies the handoff, carries real engineer decisions, and routes repair work.

The repository is the source of truth. A child report is evidence to inspect, not state to trust by itself.

This version is for existing codebases. Do not use it to bootstrap a new product or choose an initial stack.

## Invocation consent

Running `/dev-cycle <target>` is the engineer choosing to carry that target through the effective workflow tier unless they steer otherwise.

That choice preapproves routine workflow control only:

1. run the next stage in the tier
2. rerun a stage invalidated by a repair
3. accept the routine `mark done?` choice when the tier closing stage has passed

Do not ask the engineer again for those routine choices.

This consent does not cover product requirements, architecture choices, scope changes, optional discovery or critique that a child skill explicitly leaves to the engineer, contradictory durable state, privileged actions, or external irreversible actions.

Explicit steering always wins. If the engineer says to skip a stage, stop at a stage, or change the target, follow that instruction.

## Model guard

Astra is forbidden for this skill and every agent it starts, directly or indirectly.

Never request `gpt-6-astra`, an Astra alias, or a model choice described as Astra. This rule overrides any child skill instruction to use a stronger model, another model, a preferred model, or a stage specific model.

On Codex, keep the current session model for delegated work when the client controls subagent models. If explicit model selection is required, use `gpt-6-luna`. A child that cannot continue without Astra must stop and report `BLOCKED`; it must never select Astra as a fallback.

Pass this rule to every child and require each child to pass it to every descendant.

## Boundaries

Never perform phase work in the coordinator when an installed skill owns it.

Never edit code, scope, specs, tests, reviews, documentation, or `AGENTS.md` from the coordinator.

Never commit, push, merge, publish, release, deploy, or perform another irreversible external action unless the engineer explicitly asked for that action outside this skill.

Never rerun work durable state already records as complete unless later work invalidated it.

Do not copy the parent conversation into a child. Pass only the target, the skill, explicit steering, and any engineer answer the child needs.

## Subagents

Subagent support is required. If the client cannot delegate, stop and say this skill needs subagent support.

Use one fresh subagent per top level stage. A stage may use its own subagents exactly as its installed skill instructs, except for model choices that conflict with the Model guard.

All stage agents work in the same repository and current working directory as the coordinator.

Use this compact prompt:

```text
Run the installed <skill> skill for <target>.

Read and follow that skill's SKILL.md exactly. It is authoritative for this stage.
Use durable repo artifacts and the current working tree as the source of truth.

MODEL GUARD: Astra is forbidden for this run and every descendant agent. Never request `gpt-6-astra`, an Astra alias, or Astra as a stronger or alternate model. On Codex, keep the current session model when the client controls subagent models. If explicit model selection is required, use `gpt-6-luna`. Pass this rule unchanged to every descendant.

The engineer invoked /dev-cycle. Routine continuation through the effective workflow tier, routine reruns after repair, and the routine mark done choice at the tier closing stage are already approved. Do not use that approval for product decisions, architecture choices, scope changes, optional discovery or critique, privileged actions, or irreversible external actions.

Steering from the coordinator:
<only information this stage needs, or "none">

If you need a real engineer decision, ask the engineer directly when the client supports it. Otherwise return NEEDS_USER with the exact question, recommendation, and options. Do not choose for them.

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
4. Read `git status`.
5. Resolve the effective workflow tier from the feature override, else the project default.
6. Carry any explicit engineer steering for this run.

If the target is genuinely ambiguous, ask one short question.

If an existing codebase has no usable root `AGENTS.md`, dispatch `/audit`.

If the target is not enrolled, dispatch `/scope <target>`.

Do not rerun `/scope` merely to start a cycle for an enrolled target.

## Dispatch loop

After every successful stage, resolve current state again from the repo and choose the first stage still needed.

| Durable state | Dispatch |
|---|---|
| project context missing | `/audit` |
| target not enrolled | `/scope <target>` |
| load bearing decision owed | `/architect <target>` |
| build incomplete | `/develop <target>` |
| `Alpha`, `Beta`, or `GA`, verify not complete | `/check verify <target>` |
| `Beta` or `GA`, tests not complete | `/test <target>` |
| `GA`, review not complete | `/check review <target>` |
| `GA`, documentation not complete | `/document <target>` |
| selected cycle stages satisfied | `/sync` |
| sync complete, no blocking item | complete |

`Prototype` stops its normal phase sequence after `/develop`. `Alpha` adds verify. `Beta` adds tests. `GA` adds review and documentation.

The tier selects the normal automated sequence for this invocation. It does not make the workflow compulsory outside this invocation, and the engineer may skip or add stages at any time.

Treat a child's `NEXT` as a routing hint. Recheck durable state before following it.

## Evidence gate

A child saying `STATUS: COMPLETE` is not enough. Before advancing, verify the smallest durable evidence that the stage owns.

Use stage owned evidence when it exists:

1. `audit`: usable `AGENTS.md`
2. `scope`: the target is enrolled
3. `architect`: the governing decision is recorded, or durable state now says no spec is needed
4. `develop`: the claimed code change exists and the target's build progress changed as the skill owns
5. `check verify`: the claimed verification result is reflected in its owned scope or `verify.md` state
6. `test`: the claimed tests exist, and `Test it` changed when the skill says it passed
7. `check review`: the claimed review artifact exists
8. `document`: the claimed document exists, and `Document it` changed when applicable
9. `sync`: the reconciliations it claims are visible in the owned durable files

Do not demand evidence a stage does not own.

If the report says complete but its claimed durable evidence is missing, do not advance. Rerun that stage once with the exact mismatch. If the same mismatch remains, stop and report the blocker.

## Repair routing

Follow the classification from the skill that found the problem, then return to the dispatch loop.

Typical routes:

1. undecided load bearing choice goes to `/architect`
2. missing or incomplete implementation goes to `/develop`
3. broken implemented behavior goes to `/debug`
4. review findings that need code changes go to `/develop`

After code changes during this invocation, treat downstream verification already run in this invocation as invalidated and rerun the affected stages selected by the tier.

Do not restart earlier phases that the repair did not invalidate.

If the same gate fails twice for materially the same reason with no meaningful new repo evidence, stop for the engineer.

## Human decisions

The goal is to remove message carrying, not engineer judgment.

Pause only for a real engineer choice or blocker, such as:

1. product requirements or preferences that cannot be inferred
2. an architecture choice the owning skill requires the engineer to decide
3. optional discovery, research, or critique that the owning skill requires separate consent for
4. a scope change
5. an irreversible or unusually privileged action
6. contradictory durable artifacts
7. a repeated blocker after the repair limit

If a child cannot ask directly, relay its exact question, recommendation, and options. Once answered, resume that child when possible, otherwise start a fresh replacement child with the answer.

Do not pause just because a stage completed.

## Completion

The cycle is complete when the tier selected stages have either completed or been explicitly skipped, final `/sync` completes, and no blocking engineer decision remains.

Return only the useful summary. Durable files hold the detail.

```text
## /dev-cycle complete

<target> completed through <tier>.

Completed: <stages actually run>
Skipped: <stages explicitly skipped, or none>
Resumed from: <first stage this invocation needed>
Needs you: <nothing, or one remaining nonblocking item>
```
