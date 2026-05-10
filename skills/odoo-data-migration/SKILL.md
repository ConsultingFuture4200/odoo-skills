---
name: odoo-data-migration
description: Use this skill for bulk-loading or migrating data into Odoo via CSV/Excel imports, JSON payloads, or RPC batch writes. Triggers include "import into Odoo", "Odoo CSV import", "load our products into Odoo", "migrate from QuickBooks/NetSuite/Shopify to Odoo", "bulk update Odoo", "Odoo data migration", or any first-load operation. Handles dedup, validation, dry-run, rollback, lot-tracking flips, and external-ID-based idempotency. Should be invoked AFTER odoo-schema-design has produced an SDR. Do NOT use for ongoing event-driven sync (use odoo-api-sync) or one-off small writes (those go through odoo-connect directly).
---

# Odoo Data Migration

This skill turns a Schema Decision Record (SDR) into actual records inside an
Odoo database. The patterns are grounded in a real catalog migration of
~200 products with lot-tracking remediation, a 7-phase plan, and full
idempotent re-runnability.

Use this skill **after** `odoo-schema-design` has produced the SDR. The SDR
gives you the External-ID convention, the category tree, and the variant
strategy. Without that, you are guessing.

## When to use this skill

- First-time bulk load into a fresh Odoo instance
- Augmenting an existing instance with a new dataset (catalog expansion, vendor
  list, opening balances)
- Schema remediation that requires data re-shaping (re-categorize, change
  tracking mode, restructure BOMs)
- Migrating from another system (QuickBooks, NetSuite, Shopify, custom DB)

Do **not** use this skill for:
- Ongoing event-driven sync — `odoo-api-sync` or `odoo-webhooks`
- One-off small writes — `odoo-connect` is enough
- Custom Python module work — out of scope for this skill family

## The 7-phase plan template

This template is generalized from a working migration. Every column on the
right links back to a real artifact you can read.

| Phase | Purpose | Output | Artifact pattern |
|---|---|---|---|
| 0 | Existing-catalog audit | snapshot of current Odoo state | `00_audit_<entity>.json` |
| 1 | Source normalization | canonical rows w/ keys + dedup | `01_parsed_rows.json` |
| 2 | Master data setup | new categories, UoMs, partners, locations | written directly to Odoo |
| 3 | Re-categorize / re-shape existing | batch update plan + execution | `03_recategorize.csv` + `_batch.json` |
| 4 | Update existing matches | per-record update with chatter provenance | `04_updates.json` |
| 5 | Create new records | full creates with returned IDs | `05_creates.json` + `_import.json` |
| 6 | Vendor pricelists / supplierinfo | per-quote rows | `06_supplierinfo*.json` |
| 7 | Lot-tracking flip (if applicable) | dry-run + GREEN/YELLOW/BLOCKED categorization, then migrate | `07_dryrun.json` + `07_migration.json` |

Phase boundaries are gates. Don't run Phase N+1 without verifying Phase N invariants.

**Sequencing rule:** master data before transactional data. Within master data:
companies → COA → partners → categories/UoMs → products → BOMs → opening
inventory → opening balances. Skipping ahead causes foreign-key failures and
re-imports.

## Idempotency: External IDs are the spine

Every record created by this skill must have an External ID. Convention is set
by `odoo-schema-design` §7. A typical pattern: `__import__.ing_<slug>` for
ingredient imports and `__<entity>__.<entity>_<key>` for everything else.

The single most expensive mistake in any migration is not setting External IDs
on creates. Re-running the import then creates duplicate records instead of
updating, and you've now made the cleanup worse than the original problem.

**Rule:** if a record might ever be re-touched by automation, give it an
External ID at create time. Manual UI records get one only if automation will
later reference them.

**Critical constraint:** External IDs are unique across **all models**, not
just within a model. `partner_brandx` and `product_brandx` collide. Always
include the entity in the ID portion (`partner_customer_brandx` vs
`product_template_brandx_vitaminc_60ct`).

## Lot-tracking flip pattern (Option A)

The most-asked-about pattern. Odoo refuses to flip `tracking` from `none` to
`lot` while any internal-location quants exist. Three valid options; Option A
is the recommended default.

| Option | Method | When |
|---|---|---|
| **A** | Zero quants → flip → restore w/ OPENING lot | Default for any go-live where on-hand counts must be preserved exactly |
| B | Scrap-and-receive | When lot history doesn't matter and you'd rather a clean reset |
| C | Stay on `tracking='none'`, accept regulatory risk | Never, for GMP-regulated entities |

