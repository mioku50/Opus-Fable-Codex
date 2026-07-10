---
name: opus-fable-codex
description: Issue-first Claude Code orchestration where Opus or Fable is the head, Codex GPT-5.5 through the Claude plugin is a read-only scout and reviewer, Sonnet performs implementation and repository operations, and fresh Sonnet agents verify DoD and run adversarial review.
disable-model-invocation: true
---

# Opus Fable Codex

A Claude Code orchestration skill for environments where the official OpenAI Codex plugin is useful for deep read-only investigation and review, while Sonnet has reliable write access to the working repository.

## Role map

```text
Opus / Fable
= head, decisions, issue-spec, acceptance

Codex GPT-5.5 through the Claude plugin
= read-only scout
= repository reading
= diff analysis
= code review without changes

Sonnet
= coding executor
= gh / CLI
= tests
= commits

Fresh Sonnet
= DoD verifier
= adversarial review

Haiku
= do not use
```

## Core principle

The head spends tokens only on judgment.

Opus or Fable:
- understands the user's goal;
- resolves architecture, product, UX, and scope decisions;
- converts facts into a self-contained issue/spec;
- defines boundaries and acceptance criteria;
- accepts, rejects, or blocks the result after independent verification.

The head does not:
- deeply scout the repository itself;
- write implementation code;
- perform routine git, gh, or shell operations;
- treat an executor report as final acceptance;
- delegate unresolved judgment to a worker.

Workers provide facts and execution. The head makes decisions.

## Strict routing

### Head: Opus or Fable

Use for:
- goal interpretation;
- architecture and UX decisions;
- resolving contradictions discovered by scouts;
- writing or updating the issue/spec;
- deciding scope and boundaries;
- final acceptance after verifier and adversarial-review reports.

If Fable is unavailable, limited, or removed from the subscription, the current Opus model becomes the head with the same rules.

### Codex GPT-5.5: read-only scout and reviewer

Use the official OpenAI Codex plugin for:
- codebase mapping;
- reading files and contracts;
- locating routes, components, state, and data flow;
- reading git status and diffs;
- objective risk discovery;
- diff analysis;
- code review without changes.

Codex must be explicitly dispatched as read-only.

Codex must not:
- edit, create, delete, or rename files;
- run with `--write`;
- commit or push;
- install dependencies;
- change configuration;
- perform gh mutations;
- make product or architecture decisions;
- act as executor in this skill.

Preferred Claude plugin route:

```text
subagent_type: "codex:codex-rescue"
model: gpt-5.5
request: explicitly read-only research/review; do not modify files
```

For native review commands, a read-only Codex review route may also be used when appropriate.

If the plugin cannot start, loses authentication, or fails before reading the repository, stop after one retry and fall back to a read-only Sonnet scout for that scout only. Do not spend multiple retries on infrastructure.

### Sonnet: executor and repository operator

Use Sonnet for:
- coding and file edits;
- implementation;
- git and gh operations;
- shell and CLI mechanics;
- typecheck, lint, build, and tests;
- commits;
- focused rework after verification findings.

The executor must use explicit `model: sonnet` and must receive a self-contained issue/spec before changing files.

### Fresh Sonnet: independent verification

Use separate fresh Sonnet contexts for:
1. DoD verification;
2. final adversarial review.

Neither verifier may be the executor that wrote the code.

### Haiku

Do not use Haiku for any role.

## Pipeline

```text
Codex read-only scout
→ Opus/Fable issue-spec
→ Sonnet executor
→ fresh Sonnet DoD verifier
→ fresh Sonnet adversarial review
→ Opus/Fable acceptance
→ next issue
```

Work sequentially when tasks touch overlapping files. Parallel work is allowed only when file ownership is disjoint and the head explicitly confirms the split is safe.

## 1. Preflight

Before a non-trivial task, Sonnet performs repository-state checks:

```bash
git status -sb
git diff --stat
git branch --show-current
```

