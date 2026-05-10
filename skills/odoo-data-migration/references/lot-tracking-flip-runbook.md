# Lot-Tracking Flip Runbook — Option A

The proven sequence for flipping `product.template.tracking` from `none` to
`lot` on a live Odoo database while preserving on-hand stock counts exactly.
Executed against ~190 storable products in a real migration (~95 GREEN
flipped in batch, ~100 YELLOW migrated individually, several legacy/orphan
surfaced and migrated alongside).

## Why Option A

Odoo's constraint: `tracking` cannot transition from `none` to `lot` while
any internal-location quants exist for that product. The product must reach
zero on-hand at every internal location before the flip is allowed.

| Option | Approach | Use when |
|---|---|---|
| **A** | Snapshot quants → zero → flip → restore w/ OPENING lot | Production cutover; on-hand counts must be preserved exactly; lot history will start fresh |
| B | Scrap-and-receive | On-hand counts can be reset; you'd rather a clean break |
| C | Skip — keep `tracking='none'` | Never for GMP-regulated entities |

Option A is the default for any go-live. Option B is acceptable for low-stock
products where the count is approximate anyway.

## Pre-flight (dry-run)

```
1. Get the full storable-product list:
   search_records('product.template', [['is_storable','=',True]],
                  fields=['id','name','tracking','default_code'])

2. For each product, query internal-location quants:
   search_records('stock.quant',
     [['product_id.product_tmpl_id','=',<id>],
      ['location_id.usage','=','internal']],
     fields=['id','location_id','quantity','package_id'])

3. Classify:
   - tracking already 'lot' → skip
   - no internal quants → GREEN (flip in batch)
   - has internal quants → YELLOW (full migration)
   - referenced by uncanceled mrp.production or stock.picking → BLOCKED

4. Save: 13_phase7_dryrun.json with three lists.
```

The `location_id.usage='internal'` filter is critical. Quants in
`production`, `inventory`, `scrap`, or `transit` virtual locations do NOT
count toward the "must be zero" constraint and should not be touched.

## Test-order cleanup (if BLOCKED)

If BLOCKED products are held by test MOs and SO pickings (a common state
on a pre-go-live database), cancel them first.

```
1. For each draft/confirmed mrp.production blocking a product:
   update_record('mrp.production', <id>, {state: 'cancel'})

2. For each draft/confirmed/assigned stock.picking blocking a product:
   update_record('stock.picking', <id>, {state: 'cancel'})

3. CASCADE QUIRK: parent cancel via direct state write does NOT cascade.
   Sweep children:
   search_records('stock.move',
     [['picking_id','in',<canceled_picking_ids>],
      ['state','not in',['done','cancel']]])
   update_records('stock.move', <child_ids>, {state: 'cancel'})

4. Re-run pre-flight. BLOCKED list should be empty.
```

If non-test orders are blocking, escalate to the SPoC. Do NOT cancel real
production work to enable a migration.

## GREEN flip

```
update_records('product.template', <green_ids>, {tracking: 'lot'})
```

One batch, ~5 seconds for ~100 records. No further action.

## YELLOW migration

For each YELLOW product, in this exact order:

```
Step 1 — Snapshot:
  Capture quants_data = [
    {location_id, quantity, package_id}
    for each existing internal-location quant for this product
  ]

Step 2 — Zero:
  For each quant in quants_data:
    update_record('stock.quant', <quant_id>, {quantity: 0})

Step 3 — Flip:
  update_record('product.template', <product_tmpl_id>, {tracking: 'lot'})

Step 4 — Create OPENING lot:
  create_record('stock.lot', {
    product_id: <product_id>,           # variant id, not template id
    name: f'OPENING-{default_code}-{YYYYMMDD}',
    company_id: 1,
  })
  Capture returned lot_id.

Step 5 — Restore quants:
  For each entry in quants_data:
    create_record('stock.quant', {
      product_id: <product_id>,
      location_id: <entry.location_id>,
      quantity: <entry.quantity>,
      lot_id: <new_lot_id>,
      package_id: <entry.package_id>,
    })
```

Order is non-negotiable. Step 3 (flip) will fail if any quant has non-zero
quantity at an internal location.

## Verification

```
1. For each YELLOW product, confirm:
   - tracking = 'lot' on the template
   - exactly one stock.lot exists named OPENING-<sku>-<date>
   - sum of new quants = sum of original quants (per location)

2. Spot-check N high-stock products in the UI:
   - Inventory → Reporting → Stock at Date → confirm quantity matches
   - Inventory → Lots → confirm OPENING lot is present

3. Final invariant query:
   search_records('product.template',
     [['is_storable','=',True], ['tracking','!=','lot']],
     fields=['id','name'])
   → Must return zero results (excluding intentional services).
```

## Edge cases observed in practice

| Symptom | Cause | Fix |
|---|---|---|
| Products surfaced via stock.quant query that weren't in the product list | Legacy/orphan products with quants but no recent activity | Add to YELLOW list — migration is identical |
| Orphan quants at `Virtual Locations/Production` after migration | Canceled MO left virtual-location quants | Cosmetic; clears on first real production run, or zero in UI |
| Some products with quants but `is_storable=False` | Misconfigured product (service-typed but had stock) | Out of scope for the lot-flip; flag for catalog cleanup |

## Rollback

If any step fails partway:

1. **Per-product:** revert that product's tracking via
   `update_record('product.template', <id>, {tracking: 'none'})`. Quants
   created in Step 5 can be deleted; original quants are gone (Step 2 set
   them to 0).

2. **Whole-batch:** restore from the database snapshot taken before Phase 7
   started (Odoo Online: Settings → Database → Restore).

The snapshot discipline is what makes per-product rollback unnecessary in
practice. Always snapshot before starting Phase 7.