### Option A runbook (proven on ~190 storable products)

```
Pre-flight (dry-run):
  1. Query all internal-location quants for the products in scope
     (filter: stock.quant.location_id.usage='internal' — virtual locations
     don't count)
  2. Categorize:
       GREEN   = no on-hand internal stock      → flip in batch
       YELLOW  = has on-hand internal stock     → full migration sequence
       BLOCKED = held by uncanceled MO/picking  → cancel test orders first

Test-order cleanup (if BLOCKED):
  3. Cancel mrp.production.state='draft'/'confirmed' via update_record state='cancel'
  4. Cancel stock.picking.state in {'draft','confirmed','assigned'} via update_record
  5. CASCADE QUIRK: parent cancel does NOT cascade to child stock.move.
     Manually cancel children with update_records.
  6. Re-run pre-flight; BLOCKED should be empty.

GREEN flip:
  7. Single batch update_records({tracking: 'lot'}) on all GREEN templates.
     ~5 seconds for ~100 records.

YELLOW migration (per product):
  8. Snapshot existing quants_data (location_id, quantity, package_id)
  9. Zero each existing stock.quant (quantity=0)
  10. Flip product.template tracking='lot'
  11. Create stock.lot record with name 'OPENING-<sku>-<YYYYMMDD>'
  12. Create new stock.quant rows at original locations with original
      quantities, tagging the new lot_id

Verification:
  13. Spot-check N high-stock products: original qty preserved, lot tag present
  14. Final invariant: count product.template where tracking != 'lot'
      AND is_storable = True. Must equal zero (excluding intentional services).
```

This sequence has been run end-to-end. It is not a thought experiment.
Run order matters — flipping tracking before zeroing quants will fail with
`UserError: You cannot change a tracking mode while there are existing quants`.

## Direct state writes vs workflow methods

Most Odoo state transitions go through workflow methods (`action_cancel`,
`action_confirm`, `button_done`). When using RPC for migrations, you have two
choices:

| Approach | When | Trade-off |
|---|---|---|
| Workflow method (`button_X`) | Production-correct state machine; fires onchange/automations | Slower; some methods aren't RPC-exposed; harder to batch |
| Direct state write (`update_record({state: 'cancel'})`) | Migration-time only; you've audited that no automations need to fire | Fast and batchable; **does not cascade** |

The cascade non-fire is the gotcha. Canceling a `stock.picking` via direct
state write can leave child `stock.move` records still in non-cancel states;
you have to issue a second `update_records` against them. Plan for this:
when using direct state writes on parents, immediately follow with a child
sweep.

**Rule of thumb:** direct writes are fine for cleanup, never for go-live
production cutover.

## Order of operations (canonical)

```
companies (multi-company only)
  → res.currency, res.partner.industry
  → account.account (chart of accounts)
  → res.partner (vendors first, customers second — partner external IDs reused)
  → product.category, uom.uom (taxonomy must exist before products)
  → product.template (master product list)
  → product.product (variants — most created automatically with templates)
  → product.supplierinfo (vendor pricelists — references partner + product)
  → mrp.bom (BOMs — references products)
  → stock.warehouse, stock.location (warehouse tree)
  → stock.quant (opening inventory — last)
  → account.move (opening balances — last of last)
```

Foreign-key violations happen when this order is broken. The Odoo error message
will be unhelpful (`ValueError: Expected singleton`); the fix is almost always
"create the missing parent first."

## Pre-flight validation

Before any write that touches more than ~20 records, dry-run. The validation
phase produces a `*_dryrun.json` artifact that answers:

- How many records would be created vs updated vs skipped?
- For each create: is the External ID unique (cross-model)?
- For each update: does the matching record exist?
- For each foreign key: does the referenced record exist?
- Are there any cross-batch duplicates in the source data?

If the dry-run report doesn't match expectations, do not run the real import.
Iterate the source data instead.

## Dedup matchers per entity

Dedup is the hardest part of any import. Heuristics that work:

