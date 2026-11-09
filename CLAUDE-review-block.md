# Block for CLAUDE.md: Codex reviews every diff

Paste the section below into `~/.claude/CLAUDE.md` (global) or the project's `CLAUDE.md`. Model and effort live in `~/.codex/config.toml`: `model = "gpt-6.1-sol"`, `model_reasoning_effort = "xhigh"`.

```markdown
### Cross-model review before commit

Every diff is reviewed by Codex before it is committed. Workflow:

1. Diff complete -> run `/codex:review` (for large diffs `/codex:review --background`, then `/codex:status` and `/codex:result` to collect it).
2. Codex returns no findings, or a list of issues with severities.
3. Issues marked `critical` or `high` -> fix -> re-run the review (max 3 rounds).
4. Still failing after 3 rounds -> stop and report to me: what is left and why it does not converge.
5. `medium` / `low` findings -> fix if cheap, otherwise log them to the project TODO.md.

For design-heavy diffs (new API surface, new schema, new auth flow, new business rule)
also run `/codex:adversarial-review` before declaring done.

Model and effort for every review come from `~/.codex/config.toml` (`gpt-6.1-sol`, `xhigh`). Never lower them.

When a Codex run finishes or is abandoned, make sure no `codex` process is left running:
`pgrep -fl "codex (review|exec|app-server)"` and kill leftovers.
```
