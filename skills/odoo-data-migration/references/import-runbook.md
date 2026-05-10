# Import Runbook (Phase 1–7)

Reproduction guide for a real catalog migration that ran against an
Odoo Online instance: ~190 storable products migrated to lot tracking,
44 created, 147 vendor pricelist rows. Anchored to a contract manufacturer
of dietary supplements (anonymized as **ExampleCo**); generalizes to any
similar import.

## Inputs

- A Schema Decision Record (SDR) — produced by `odoo-schema-design`
- A source dataset — for example, a Google Sheet of vendor ingredient
  quotes, ~350 rows, schema width varying 8–17 columns
- A target Odoo instance with the modules in scope already installed
- An authenticated client (see `odoo-connect`)

## Outputs

| Phase | Artifact |
|---|---|
| 0 | `00_existing_audit.json` — current product/category/UoM/partner/quant snapshot |
| 1 | `01_parsed_rows.json` — canonical rows w/ vendor canonical, is_sample flag, price/kg |
| 1 | `02_normalization_map.csv` — canonicals → CREATE/UPDATE/REVIEW |
| 1 | `03_vendor_map.csv` — vendors → existing partner id or CREATE |
| 1 | `04_cost_basis.csv` — per canonical: chosen `standard_price`, basis, exclusions |
| 2 | `06_vendor_consolidation.csv` — contacts to demote |
| 2 | `07_uom_cleanup.csv` — UoM duplicates to archive |
| 3 | `08_existing_recategorize.csv` + `08_batch_updates.json` |
| 4 | `09_phase4_updates.json` |
| 5 | `10_phase5_creates.json` + `11_phase5_import.json` |
| 6 | `12_supplierinfo*.json` — vendor pricelist payloads |
| 7 | `13_phase7_dryrun.json` — GREEN/YELLOW/BLOCKED categorization |
| 7 | `14_yellow_plan.json` — YELLOW migration full plan with quants_data + lot mappings |

## Phase-by-phase

### Phase 0 — Existing-catalog audit

Before any write, snapshot what's there.

```
search_records('product.template', [], fields=['id','name','default_code',
  'categ_id','tracking','is_storable','standard_price','uom_id'], limit=10000)
search_records('product.category', [], fields=['id','name','complete_name','parent_id'])
search_records('uom.uom', [], fields=['id','name','category_id','factor','active'])
search_records('res.partner', [['supplier_rank','>',0]], fields=['id','name',
  'is_company','parent_id','supplier_rank','vat'])
search_records('stock.quant', [['location_id.usage','=','internal']],
  fields=['id','product_id','location_id','quantity','lot_id'])
search_records('mrp.production', [], fields=['id','name','state','product_id'])
search_records('stock.picking', [], fields=['id','name','state','picking_type_id'])
```

Save outputs to `00_existing_audit.json`. This is the baseline; later phases
diff against it.

### Phase 1 — Source normalization

- Parse all rows from the source dataset
- Drop blanks and `NO_HEADER`-style sentinel rows
- Canonicalize ingredient names (fuzzy match, lowercase, strip)
- Canonicalize vendor names
- Compute `price_per_kg` per row
- Flag sample rows (`is_sample=True` if qty < threshold)
- Match against existing product catalog: classify each canonical as
  `CREATE`, `UPDATE`, or `REVIEW` (low-confidence matches go to REVIEW)
- Map vendors to existing partner IDs or `CREATE`

Output: `01_parsed_rows.json`, `02_normalization_map.csv`, `03_vendor_map.csv`.

### Phase 2 — Master data

Order: categories → UoMs → vendor companies → vendor contact reparenting →
quality scaffolding.

- Create new `product.category` records under the parent set in the SDR
- Archive UoM duplicates (`active=False`); never delete (foreign keys exist)
- Create new vendor `res.partner` records (`is_company=True`, `supplier_rank=1`)
- Demote orphan contact-shaped partners: `is_company=False`, `supplier_rank=0`,
  `parent_id=<new company id>`
- Quality scaffolding (if GMP):
  - Create one or more `quality.alert.team` records (e.g. `Receiving QA`)
  - Create `quality.point` records on receipt picking types — at minimum one
    universal CoA-receipt check, one universal visual inspection, one
    identity-verification check scoped by product category

