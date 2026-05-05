---
description: Optional guardrails for Cloudflare R2 operations
---

# Cloudflare R2 DevOps Guardrails

Optional prompt template for Cloudflare R2 operations. Type `/r2-guardrails` in Pi to invoke.

Note: This is an OPTIONAL prompt template, not auto-injected. For mandatory rules that must always apply, put them in `skills/*/SKILL.md` instead.

## Rules

1. Always verify the API token has R2 permissions before making R2 API calls.
2. Use the fixed Account ID from the environment; do not guess.
3. Present bucket/object lists in clean markdown tables.
4. Warn the user if a bucket deletion would affect public resources (e.g., public.r2.dev).
5. Report failures with the full error message and suggest remediation steps.
