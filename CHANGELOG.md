# Changelog

All notable changes to this project are documented here. Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/); the project uses semver at the family level (in addition to per-skill `VERSION` files).

## [0.2.1] — Initial public release

First public-facing release. Five of eight skills are methodology-grade or have working specs; three remain stubs.

### Skills at this release

| Skill | Status | Notes |
|---|---|---|
| `odoo-implementation-methodology` | Stable | Full OIM encoding |
| `odoo-schema-design` | Stable | Seven decision frameworks |
| `odoo-training-and-adoption` | Stable | Role-based pathways + SOP system |
| `odoo-data-migration` | Working spec | 7-phase plan + lot-tracking flip runbook from a real migration |
| `odoo-connect` | Working spec | MCP / JSON-RPC / XML-RPC patterns |
| `odoo-gmp-compliance` | Partial | Lot strategy matrix + Receiving QA pattern; CoA workflow + recall + Part 11 still stubbed |
| `odoo-api-sync` | Stub | Awaiting first real integration |
| `odoo-webhooks` | Stub | Awaiting first real event-driven flow |

### What's new vs internal pre-release

- Worked examples grounded in a real catalog migration (~190 storable products, ~150 vendor pricelist rows, lot-tracking remediation)
- Lot-tracking flip Option A runbook (zero → flip → restore w/ OPENING lot)
- Vendor consolidation pattern
- Odoo 18 schema-compatibility notes (`uom_po_id` removed, etc.)
- Cross-model External-ID uniqueness gotcha documented
- Rescue / retroactive-backfill case study for projects that skipped Phase 1

### Sources

Methodology references in `research/research-bibliography.md`.

## Pre-public history

Prior versions (0.1.0, 0.2.0) shipped privately and are not in this repo's git history. They informed the structure but had not been sanitized for public release.
