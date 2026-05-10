# Contributing

Contributions welcome. This is a small repo with a strong opinion, so a few ground rules:

## What's a good contribution

- **Worked examples from a different industry.** The current worked examples are anchored to dietary supplement CMOs. Cannabis, cosmetics, food (Part 117), B2B ingredients, and Amazon-fulfilled CPG would all benefit from their own §3 (product modeling) and §5 (warehouse/location) examples.
- **Filling out the stubs.** `odoo-api-sync` and `odoo-webhooks` are stubs. If you've built a production-grade Odoo integration with Shopify, Amazon, NetSuite, or any other system, the patterns you used probably belong in one of those.
- **Battle-tested edge cases.** The lot-tracking flip runbook documents the edge cases observed in one migration. There are more. Add them.
- **Odoo version matrix.** Most patterns target Odoo 18. Notes on what changes for 17 and 19 are useful, especially for `odoo-data-migration` gotchas.
- **References.** If a `references/` file is missing (e.g., `customer-owned-inventory-patterns.md` is referenced but not present), filling it in is high-value.

## What's not a good contribution

- **Customizations that violate "Standard before Studio before Custom."** This skill family encodes the OIM doctrine. Custom-module recipes that bypass standard Odoo are out of scope unless they document a justified exception.
- **Generic Odoo tutorials.** Point at `learn.odoo.com` for those — this repo is about implementation methodology and battle-tested patterns, not Odoo basics.
- **Proprietary information.** Do not include real client names, real vendor names, real production data, internal URLs, or anything that would leak a specific company's operations. Anonymize.

## How to contribute

1. Fork the repo.
2. Create a branch: `git checkout -b feat/your-change` (or `fix/`, `docs/`).
3. Make your change. Mirror the existing style — read 2–3 neighboring files before writing new ones.
4. If you're adding a worked example, anonymize it. The pattern is "ExampleCo, a contract manufacturer of dietary supplements."
5. Bump the affected skill's `VERSION` file according to semver:
   - patch for typos
   - minor for new content / references / decision frameworks
   - major for breaking changes to triggers or output shape
6. Open a PR with a short description of what changed and why.

## Style conventions

- Markdown, no emojis
- Tables for comparisons; bullets for lists; prose for reasoning
- Lead with a short purpose statement; no boilerplate intros
- File paths use forward slashes
- Code blocks are language-tagged where it helps highlighting

## Testing your skill changes

If you have an Odoo instance available, the best test is to actually invoke the skill from Claude Code on a real (or test) database. Failing that:

- Read your SKILL.md frontmatter against the [Claude Code skills documentation](https://docs.claude.com/en/docs/claude-code/skills) — the `description` field is what triggers the skill, so it needs to be specific.
- Read your reference files end-to-end as if you were the LLM being handed them — are the steps reproducible? Is anything ambiguous?

## Questions

Open an issue. For broader methodology debates (e.g., "I think the SPoC role should be split into two roles"), open an issue tagged `discussion` first before a PR.
