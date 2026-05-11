---
name: odoo-data-migration
description: Use this skill for bulk-loading or migrating data into Odoo via CSV/Excel imports, JSON payloads, or RPC batch writes. Triggers include "import into Odoo", "Odoo CSV import", "load our products into Odoo", "migrate from QuickBooks/NetSuite/Shopify to Odoo", "bulk update Odoo", "Odoo data migration", or any first-load operation. Handles dedup, validation, dry-run, rollback, lot-tracking flips, and external-ID-based idempotency. Should be invoked AFTER odoo-schema-design has produced an SDR. Do NOT use for ongoing event-driven sync (use odoo-api-sync) or one-off small writes (those go through odoo-connect directly).
---

# Odoo Data Migration

> **STATUS v0.3.0:** Released against Odoo 19 Online deployment. Patterns grounded in: a 200-product catalog migration with lot-tracking flip, 14-vendor consolidation, 3-warehouse collapse with quant migration, Studio-model scaffolding via raw API, and 7-phase idempotent re-runnability. All patterns are deployment-tested.

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

## Multi-warehouse collapse pattern

The mirror image of opening balance loading: when an SDR analysis reveals that
multiple `stock.warehouse` records were over-modeled (configured in
anticipation of ops that never materialized), the cleanup is a structured
4-step sequence. Pattern proven on a 4-warehouse → 1-warehouse collapse with
6 quants migrated and zero data loss.

### Pre-flight (empirical state check)

```python
# For each warehouse, count active quants + lifetime pickings
for wh_id in warehouse_ids:
    quants = search_records('stock.quant',
        [['location_id.warehouse_id', '=', wh_id],
         ['location_id.usage', '=', 'internal'],
         ['quantity', '!=', 0]])
    pickings = search_records('stock.picking',
        [['picking_type_id.warehouse_id', '=', wh_id]],
        fields=['id'])
    print(wh_id, len(quants), len(pickings))
```

Sister warehouses with zero pickings ever AND ≤ a handful of quants are the
collapse candidates.

### Execution sequence

```
1. Migrate quants from sister WH internal locations → primary WH equivalents
   - For each non-zero quant: stock.quant.write(location_id=<primary_loc_id>)
   - For zero-quant rows: archive (active=False) or unlink

2. Archive sister warehouses' picking types (stock.picking.type.active=False)
   - Skipping this step leaves stale picking-type dropdowns in the UI

3. Re-parent any meaningful child locations from sister WH to primary WH
   - e.g., LDF's drying speed racks become WH/Drying Area/Speed Rack 1..7
   - Update stock.location.location_id to the new primary parent
   - Update warehouse_id to match

4. Archive sister stock.warehouse records (active=False)
   - DO NOT unlink — historical picking references would orphan
```

### Verification invariants

- All quants now under primary WH (no orphaned `location_id.warehouse_id != primary`)
- Sister warehouses' `active=False`, but their `id` is preserved for historical lookups
- Re-parented locations: `location_id.warehouse_id == new_parent.warehouse_id`

This pattern works on Odoo 18 and 19. On 19, `stock.warehouse.archive()` is
the standard method; on 18, `write({'active': False})` is equivalent.

---

## Studio scaffolding via raw API

When the SDR calls for custom Studio fields or models AND the deployment is
Odoo Online (forbidding custom modules), you can scaffold Studio assets
programmatically via the `ir.model` / `ir.model.fields` / `ir.model.access`
models. Faster than clicking through Studio UI for large field sets, and
the operations are version-controllable.

### Creating a Studio model

