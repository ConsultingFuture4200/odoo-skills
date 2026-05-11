# CoA Workflow Pattern — Inbound + Outbound

Reference encoding the CoA workflow that's deployment-tested at a Part-111
dietary supplement contract manufacturer running Odoo 19 Online. Generalizes
to any FDA-regulated CPG manufacturer with similar lot-release requirements.

The pattern is **deliberately Studio-only on the Odoo side** — no custom
module — so it deploys on Odoo Online without infrastructure changes.

---

## §1 — Inbound flow (deployed)

### 1.1 Location topology

```
Vendors (Partners)
   ↓ stock.picking (Receipts picking type, code='incoming')
WH/Input                              ← initial landing zone
   ↓ stock.picking (Internal Transfer)
WH/Stock/Quarantine                   ← QA hold until CoA + checks pass
   ↓ stock.picking (Internal Transfer, manual or auto via Studio Rule)
WH/Stock/Released                     ← available for production
   ↓
WH/Pre-Production → MO → ...
```

`WH/Stock/Rejected` is the alternate destination when QC fails (manual move
during disposition per SOP-INV-008).

### 1.2 Studio fields on `stock.lot`

Six fields, all `state=manual`, all `store=True`:

| Field | Type | Notes |
|---|---|---|
| `x_coa_status` | Selection | Values: `pending` (default), `received`, `released`, `rejected`. Required. |
| `x_coa_attachment_id` | Many2one → `ir.attachment` | Required when `x_coa_status='received'` or later |
| `x_coa_received_date` | Date | Auto-set by Studio Rule when status moves to `received` |
| `x_coa_released_by` | Many2one → `res.users` | Required when `x_coa_status='released'` or `'rejected'` |
| `x_coa_release_date` | Date | Auto-set by Studio Rule when status moves to `released`/`rejected` |
| `x_coa_release_notes` | Text | Required when `x_coa_status='rejected'`; recommended on `released` |

### 1.3 Saved filters on `stock.lot` and `stock.quant`

| Filter | Model | Domain | Use |
|---|---|---|---|
| Pending CoA Release | `stock.lot` | `[('x_coa_status','in',['pending','received'])]` | QA Manager's daily AM queue |
| Released | `stock.lot` | `[('x_coa_status','=','released')]` | Active inventory by release status |
| Rejected | `stock.lot` | `[('x_coa_status','=','rejected')]` | Disposition queue for SOP-INV-008 |
| In Quarantine | `stock.quant` | `[('location_id','child_of',<quarantine_id>)]` | Receiving Operator's view of physical quarantine |

Register these as `ir.filters` with `user_id=False` so they're shared across users.

### 1.4 Operator workflow

| Step | Actor | Action | Odoo touchpoint |
|---|---|---|---|
| 1 | Receiving Operator | Validate the incoming receipt picking | `stock.picking` (Receipts) — lot lands in WH/Input |
| 2 | Receiving Operator | Internal Transfer to `WH/Stock/Quarantine`; lot record auto-created with `x_coa_status='pending'` | `stock.picking` (Internal) |
| 3 | Receiving Operator | Attach supplier CoA PDF via lot's chatter; link via `x_coa_attachment_id`; set `x_coa_status='received'`, `x_coa_received_date=today` | `stock.lot` |
| 4 | QA Inspector | Run Quality control points (CoA Received / Visual Inspection / Identity Verification per §3 of parent SKILL.md) | `quality.check` |
| 5 | QA Manager | Review CoA + QC results; decide release vs reject; fill `x_coa_released_by`, `x_coa_release_date`, `x_coa_release_notes`; set `x_coa_status='released'` or `'rejected'` | `stock.lot` |
| 6 | QA Manager | (or Studio Approval Rule in Phase 2) — Internal Transfer Quarantine → Released (or Rejected) | `stock.picking` (Internal) |

SOPs that wrap this: SOP-INV-001 (Receiving Raw Materials) + SOP-INV-002
(Releasing Quarantined Materials). Template at
`odoo-training-and-adoption/references/sop-template.md`.

### 1.5 Phase 2 automation (deferred)

| Need | Studio mechanism |
|---|---|
| Auto-move quants Quarantine → Released on `x_coa_status='released'` | Studio Automated Action firing on `stock.lot.x_coa_status` change |
| Auto-set `x_coa_received_date`, `x_coa_release_date` | Studio Computed field OR Automated Action |
| Required-fields enforcement (rejected → `x_coa_release_notes` not empty) | Studio Constraint on `stock.lot` |

These are UX improvements, not compliance requirements. Phase 1 is
auditable as-is via chatter + Studio fields.

---

## §2 — Outbound flow (design locked)

