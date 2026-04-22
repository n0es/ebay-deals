# ebay-deals

Persistent project context for AI assistants (Claude Code, Cursor, and similar tools) lives in **Cursor rules** under `.cursor/rules/`. That folder is the canonical source; this file is a short index so tools that auto-load `CLAUDE.md` still land in the right place.

| Rule | Topics |
|------|--------|
| [ebay-deals-overview.mdc](.cursor/rules/ebay-deals-overview.mdc) | Product, stack, target architecture, repo layout (current + planned) |
| [ebay-deals-conventions.mdc](.cursor/rules/ebay-deals-conventions.mdc) | Supabase/DB, env vars, eBay API, RLS, shared code patterns |
| [ebay-deals-git-workflow.mdc](.cursor/rules/ebay-deals-git-workflow.mdc) | Versioning in `package.json`, branches, commits, PRs |

**Related docs:** [PLAN.md](PLAN.md) is the full implementation spec. Archive of rules from another project: [agent-rules-old/](agent-rules-old/) (not authoritative for this repo).

Gemini CLI: follow [GEMINI.md](GEMINI.md), which points at the same rules.
