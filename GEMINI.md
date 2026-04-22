# Gemini CLI — ebay-deals

Instructions in this file and the referenced **Cursor rules** are foundational for work in this repository. Prefer the `.mdc` files as canonical; this file is the Gemini-specific entry point.

## Project rules (mandatory)

Gemini MUST follow these rules (same content Cursor loads from `.cursor/rules/`):

- [.cursor/rules/ebay-deals-overview.mdc](.cursor/rules/ebay-deals-overview.mdc) — Product, stack, architecture, and repository layout.
- [.cursor/rules/ebay-deals-conventions.mdc](.cursor/rules/ebay-deals-conventions.mdc) — Supabase, database, environment variables, and eBay integration conventions.
- [.cursor/rules/ebay-deals-git-workflow.mdc](.cursor/rules/ebay-deals-git-workflow.mdc) — Versioning, branching, commits, and pull requests.

## Spec and archive

- [PLAN.md](PLAN.md) is the full implementation plan (schema, scoring, cron, file tree).
- [agent-rules-old/](agent-rules-old/) is an archived copy of rules from a different project; do **not** treat it as authoritative for ebay-deals.