When an MO closes and produces a finished-good lot, the entity issues its
own CoA for the customer.

### 2.1 Data flow

```
MO close (mrp.production.button_mark_done)
   ↓
Studio Approval Rule on state transition to 'done'
   ↓
Triggers report generation (Studio Spreadsheet OR QWeb report)
   ↓
PDF rendered, stored as ir.attachment
   ↓
Linked via x_coa_attachment_id on the finished-good stock.lot
   ↓
Outbound delivery quality.point requires x_coa_attachment_id != False
```

### 2.2 Studio Spreadsheet vs QWeb report

| Option | Pros | Cons |
|---|---|---|
| Studio Spreadsheet | Deployable on Odoo Online (no .sh required) | Less flexible layout; learning curve for spreadsheet-as-report |
| QWeb XML report | Full layout control; standard Odoo pattern | Requires custom module → Odoo.sh or self-hosted |

For Odoo Online deployments: **Studio Spreadsheet** is the only viable
path. If you're on Odoo.sh or self-hosted, QWeb is cleaner.

### 2.3 CoA content (typical fields pulled from MO)

| Section | Source |
|---|---|
| Header (customer, customer PO, product, lot, dates) | `mrp.production` + `stock.lot` |
| Specifications | `product.template` per-product spec sheet (`ir.attachment` or Studio field) |
| Test results | `quality.check` records linked to the MO + lot |
| Components used | `mrp.production.move_raw_ids` → consumed `stock.lot` records |
| Active ingredient mass | Studio field `x_active_mass_g` on `mrp.bom` (carried to lot via MO) |
| Signatures | Studio Approval Rule audit trail (Part 11 strict requires more — see SKILL.md §7) |

### 2.4 Outbound delivery gate

`quality.point` on the Delivery Orders picking type with:

- `test_type = 'instructions'`
- Instruction text: "Verify finished-good lot has a Heron-issued CoA attached"
- (Phase 2.5) Studio Computed field on `stock.move.line` checking
  `lot_id.x_coa_attachment_id != False` — auto-fails the check if missing

---

## §3 — Customer-owned material exception

Per `odoo-schema-design/references/customer-owned-inventory-patterns.md`,
customer-owned material runs the SAME inbound QC flow but with two
differences:

1. **`stock.move.line.owner_id`** is set to the customer's `res.partner`
   (this is what excludes the material from inventory valuation)
2. **If QC fails**, the disposal path is customer-directed
   (return-to-customer OR destruction-with-customer-authorization),
   NOT the standard SOP-INV-008 disposal flow

CoA binding to the customer-owned lot works identically — same Studio
fields, same release decision. The only Workflow difference is at
Step 5/6: reject branch contacts the customer instead of executing
internal disposition.

---

## §4 — Records retention

Per 21 CFR Part 111 §111.605, all batch-related records must be retained
**7 years**. Confirm:

- `ir.attachment.create_date` on every CoA is preserved
- Odoo's standard configuration does NOT delete `ir.attachment` records
- Any IT-side backup/archive policy is at least 7 years
- Document this in the SDR §6 (records retention)

For records that exist only electronically (no wet-ink paper backup),
Part 11 controls apply — see SKILL.md §7.

---

## §5 — Worked example

Example block names + IDs are sanitized; substitute with your entity's
actuals.

```
Lot: A23 (raw material — Pectin Prime, product_id=286)
  x_coa_status: pending → received → released
  x_coa_attachment_id: ir.attachment id 12345 (CoA PDF, 320KB)
  x_coa_received_date: 2026-05-08
  x_coa_released_by: QA Manager (res.users id 11)
  x_coa_release_date: 2026-05-09
  x_coa_release_notes: "All specs in range. Released for production."

Consumed by:
  MO-2026-006 (Yuzu Lemon Gummy 2.65g, finished-good lot Y26-006)
    Quality checks: 3 IPCs all PASS
    BOM used: __heron__.bom_yuzu_lemon_20580_revfix
    Finished lot Y26-006 → outbound CoA generated (Phase 2)
    Shipped to: Customer-X (delivery order DO-2026-1142)
```

If lot A23 retroactively fails testing (e.g., contamination identified
post-release), the forward trace per SKILL.md §6.1 yields:

```
A23 → MO-2026-006 → Y26-006 → DO-2026-1142 → Customer-X
```

Two records to pull for recall: customer + delivery date + quantity
shipped. Recall procedure per SOP-QC-004.

---

## Revision history

| Version | Date | Changes |
|---|---|---|
| v0.3.0 | 2026-05-11 | Initial release. Inbound flow encoded from deployed Heron Labs configuration; outbound design locked, implementation deferred to next minor. |
