# Opus Fable Codex

Claude Code orchestration skill built for a split-model workflow:

- **Opus / Fable** — head, decisions, issue-spec, acceptance
- **Codex GPT-5.5 through the Claude plugin** — read-only scout, repository reading, diff analysis, code review without changes
- **Sonnet** — coding executor, `gh` / CLI, tests, commits
- **Fresh Sonnet** — DoD verifier and adversarial reviewer
- **Haiku** — disabled

The skill command is:

```text
/opus-fable-codex
```

## Pipeline

```text
Codex read-only scout
→ Opus/Fable issue-spec
→ Sonnet executor
→ fresh Sonnet DoD verifier
→ fresh Sonnet adversarial review
→ Opus/Fable acceptance
```

This setup is useful when Codex works reliably for investigation and review through the official Claude Code plugin, while repository writes are delegated to Sonnet.

## Manual installation into one project

Clone this repository, then copy the skill folder into the target project:

```bash
mkdir -p .claude/skills
cp -R skills/opus-fable-codex .claude/skills/
```

Expected path:

```text
.claude/skills/opus-fable-codex/SKILL.md
```

Reload VS Code or start a new Claude Code session, then run:

```text
/opus-fable-codex Continue the current task.
```

## Routing rules

Codex is explicitly read-only in this skill:

- no `--write`;
- no file modifications;
- no commits or pushes;
- no GitHub mutations;
- no dependency installation.

Sonnet is the only implementation worker. Verification and adversarial review must run in fresh Sonnet contexts, separate from the executor.

## Safety

- no push without explicit user approval;
- no reset, clean, or destructive git operation without approval;
- no dependency installation without approval;
- no auto-closing GitHub Issues before final acceptance;
- no Haiku.