If the tree is dirty:
- do not reset or clean;
- identify which changes belong to the current task;
- ask the head whether to continue, create a WIP commit, or stash;
- never include secrets, `.env`, generated dumps, or unrelated artifacts in a WIP commit.

Do not push without explicit user approval.

## 2. Codex scouting

Dispatch one or more narrowly scoped read-only Codex scouts.

Good scout questions:
- identify the exact route, component, state, and API chain for a page;
- map the files involved in one feature;
- locate existing patterns that the implementation should reuse;
- inspect the current diff for regressions or duplicated logic;
- verify exact function signatures, props, schemas, and event names.

Avoid prompts such as “study the entire repository and tell me what you think.”

### Codex scout envelope

```text
You are Codex GPT-5.5 acting as a read-only codebase scout.
Working directory: <absolute path>.

Task:
<specific investigation question>

Strict rules:
- Read-only. Do not modify, create, delete, or rename files.
- Do not use a write-capable route.
- Do not commit, push, install dependencies, or mutate GitHub.
- Do not make product or architecture decisions.
- Return facts, contracts, risks, and unknowns with file:line coordinates.

Return:
## Facts
- file:line — fact

## Relevant files
- path — why it matters

## Contracts
- exact function / prop / route / data shape

## Risks
- file:line — objective risk

## Checks run
- command → result

## Unknowns
- what could not be verified and why
```

Scout recommendations are not decisions. Opus/Fable re-evaluates them.

## 3. Issue-first spec

Every non-trivial implementation becomes a self-contained GitHub Issue before executor dispatch.

The issue body is the source of truth. It must be sufficient for Sonnet to implement the task without repeating broad repository research.

If GitHub Issues are unavailable because of proxy, authentication, or `gh` failure, use one of these fallbacks:
- `.claude/scratchpad/specs/<task-name>.md`;
- `docs/dev/specs/<task-name>.md`;
- a complete spec in the current Claude session.

Report briefly: `GitHub is unavailable; using a fallback spec.`

### Issue/spec template

```markdown
# <Task title>

## Goal
One sentence describing what the user will see after completion.

## Context
Exact routes, files, components, current behavior, relevant existing changes, and traps.

## Contract
Exact props, functions, API routes, schemas, CSS classes, states, events, and examples.

## Resolved decisions
- Decision → short rationale.

## Steps
1. Specific change in a specific file.
2. Specific change in a specific file.

## Boundaries
What must not be changed or added.

## DoD + checks
- Observable result.
- Commands to run.
- Expected outcomes.
- What counts as failure.
```

For flows involving two or more components/services, include a fenced Mermaid diagram.

Do not leave unresolved words such as “probably”, “most likely”, or “apparently” in the final spec. Run another scout or make an explicit head decision.

## 4. Sonnet executor dispatch

The executor is a Sonnet subagent with explicit `model: sonnet`.

### Executor envelope

