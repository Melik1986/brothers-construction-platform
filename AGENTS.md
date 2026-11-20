# AGENTS.md

Rules for coding agents working in this repository.

## Quality gates

- Toolchain is pinned in `mise.toml` (Node.js, pnpm, hk, rumdl, typos, betterleaks, lychee). Install with `mise install`.
- Run `mise run check` (alias for `hk check --all`) before considering a change done. CI runs the same command and must pass.
- Git hooks are managed by **hk** (`hk.pkl`). Do not add Husky, Lefthook, lint-staged or other hook managers.
- Commit messages must follow Conventional Commits (enforced by the `commit-msg` hook).
- `hk fix --all` applies safe autofixes (Markdown formatting, whitespace, EOF newlines). Review the diff afterwards.

## Rules

- Never edit generated files by hand; regenerate them instead.
- Do not disable lint/format rules or add broad ignores to make a check pass. Exceptions must be narrow (one path + one rule) with a written reason, as in `.rumdl.toml` and `_typos.toml`.
- Never commit secrets or credentials. `betterleaks` scans staged files and the full tree; treat any finding as blocking.
- Documentation is the product spec right now: keep Markdown style consistent with `.rumdl.toml` rules.
