---
name: dev-cycle
allowed-tools: Bash, Read, Grep, Glob, Write, Edit, Agent, AskUserQuestion
description: "Run /dev-cycle to carry one existing-codebase feature or slice through the installed engineering workflow using subagents. Coordinates /scope, /architect, /develop, /check, /test, /document, /sync, and /debug without replacing any of them. Resumes from durable repo state and asks the engineer only when an underlying skill requires a real decision."
---

## Output style (plain words, no dashes, no hyphens)

<!-- OUTPUT-STYLE:START -->
Write everything this skill produces, files and messages alike, in plain simple language. Talk to the reader as `you`, warm and direct like a colleague, and present every step as a recommendation they may run or skip, never an order. Keep technical terms that carry real meaning; explain each in plain words. Never use a dash or a hyphen as punctuation: no em dash, no en dash, and no hyphenated compounds. Write `read only`, not `read-only`. Say it in simple words, or reword the sentence. Code, file paths, command flags, and values other skills match on keep their hyphens. Use short sentences, commas, or parentheses. Clear beats clever.
<!-- OUTPUT-STYLE:END -->

## What this skill does

Coordinates the existing engineering workflow for one feature or development slice in an existing codebase.

This skill owns no development method of its own. It does not redesign, summarize, or replace `/scope`, `/audit`, `/architect`, `/develop`, `/check`, `/test`, `/document`, `/sync`, or `/debug`. Each stage is delegated to a fresh subagent that must load and follow the installed skill for that stage.

The coordinator owns only:

1. finding the target and current durable state
2. choosing the next incomplete workflow stage
3. delegating that stage to a fresh subagent
4. carrying real user decisions back to the stage that asked for them
5. following explicit handoffs from one installed skill to another
6. stopping when the required workflow is complete or the engineer is genuinely needed

The repository is the source of truth. Chat summaries are not.

This first version is for existing codebases. Do not use it to bootstrap a brand new product or choose an initial stack.

## Hard boundaries

Never perform a stage's work in the coordinator when the corresponding installed skill exists.

Never substitute a home grown architecture, scoping, implementation, review, testing, documentation, synchronization, or debugging process.

Never answer a question that an underlying skill says belongs to the engineer.

Never commit, push, merge, publish, release, deploy, or perform another irreversible external action unless the engineer explicitly asked for that action outside this skill.

Never rerun a stage that durable repo state already shows as complete unless a later stage invalidated it.

Do not hand the full parent conversation to a stage when the stage can recover context from the repo. Give it the target, the named skill, and any specific steering or user answer it needs.

## Subagents

Subagent support is required. If the client cannot delegate to subagents, stop and say that `/dev-cycle` needs subagent support.

Use one fresh subagent for each top level stage. The stage subagent may use its own subagents exactly as its installed skill instructs. Do not flatten or suppress nested delegation.

Do not override a stage skill's model choices. A spawned stage agent may inherit the current model unless the installed skill itself requires a different model for one of its internal checks.

All stage agents work in the same repository and current working directory as the coordinator.

### Stage prompt contract

For every stage, send a compact prompt with this shape:

```text
Run the installed <skill> skill for <target>.

Read and follow that skill's SKILL.md exactly. It is authoritative for this stage.
Do not replace or reinterpret its workflow.
Use the repository's durable artifacts and current working tree as the source of truth.

Steering from the coordinator:
<only information this stage actually needs, or "none">

If the skill requires a decision that belongs to the engineer, ask the engineer directly when your client supports that. Otherwise return NEEDS_USER with the exact question and options. Do not choose for them.

When the stage is finished, report:
STATUS: COMPLETE | BLOCKED | NEEDS_USER
ARTIFACTS: files or durable state changed
NEXT: the next skill or action this skill recommends, if any
SUMMARY: a compact result
```

On Codex, prefer a fresh subagent without inherited chat turns when the spawn interface supports it. The explicit stage prompt plus repository state should carry the work. On another Agent Skills client, use the closest available fresh subagent behavior.

## Start or resume

