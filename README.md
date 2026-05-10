# Odoo Skills

A research-backed, composable family of [Claude Code skills](https://docs.claude.com/en/docs/claude-code/skills) for connecting business data to Odoo, structuring Odoo correctly along the way, and getting humans to actually use it.

Anchored to the canonical hard-mode use case — **GMP-regulated contract manufacturing of dietary supplements** — but the decision frameworks generalize to any company implementing or remediating Odoo.

Version: **0.2.1**

## Why this exists

Most Odoo implementations fail not because Odoo can't do the work, but because:
- The schema is decided in flight, ad-hoc, with no decision record
- A SPoC is never named, so decisions queue forever
- Historical data is imported uncritically and ages into noise
- Lot tracking, quarantine, and CoA workflows are bolted on after go-live
- Training is treated as a final-week concern instead of the third pillar

This family encodes the methodology that prevents those failures, and the technical patterns that make the implementation reproducible and idempotent.

## Design philosophy

1. **Methodology first.** The OIM principles (Standard > Studio > Custom; Replace, don't replicate; SPoC, not committee; avoid importing data history; satisfaction is not a KPI) are non-negotiable. They run through every skill.
2. **Schema first, data second.** The wrong Odoo schema produces months of cleanup. `odoo-schema-design` runs before any sync skill touches a record.
3. **Idempotency is mandatory.** Every write to Odoo must be safely re-runnable. External IDs are the spine.
4. **Existing data is the default case.** Most projects have an Odoo running already. Skills handle "merge / dedupe / migrate-in-place," not just "fresh load."
5. **Deployment-portable.** Same skill works against Odoo Online, Odoo.sh, and self-hosted.
6. **GMP compliance is regulatory, not optional.** For supplement / food entities, lot tracking, quality checks, and CoA workflows are 21 CFR Part 111 / 117 requirements.
7. **Adoption is built in.** A working Odoo no one uses is a failure. Training is part of every project, not an afterthought.

## The eight skills

| Skill | Purpose | Status |
|---|---|---|
| **odoo-implementation-methodology** | OIM phases, SPoC role, anti-patterns. Runs first on any project. | Stable |
| **odoo-schema-design** | Decision frameworks for COA, products/variants, BOMs, locations, multi-company. | Stable |
| **odoo-training-and-adoption** | Role-based training, internal SOPs, change management. | Stable |
| **odoo-data-migration** | CSV/JSON bulk import, dedup, dry-run, rollback, lot-tracking flips. | Working spec |
| **odoo-connect** | Auth + RPC client setup. MCP, JSON-RPC, XML-RPC. | Working spec |
| **odoo-api-sync** | Idempotent upsert, retry, batching. | Stub |
| **odoo-webhooks** | Event-driven sync via base_automation, outbound/inbound hooks. | Stub |
| **odoo-gmp-compliance** | 21 CFR Part 111/117 config: lot, quality checks, CoA workflows. | Partial |

## How they compose

```
                       ┌──────────────────────────────────────┐
                       │  odoo-implementation-methodology     │  Phase 1-4 framework
                       │  (the gating skill — runs first)     │  SPoC, OIM phases
                       └────┬─────────────────────────────┬───┘
                            │                             │
                ┌───────────▼──────────┐    ┌─────────────▼──────────┐
                │  odoo-schema-design  │    │ odoo-training-adoption │
                │  (SDR producer)      │    │ (SOP, training, change)│
                └───────────┬──────────┘    └────────────────────────┘
                            │
                ┌───────────▼──────────┐
                │   odoo-connect       │  (working RPC client)
                └───────────┬──────────┘
                            │
            ┌───────────────┼───────────────┐
            │               │               │
   ┌────────▼────────┐ ┌────▼─────────┐ ┌──▼────────────┐
   │ data-migration  │ │   api-sync   │ │   webhooks    │
   │  (one-shot CSV) │ │ (ongoing RPC)│ │ (event-driven)│
   └────────┬────────┘ └──────┬───────┘ └───────┬───────┘
            │                 │                 │
            └─────────────────┼─────────────────┘
                              │
                  ┌───────────▼─────────────┐
                  │  odoo-gmp-compliance    │  (regulated overlay)
                  └─────────────────────────┘
```

## Installation

These skills are designed to be loaded by Claude Code via the [skills system](https://docs.claude.com/en/docs/claude-code/skills).

### Per-project install

```bash
git clone https://github.com/<your-org>/odoo-skills.git ~/.claude-projects/odoo-skills
# Or clone elsewhere and symlink the individual skill dirs into your project's .claude/skills/
ln -s ~/.claude-projects/odoo-skills/skills/odoo-schema-design \
      <your-project>/.claude/skills/odoo-schema-design
```

### Global install

```bash
git clone https://github.com/<your-org>/odoo-skills.git ~/odoo-skills
for s in ~/odoo-skills/skills/*/; do
  ln -s "$s" ~/.claude/skills/$(basename "$s")
done
```

Each skill's `SKILL.md` frontmatter includes the trigger conditions Claude Code uses to invoke it. The `description` field is the main signal — it tells Claude when this skill is relevant.

## Quick-start for a new Odoo project

1. **Start with `odoo-implementation-methodology`.** Produce a GAP analysis. Name a SPoC. Don't skip this.
2. **Then `odoo-schema-design`.** Walk the seven decision frameworks. Produce an SDR.
3. **In parallel, `odoo-training-and-adoption`.** Plan the change-management runbook and SPoC pathway.
4. **`odoo-connect` once.** Set up authenticated RPC.
5. **`odoo-data-migration` for the initial load.** Use the 7-phase plan template.
6. **`odoo-gmp-compliance` if regulated.** Lot strategy + quality control points.
7. **`odoo-api-sync` and `odoo-webhooks` as ongoing integration needs surface.**

## Versioning convention

- Each skill carries a `VERSION` file (semver)
- Family-level releases are tagged on the repo
- Bump rules:
  - **patch** for typos, small clarifications
  - **minor** for new decision frameworks, references, or scripts
  - **major** for breaking changes to triggers or output shape
- The `name:` field inside SKILL.md frontmatter is unversioned so it stays stable when installed

## Status & roadmap

The 8-skill family is at **v0.2.1**. Three skills are methodology-grade and stable; two have working specs grounded in a real implementation; three remain stubs awaiting a real first use case.

Roadmap to v0.3:

| Skill | Trigger for next bump |
|---|---|
| `odoo-data-migration` | Battle-test against a non-supplement-CMO data shape |
| `odoo-connect` | Reference clients in Python and Node, ready to paste |
| `odoo-gmp-compliance` | A complete CoA workflow built and documented |
| `odoo-training-and-adoption` | A SPoC pathway taught and validated end-to-end |
| `odoo-api-sync` | A real first integration (Shopify / Amazon / etc.) |
| `odoo-webhooks` | A real first event-driven flow |

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md). PRs welcome, especially:
- Worked examples from non-supplement industries (food CGMP, cannabis, cosmetics, B2B ingredients)
- Filled-out reference content for the stubs
- Battle-tested edge cases for the data-migration patterns

## License

[MIT](./LICENSE) — see file for full text.

## Acknowledgments

Methodology grounded in Odoo S.A.'s own Implementation Methodology (Catherine Vieslet, Head of Business Services). See `research/research-bibliography.md` for the full source list.