| Entity | Match key | Fallback |
|---|---|---|
| `res.partner` (companies) | normalized name + country | VAT/tax ID if present |
| `res.partner` (contacts) | email if present, else `parent_id + name` | manual REVIEW queue |
| `product.template` | `default_code` (SKU) if present | normalized name + category |
| `mrp.bom` | `product_tmpl_id + sequence` | n/a (don't auto-dedup BOMs — flag for human) |
| `product.supplierinfo` | `partner_id + product_tmpl_id + min_qty` | always create new, then dedup post-hoc |

For vendor consolidation (turning multiple "sales contact" partner records
for one vendor company into one company partner with reparented contacts):
14 contact-shaped partner records can be demoted to `supplier_rank=0` and
reparented under newly-created company partners in a single pass. The pattern:
`parent_id` is the company, `is_company=False` on the contact, supplier_rank
moves to the parent.

## Common gotchas (Odoo 18, current as of late 2025)

| Gotcha | Symptom | Fix |
|---|---|---|
| `uom_po_id` removed from `product.template` | `KeyError` on import payload | Drop the field; purchase UoM derives from `uom_id` + `uom_ids` (m2m) |
| Lot-flip blocked by quants in virtual locations | "still has quants" error after zeroing internal | Filter on `location_id.usage='internal'` only; production/inventory/scrap virtuals are not counted |
| External ID collision across models | Cryptic `ValidationError` | Always namespace by entity (`partner_X` vs `product_X`) |
| Cascade-cancel doesn't fire on direct state writes | Child stock.moves left in confirmed state | Always sweep children after parent cancel via direct write |
| `product.supplierinfo.product_uom_id` ignored | "we set kg, but it imports as g" | Confirm `product_uom_id` is in the payload; if missing, Odoo defaults to product's stocking UoM |
| Sample/retail vendor quotes pollute cost basis | `standard_price` set to a $5 retail listing | Filter source data: `is_sample=True` and retail-aggregator vendors → cost basis only or chatter-only, not supplierinfo |
| Vendor extraction from free-text "notes" columns | ~50% of source rows have no parseable vendor | Insist on a structured vendor column in future intake sheets; what you can't extract goes to cost basis only |
| Test orders block lot-flips | Stragglers from QA / smoke testing | Establish a "test orders are canceled before any go-live" convention with the SPoC |

## Cost basis selection rule

When multiple vendor quotes exist for the same product:

```
priority 1: lowest USD/kg from a bulk quote where qty >= 2 kg
            (or qty >= 0.05 kg for high-potency actives)
priority 2: any bulk quote
priority 3: sample-only quote — flag in chatter, treat as provisional
exclude:    retail/aggregator quotes (Amazon, Walmart) → chatter-only
exclude:    rows missing vendor identity → cost basis only, no supplierinfo
```

## Naming conventions

| Asset | Pattern | Example |
|---|---|---|
| Product `name` | `Category - Subtype - Name - Vendor` | `Vitamin - Vitamin C - Ascorbic Acid - VendorA` |
| Product `default_code` (SKU) | `[CUSTOMER-SKU]` for customer products | `[BX-VC-60]` Brand X Vitamin C 60ct |
| External ID | `__import__.ing_<slug>` for ingredients | `__import__.ing_ascorbic_acid_vendora` |
| OPENING lot | `OPENING-<sku>-<YYYYMMDD>` | `OPENING-PEC-CS509-20250509` |
| Test data | prefix `TEST-` | `TEST-Gummy 12.5.25` |

## Outputs of this skill

- A migration runbook (markdown): phase-by-phase plan, rollback strategy
- One JSON/CSV artifact per phase (the `0X_*.json|csv` pattern)
- A dry-run report per write phase (`*_dryrun.json`)
- A post-import audit: counts in vs counts out, sample record verification
- Any flagged REVIEW items in a separate file for human disposition

## Rollback strategy

| Deployment | Rollback approach |
|---|---|
| Odoo Online | Database snapshot before each gate (UI: Settings → Database → Take Snapshot). Restore is full-DB rollback. |
| Odoo.sh | Branch + restore from staging build before promote. Native to .sh. |
| Self-hosted | `pg_dump` before each phase; restore is `pg_restore`. Test the restore at least once. |

Snapshots taken before Phase 3, before Phase 5, before Phase 7 flip. Often
none are needed; the discipline is what makes them not needed.

## Hand-off

This skill produces validated payloads. The actual writes to Odoo go through
`odoo-connect` (which knows about MCP vs JSON-RPC vs XML-RPC, batching, retry).

## References

- `references/import-runbook.md` — phase-by-phase reproduction of a real
  Phase 1–7 import, with exact payload shapes and verification queries
- `references/lot-tracking-flip-runbook.md` — Option A in full, including
  the test-order cleanup sequence