External IDs assigned at create time per SDR convention.

### Phase 3 — Re-categorize existing

If the existing catalog lives in a flat or wrong category, batch-update it.

- Build `08_existing_recategorize.csv`: `product_id, current_categ, target_categ`
- Spot-check a sample (~10) before running the full batch
- Batch via `update_records('product.template', ids_in_batch, {categ_id: ...})`
- One product may legitimately not fit — annotate exceptions in the CSV

### Phase 4 — Update existing matches (Option B: minimum-impact)

For products that already exist in Odoo but have updated source data, prefer
the **minimum-impact update**:

- Add provenance to chatter via `post_message` (vendor quote details, source
  row reference, decision rationale)
- Update `standard_price` **only when the current price is 0** (avoid
  overwriting a manually-set price with sample-only data)
- Surface UoM hygiene issues in chatter rather than silently fixing
  (`is_storable=True` priced per `Units` instead of grams is a flag)

### Phase 5 — Create new

- Build `10_phase5_creates.json` with full payload for each new product
- Use `import_records('product.template', payload)` with the External-ID
  column populated (`__import__.<entity>_<slug>`)
- Capture returned IDs for use in Phase 6 (vendor pricelists)
- Apply naming convention: `Category - Subtype - Name - Vendor`
- Schema-compatibility note: Odoo 18 dropped `uom_po_id` from
  `product.template`. Strip from payload.

### Phase 6 — Vendor pricelists

For each (product × vendor × min_qty) tuple from `01_parsed_rows.json` that
qualifies (bulk, sourced):

```
{
  partner_id: <vendor partner id>,
  product_tmpl_id: <product template id>,
  min_qty: <kg>,
  price: <USD per kg>,
  currency_id: <USD>,
  delay: 14,
  product_uom_id: <kg>,    # explicit — different from product's stocking UoM (g)
}
```

Skip retail rows (Amazon, Walmart, etc.) and unsourced quotes — they fed cost
basis but should not become supplierinfo records.

### Phase 7 — Lot-tracking flip

See `references/lot-tracking-flip-runbook.md` for the full Option A sequence.
Summary:

1. Pre-flight dry-run: classify storable products as GREEN / YELLOW / BLOCKED
2. If BLOCKED: cancel test orders (MOs, pickings) — both parent and child moves
3. GREEN: single `update_records({tracking: 'lot'})` batch
4. YELLOW: per-product, zero quants → flip → create OPENING lot → restore
   quants with lot tagged
5. Verify: zero `is_storable=True AND tracking != 'lot'` records remain

## Verification (post-import)

```
1. Counts in vs counts out
   - Source rows in: ~350
   - Canonicals identified: ~75
   - Vendors identified: ~45
   - Products created: ~45 (Phase 5)
   - Products updated: as planned per 09_phase4_updates.json
   - Supplierinfo rows: ~150 (Phase 6)
   - Lot-flips: 100% of storable products
   - OPENING lots: one per YELLOW product

2. Spot-check N=6 high-stock products: original quantity preserved exactly,
   OPENING lot tag attached.

3. Final invariants:
   - product.template where tracking != 'lot' AND is_storable = True: 0
     (excluding intentional services)
   - Every newly-created product has an External ID
   - Every supplierinfo row has currency_id and product_uom_id set explicitly

4. Cleanup queue (deferred — log, don't fix during import):
   - Orphan virtual-location quants from canceled MOs
   - Suspected-duplicate legacy products (annotate, don't merge)
   - UoM hygiene fixes (chatter flags from Phase 4)
```

## Lessons (generalizable)

| Lesson | Reusable as |
|---|---|
| Vendor extraction caught ~50% of source rows | Insist on a structured vendor column in future intake sheets |
| Test orders blocked the lot-flip | Establish "test orders are canceled before any go-live" as an SOP |
| Direct state writes don't cascade to children | Always sweep child stock.moves after parent picking cancel |
| Odoo 18 dropped `uom_po_id` | Pin import schemas to specific Odoo versions; gate upgrades |
| Some quotes are sample-only | Cost basis can be sample-only with a flag; supplierinfo cannot |
