# Schema Design — Worked Examples

Concrete answers a contract manufacturer of dietary supplements (anonymized
as **ExampleCo**) gave to each of the seven decision frameworks during a
real catalog migration. Use as a reference when answering the same question
for another entity.

These are *worked examples*, not normative defaults. Another contract
manufacturer may legitimately answer differently — these examples show what
"answered" looks like, not what to copy.

## ExampleCo at a glance

- Industry: functional gummy contract manufacturing
- Regulation: 21 CFR Part 111 (dietary supplement CGMP)
- Scale at the time of migration: ~200 product templates, ~190 vendor partners, ~100 active SKUs
- Deployment: Odoo Online, single company

## §1 — Multi-company strategy

| Decision | Single company "ExampleCo" |
|---|---|
| Why | Today only this entity uses this database. Sister entities (if any) live elsewhere. |
| Future | Multi-company single-database becomes the likely answer once shared customer overlap materializes between sister entities. A sister cannabinoid business is the candidate for separate-DB if/when sold. |
| Inter-entity transactions | None today. Brand customers are external partners. |

**Reusable lesson:** start single-company unless you have signed inter-entity
transactions today. Adding a second company to an existing DB is cheaper than
splitting an over-modeled DB later.

## §2 — Chart of accounts

| Decision | l10n_us localization template, augmented (not replaced) |
|---|---|
| Source system | n/a — fresh start in Odoo |
| Custom additions | Pending — surfaced during SDR backfill |
| Migration of historical entries | None — only opening balances loaded at go-live |

**Reusable lesson:** l10n_us is the right baseline for any US entity. Never
delete localization-supplied accounts; add to them.

## §3 — Product modeling: variants vs separate products

| Pattern | When ExampleCo uses it | Example |
|---|---|---|
| **Separate products per customer per SKU** | Default for contract manufacturing — every brand-specific finished good is its own template | `[BX-VC-60] Brand X Vitamin C 60ct` ≠ `[BY-VC-60] Brand Y Vitamin C 60ct` |
| **Concentrations as separate products** | Adopted for actives where the % is the key buying decision | `Astaxanthin 2%`, `Astaxanthin 2.5%`, `Astaxanthin 3%`, `Astaxanthin 5%` are 4 templates, not 1 with variants |
| Concentration as variant | Rejected — would muddy BOM / cost / supplier relationships | n/a |
| **Size variants** | For an own-brand SKU — likely accepted for `OWN-SLEEP 30ct vs 60ct` if BOMs scale linearly | Pending |

**Reusable lesson:** for a CMO, the default of "separate products per
customer per SKU" almost always wins. Variants pull customers + SKUs
together in reporting in ways that make traceability harder, not easier.

For ingredient catalogs, treat **functional concentration as identity**, not
as a variant axis. A 2% Astaxanthin and a 5% Astaxanthin have different
BOM contributions, different vendor pricing, and different specs. They are
different products.

## §4 — BOM strategy

| Decision | Use normal Manufacturing BOMs, not Subcontracting routes |
|---|---|
| Why | The CMO is the *subcontractor* in the brand's supply chain. Subcontracting routes in Odoo are for the *contracting company's* DB. In the CMO's own DB, the work is normal manufacturing. |
| Customer-owned materials | Pattern A (default): `stock.quant.owner_id` on receipts. Pattern B (dedicated CustomerOwned location) reserved for customers with commingling concerns. |
| BOM levels | Multi-level where intermediate WIP is stocked/tested/held (slab → cut → bottled). Flat for single-pass productions. |

**Reusable lesson:** the "are we the subcontractor or the contractor in this
relationship?" question must be answered explicitly in the SDR. Getting it
wrong leads to inventory in confusing virtual locations and PO/MO mechanics
fighting the physical process.

## §5 — Warehouse and location hierarchy

| Decision | Multiple warehouses + many internal locations |
|---|---|
| Warehouses | A primary warehouse plus building/process-specific warehouses (e.g. an off-site drying floor as its own warehouse) |
| Notable internal locations | Drying area (with rack-level sub-locations), R&D Lab, Pre-Production, Post-Production, Packing Zone, Quality Control |
| Quarantine location | **Initially missing — flagged as remediation.** GMP requires a real quarantine location, not just a status. |
| Customer-owned segregation | Not initially configured |

**Reusable lesson:** physical buildings are usually warehouses; process steps
within a building are usually locations. A multi-warehouse posture needs an
explicit rationale recorded in the SDR — it may be right, but it's unusual
enough that "we did it this way" is not sufficient.

For any GMP-regulated entity, a Quarantine location is required, not
optional. FDA expects you to demonstrate physical/logical separation of held
material from released material.

## §6 — Lot/serial tracking

| Product class | Tracking mode | Source |
|---|---|---|
| Raw materials | `lot` (mandatory) | Executed |
| Finished goods | `lot` (mandatory) | Executed |
| WIP / intermediate | `lot` (recommended) | Pectin Blend A is the case in point |
| Packaging materials | `none` (default off) | Defer until a customer requires it |
| Serial | `none` | Not applicable to gummies |

**Reusable lesson:** lot tracking imposes real friction at the warehouse
floor (every receipt, every move, every output requires a lot value). Apply
it where regulation or recall capability requires it; do not apply it
universally.

The lot-tracking flip pattern (Option A: zero → flip → restore w/ OPENING
lot) is documented in `odoo-data-migration/references/lot-tracking-flip-runbook.md`.

## §7 — External ID convention

| Convention | ExampleCo's choice |
|---|---|
| Namespace prefix | `__import__` for the initial migration; `__exampleco__` for ongoing automation |
| Format | `<namespace>.<entity>_<stable_business_key>` |
| Examples | `__import__.ing_ascorbic_acid_vendora`, `__exampleco__.product_template_brandx_vitaminc_60ct`, `__exampleco__.partner_customer_brandx` |
| Anti-pattern caught | Bare `partner_<slug>` collides with `product_<slug>` because Odoo External IDs are unique cross-model. Always include the entity in the ID. |

**Reusable lesson:** External IDs are the spine of every idempotent migration.
Decide the convention before writing one record. Once you have records under
two competing conventions, you have a duplicate-detection problem on top of
your migration.

## Vendor consolidation pattern

Not one of the seven frameworks, but a pattern used heavily in this
migration. Encoded as a reusable template:

When the existing partner list has multiple records for what's actually one
vendor company (e.g., 8 separate rows for the same vendor from different
sales contacts):

1. Identify the canonical company partner (or create one if none is a clean
   company-shaped record)
2. For each duplicate contact-shaped partner: set `is_company=False`,
   `supplier_rank=0`, `parent_id=<canonical company id>`
3. The contact retains its name and links to the parent company
4. Subsequent supplier transactions reference the company partner; existing
   PO history stays attached to whichever record was used

ExampleCo consolidated 14 contacts under their parent companies and created
42 new vendor companies in this pass.

## UoM duplicate cleanup

The existing `uom.uom` table had two `lb` records (different IDs, same name).
Decision: archive the duplicate (`active=False`). Never delete — `uom_id`
foreign keys exist across the catalog. Archive is invisible to most reports
but preserves historical references.

**Reusable lesson:** UoM duplicates are a frequent side effect of
QuickBooks/NetSuite migrations and of Odoo demo data. Archive duplicates;
never delete.
