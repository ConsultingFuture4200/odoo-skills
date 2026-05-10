# Research Bibliography & Audit

This document records the research that informed the current release of the
skill family. Future maintainers should treat this as a starting point for
re-audit, not as a permanent source of truth — Odoo evolves quickly.

## Methodology

Search waves conducted: Odoo official methodology, contract manufacturing
patterns, OCA modules, FDA 21 CFR Part 11 / 111 / 117, training and change
management, current data import patterns. Bias toward sources from 2024
onward; older sources only when they cover stable mechanics (e.g., External
ID semantics).

## Canonical sources

### Odoo official

- **Odoo Implementation Methodology (OIM)** — Catherine Vieslet, Head of Business Services, Odoo S.A. The book and SlideShare. Source of the SPoC role, the "Replace, don't replicate" principle, the "Standard > Studio > Custom" hierarchy, the four-phase structure (GAP Analysis → Kick-off → Implementation → Go-Live), and the "avoid importing data history" guidance. <https://www.slideshare.net/slideshow/odoo-implementation-methodology-188219992>
- **Odoo 19 documentation: Subcontracting** — three workflows (basic, resupply, dropship). Definitive source for what subcontracting in Odoo *is* (the contracting company's view, not the subcontractor's). <https://www.odoo.com/documentation/19.0/applications/inventory_and_mrp/manufacturing/subcontracting.html>
- **Odoo 19 documentation: Export and import data** — confirms External ID semantics, including the constraint that External IDs must be unique across all models, not just within one. <https://www.odoo.com/documentation/19.0/applications/essentials/export_import_data.html>
- **Odoo Experience 2018: Is Odoo GxP Compliant?** — Odoo's own talk on Part 11 compliance posture. Confirms that Odoo *can be configured* to comply but doesn't claim out-of-the-box compliance. <https://www.odoo.com/event/odoo-experience-2018-1206/track/is-odoo-gxp-compliant-with-the-pharmaceutical-industry-standard-1234>
- **Odoo Experience 2021: User adoption — no project success without change management** — establishes the three pillars of OIM, with change management as one of them. <https://www.odoo.com/event/odoo-experience-2021-2847/track/user-adoption-there-is-no-project-success-without-change-management-4456>
- **learn.odoo.com (Odoo Learn / eLearning)** — official self-paced training. Source for the SPoC training pathway. <https://www.odoo.com/slides>
- **Odoo Functional Certification (v18/v19)** — 120-question / 1.5h exam at 70% pass. Useful as a "what does Odoo think a competent functional user knows" benchmark. <https://www.odoo.com/slides/odoo-19-functional-certification-502>

### OCA (Odoo Community Association)

- **OCA "Must Have" Base Modules list** — vetted by OCA working group. Starting point for any installation. <https://www.odoo-community.org/list-of-must-have-oca-modules>
- **mgmtsystem (Quality Management System) module family** — Quality Manual, Procedures, Audits, Nonconformities, CAPAs, Improvement Opportunities, Employee Training. Core to the GMP compliance skill. Repo: <https://github.com/OCA/management-system>
- **quality_control_oca / quality_control_mrp_oca** — generic quality test infrastructure beyond Odoo's stock Quality module. Repo: <https://github.com/OCA/manufacture>
- **OCA ecosystem in general** — the answer to many "Odoo doesn't do X out of the box" problems is "OCA has a module for X." Always check OCA before commissioning a custom development.

### FDA / regulatory

- **21 CFR Part 11** (eCFR) — electronic records and electronic signatures. <https://www.ecfr.gov/current/title-21/chapter-I/subchapter-A/part-11>
- **FDA Part 11 Scope and Application guidance** — FDA's risk-based interpretation; reduces the practical compliance burden vs. a literal reading. Important: clarifies that Part 11 applies to records used to satisfy *other* CFR parts (such as Part 111 for supplements), not as a standalone requirement. <https://www.fda.gov/regulatory-information/search-fda-guidance-documents/part-11-electronic-records-electronic-signatures-scope-and-application>
- **21 CFR Part 111** — Current Good Manufacturing Practice in Manufacturing, Packaging, Labeling, or Holding Operations for Dietary Supplements.
- **21 CFR Part 117** — Current Good Manufacturing Practice and Hazard Analysis and Risk-Based Preventive Controls for Human Food.

### Community and forum sources

- **Odoo forum thread: receiving customer-supplied raw materials without inventory accounting impact** — establishes two valid patterns: (a) `stock.quant.owner_id` field on receipts (lighter-weight, native), and (b) dedicated customer-owned location with internal-transfer disabled. The skill family documents both. <https://www.odoo.com/forum/help-1/how-to-receive-goods-from-customer-for-subcontracting-jobs-and-not-counted-on-inventory-accounting-in-odoo16-221934>
- **Odoo forum: External IDs must be unique across all models** — the import gotcha that breaks naive id schemes. Drives the namespacing convention used throughout.

### Industry / partner blog sources (used with skepticism — many are SEO content from Odoo partners)

- Tatvamasi Labs, TeleNoc, Globalteckz, Eezee-IT — all converge on similar phased methodology with SPoC. Useful for triangulation; not authoritative.
- Greg Moss / OdooClass.com — long-running independent training resource. Useful for technical depth.

## Open audit questions

- Does `learn.odoo.com` have a structured curriculum that should be embedded directly into the training skill, vs. just linked?
- Are there relevant OCA modules for specific verticals (cannabis Metrc integrations, cosmetics serialization, B2B ingredients) that deserve their own subskill?
- How does Odoo 19's AI feature set change schema design (e.g., AI-powered chart of accounts mapping)?

## Maintenance protocol

When a major Odoo version drops (annually, October):

1. Re-read the migration guide and changelog
2. Verify subcontracting and customer-owned inventory patterns haven't changed
3. Re-audit the OCA modules listed (some are abandoned each cycle)
4. Update version-specific references in skill bodies
5. Bump the family minor version
