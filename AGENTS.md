# Agent instructions (ebay-deals)

This repository uses **`.cursor/rules/*.mdc`** as the source of truth for how to work in this codebase. Read those rules when making non-trivial changes; they are kept in sync with [PLAN.md](PLAN.md).

## Where to look

| File | Role |
|------|------|
| `.cursor/rules/ebay-deals-overview.mdc` | What the app is, stack, architecture, layout |
| `.cursor/rules/ebay-deals-conventions.mdc` | Database, env, eBay, Supabase, RLS |
| `.cursor/rules/ebay-deals-git-workflow.mdc` | Version bumps, Git and PR workflow |

## Entry points by tool

- **Claude Code / `CLAUDE.md` loaders:** [CLAUDE.md](CLAUDE.md) points here.
- **Cursor:** Project rules in `.cursor/rules/` (many rules use `alwaysApply: true`).
- **Gemini CLI:** [GEMINI.md](GEMINI.md).

If instructions conflict, follow the user’s explicit request for the current task, then the `.mdc` rules, then this file.