Given `/dev-cycle <target>`:

1. Read root `AGENTS.md` if present.
2. Locate the relevant scope file under `docs/scope/` or `.workflow/scope/`.
3. Locate any linked spec for the target.
4. Read only enough of those artifacts plus `git status` to determine the current workflow stage.
5. If the target is ambiguous, ask one short clarifying question and stop until answered.
6. If there is no usable root `AGENTS.md` for an existing codebase, delegate `/audit` before continuing.
7. If the target is not enrolled in scope, delegate `/scope <target>`.
8. If the target is already enrolled, do not rerun `/scope` merely to begin the cycle.
9. Resume from the first incomplete required stage.

Treat durable artifact status as stronger evidence than a previous agent's prose summary.

## Workflow

Follow the installed skills and the target's recorded workflow tier.

### Design gate

If the scope says the target needs a spec and no governing build spec exists, delegate `/architect <target>`.

If `/develop` says a load bearing decision is still owed, delegate `/architect` with the exact decision it surfaced, then return to a fresh `/develop` subagent.

If the scope says no spec is needed, do not invent an architecture stage.

### Build

Delegate `/develop <target>`.

The develop skill owns its own gates, implementation method, build plan, self checks, scope advancement, and any assumed decision behavior.

### Verification tail

Read the effective workflow tier from the scope. A per feature override wins over the project default.

Run only the tail required by that tier:

| Tier | Required tail after `/develop` |
|---|---|
| `Prototype` | none |
| `Alpha` | `/check verify <target>` |
| `Beta` | `/check verify <target>` then `/test <target>` |
| `GA` | `/check verify <target>` then `/test <target>` then `/check review <target>` then `/document <target>` |

Do not add extra stages merely because they seem prudent. The installed scope skill owns the tier policy.

### Sync

After the required tail is complete, delegate `/sync`.

If `/sync` reports decision debt, stale architecture, or a context gap, follow the skill it explicitly names. Do not silently rewrite the affected artifact in the coordinator.

## Failures and repair loops

Follow the handoff named by the skill that found the problem.

Examples:

* If `/develop` routes to `/architect`, run `/architect`, then retry `/develop`.
* If `/check verify` routes a behavioral failure to `/debug`, run `/debug`, then rerun the verification stages affected by the fix.
* If `/check verify` says implementation is incomplete rather than broken, return to `/develop`, then rerun the required verification tail.
* If `/test` surfaces a product defect, follow its named repair path, then rerun the affected verification stages.
* If `/check review` records findings that require code changes before the cycle can finish, return those findings to a fresh `/develop` subagent, then rerun the required checks that the change may have invalidated.

Do not invent a repair route when the reporting skill gives one.

If the same gate fails twice for materially the same reason with no meaningful new repo evidence, stop and ask the engineer rather than burning another loop.

## Human decisions

The goal is to remove message carrying, not to remove the engineer from decisions.

Pause for the engineer when an installed skill requires:

* product requirements or preferences that cannot be inferred
* a load bearing architecture choice the skill presents for confirmation
* permission for optional discovery, cross model review, or another action the skill explicitly leaves to the engineer
* a scope change
* an irreversible or unusually privileged action
* resolution of contradictory durable artifacts
* a repeated blocker after the repair limit above

If a child cannot ask the engineer directly, relay its question without answering it. Once the engineer answers, resume that same child when the client supports follow up. Otherwise start a replacement subagent with the answer plus the same stage target.

Do not pause merely because a stage finished successfully.

## Completion

The cycle is complete when:

1. the target's required build stage and verification tail are complete for its effective workflow tier
2. the final `/sync` run completes or reports only items that the installed workflow explicitly treats as nonblocking debt
3. no required stage is still asking for engineer input

Return one compact report:

```text
## /dev-cycle complete

**<target> completed through <tier>.**

Completed: <stages actually run>
Resumed from: <first stage this invocation needed>
Needs you: <nothing, or the one remaining nonblocking item>
```

Do not repeat every subagent report. The durable files hold the detail.