```text
You are the Sonnet coding executor.
Model: sonnet.
Working directory: <absolute path>.
Your source of truth is GitHub issue #N: read it with `gh issue view N`.
If GitHub is unavailable, use this fallback spec: <path>.

Rules:
- Work strictly within the spec boundaries.
- Do not make product or architecture decisions.
- If repository reality contradicts the spec, stop and return a discrepancy report.
- Preserve existing user changes.
- Do not install dependencies without approval.
- Do not touch unrelated files.
- Do not push.

When finished:
1. Run the DoD checks that are available.
2. Create one conventional commit if the spec requires a commit.
3. Use `(#N)` in the commit message when an issue exists.
4. Do not use `closes #N` or `fixes #N` before acceptance.
5. Do not close the issue.
6. Return changed files, checks, deviations, and noticed-but-not-touched items.
```

If continuing partial work, the executor first reads:
- `git status -sb`;
- relevant `git diff`;
- the latest issue/spec and reports.

It must continue without reverting existing changes unless the head explicitly decides otherwise.

## 5. Fresh Sonnet DoD verifier

The verifier is a new Sonnet subagent with no implementation context beyond the issue/spec, final diff, and check results.

### DoD verifier envelope

```text
You are a fresh-context Sonnet DoD verifier.
Model: sonnet.
Do not edit files.

Verify issue #N against the final repository state.
Read:
- the issue/spec;
- git status;
- the diff from <base SHA> to HEAD;
- changed files only, plus narrowly required dependencies.

You must:
- verify every DoD item;
- run available checks from the spec;
- report command → result;
- report file:line → observation;
- return one verdict: passed / failed / not verifiable.

Do not rewrite code.
Do not commit or push.
Do not re-scout the whole repository.
```

`not verifiable` is a valid and honest result.

## 6. Fresh Sonnet adversarial review

Run a second fresh Sonnet context after DoD verification.

### Adversarial-review envelope

```text
You are a fresh Sonnet adversarial reviewer.
Model: sonnet.
Do not edit files.

Review issue #N against the final diff from <base SHA> to HEAD.
Inspect only:
- issue/spec;
- changed files;
- verification results;
- narrowly required dependencies.

Review axes:
1. scope violations;
2. regressions;
3. broken data flow or state assumptions;
4. missing error, loading, and empty states;
5. UI and mobile inconsistencies when relevant;
6. security, permission, and secret-handling risks;
7. code that passes checks but does not satisfy user intent;
8. accidental inclusion of unrelated or generated files.

Return:
- blockers;
- non-blocking concerns;
- passed areas;
- recommendation: accept / revise / manual check required.
```

## 7. Rework policy

If verification fails:
1. first rework goes to the same Sonnet executor with the exact findings;
2. second rework goes to the same Sonnet executor with remaining findings only;
3. after a second failed verification, use a fresh Sonnet executor with the issue/spec and verifier diagnosis;
4. if the fresh executor also fails, stop and mark the task blocked.

Do not ask Codex to apply fixes in this skill. Codex remains read-only.

## 8. Acceptance by Opus/Fable

The head reviews only concise evidence:
- issue/spec;
- executor digest;
- DoD verdict;
- adversarial-review recommendation;
- unresolved manual checks.

Decision rules:
- `passed` + no blocker → accept;
- `failed` → send targeted rework;
- `not verifiable` → request manual verification or another narrow check;
- adversarial blocker → revise before acceptance.

Only after acceptance may the issue be closed or a push be proposed.

## 9. Git and GitHub safety

- Never push without explicit user approval.
- Never reset, clean, or discard a dirty tree without explicit approval.
- Do not include `.env`, keys, tokens, credentials, caches, generated dumps, screenshots, or unrelated artifacts in commits.
- Use conventional commits.
- Link commits with `(#N)` when appropriate.
- Do not use auto-close keywords before final acceptance.
- Do not close issues from an executor or verifier.

## 10. Check order

Run checks in this order when they exist:
1. targeted typecheck;
2. targeted lint;
3. targeted unit tests;
4. broader build/typecheck;
5. manual UI checklist;
6. E2E only if already installed and approved.

Do not install Playwright, Puppeteer, or other dependencies without user approval.

For visual tasks, avoid stealing focus. Prefer existing headless screenshot or browser tooling when available.

## 11. Communication

Keep status updates short:
- done;
- active worker and role;
- blocked reason;
- next step.

Do not paste full worker reports unless the user asks. The head should receive a concise, self-contained digest with the full report path when available.

## 12. Continuation after a limit reset

```text
/opus-fable-codex Continue the current task.

Routing:
- Opus or Fable: head, decisions, issue-spec, acceptance.
- Codex GPT-5.5 through the Claude plugin: read-only scout, repository reading, diff analysis, and code review without changes.
- Sonnet: coding executor, gh/CLI, tests, and commits.
- Fresh Sonnet: DoD verifier and adversarial reviewer.
- Haiku: do not use.

First:
1. run git status -sb and git diff --stat through Sonnet;
2. find the current issue/spec and latest reports;
3. reconstruct completed and pending work;
4. do not revert existing changes;
5. continue the pipeline from the correct stage.

Do not use Codex with --write.
Do not push without my explicit approval.
Do not install dependencies without approval.
```