```python
# 1. Create the model
model = create_record('ir.model', {
    'name': 'Heron Drive Ingest State',
    'model': 'x_heron_drive_ingest_state',  # MUST start with 'x_'
    'state': 'manual',                       # Studio-equivalent
})
# Odoo auto-creates default fields: id, create_date, create_uid,
# write_date, write_uid, display_name, x_name

# 2. Create custom fields (parallel-able)
create_records('ir.model.fields', [
    {'name': 'x_drive_file_id', 'model_id': model['id'],
     'ttype': 'char', 'field_description': 'Drive File ID',
     'state': 'manual', 'store': True, 'required': True},
    # ... more fields
])

# 3. Create ACLs (required — model is inaccessible without them)
create_record('ir.model.access', {
    'name': 'x_heron_drive_ingest_state inventory admin',
    'model_id': model['id'],
    'group_id': <admin_group_id>,
    'perm_read': True, 'perm_write': True,
    'perm_create': True, 'perm_unlink': True,
})
# Recommend a parallel internal-user read ACL (group_id=1) so the model is
# queryable from any logged-in session.
```

### Adding fields to an existing model

Same pattern, but `model_id` points to an existing `ir.model` record:

```python
create_records('ir.model.fields', [
    {'name': 'x_coa_status', 'model_id': <stock_lot_model_id>,
     'ttype': 'selection',
     'selection': "[('pending','Pending'),('received','Received'),"
                  "('released','Released'),('rejected','Rejected')]",
     'field_description': 'CoA Status', 'state': 'manual', 'store': True},
    {'name': 'x_coa_attachment_id', 'model_id': <stock_lot_model_id>,
     'ttype': 'many2one', 'relation': 'ir.attachment',
     'field_description': 'CoA Attachment', 'state': 'manual', 'store': True},
])
```

### Caveats

| Caveat | Workaround |
|---|---|
| `state='manual'` is required for Studio-style fields | Without it, the field is treated as a custom-module declaration and won't render in Studio UI |
| Model + field names MUST start with `x_` | Odoo enforces this; non-prefixed names get rejected silently in some versions |
| `ir.model.access` cache lag | After raw inserts, ACL grants don't take effect until the registry reloads (next page load OR server-side process restart). MCP sessions that just created the rules can still get 403. Expect this; warn operators. |
| `selection` field on `ir.model.fields` is a string-encoded list of tuples | NOT a Python list. Format: `"[('a','A'),('b','B')]"` |
| Studio UI may "discover" your raw-API records after a page reload | They become editable in Studio post-hoc. Good — single source of truth. |
| Model unlink is dangerous | Once data is in your custom model, archiving is preferred over unlinking |

---

## Common gotchas (Odoo 18-19, current as of mid-2026)

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
| Odoo 19 `create()` returns a list for single-dict input | `int()` cast on result raises `TypeError: list` | Normalize: `int(result[0] if isinstance(result, list) else result)`. Same for `message_post()`. |
| Odoo 19 `res.groups.users` field renamed | `Invalid field 'users' in 'res.groups'` | Use `user_ids` (m2m) on 19; `users` still works on 18. |
| Odoo 19 `ir.filters.user_id` renamed | `Invalid field 'user_id' in 'ir.filters'` | Use `user_ids` (m2m) on 19; or omit (defaults to "all users"). |
| ACL cache doesn't refresh after raw `ir.model.access` create | 403 on read even with correct grants | Wait for next process reload OR page refresh. Don't fight it during migration; smoke-test in a later session. |
| `stock.location` doesn't have a `comment` field | `Invalid field 'comment' in 'stock.location'` | Use chatter (`message_post`) for location-level notes, not a field. |

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

## Revision history

| Version | Date | Changes |
|---|---|---|
| v0.3.0 | 2026-05-11 | Add Multi-warehouse collapse pattern. Add Studio scaffolding via raw API (ir.model / ir.model.fields / ir.model.access). Add Odoo 19 quirks: list-returns on create/message_post, res.groups.user_ids rename, ir.filters.user_ids rename, ACL cache lag. Update header range to Odoo 18-19, mid-2026. |
| v0.2.1 | 2025-late | Lot-tracking flip Option A runbook, vendor consolidation pattern, 7-phase plan template. |
