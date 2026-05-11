---
name: odoo-gmp-compliance
description: Use this skill for configuring Odoo to meet 21 CFR Part 111 (dietary supplement GMP) and Part 117 (food GMP) requirements, or analogous regulatory frameworks. Triggers include "GMP in Odoo", "Odoo quality module", "21 CFR Part 111", "lot tracking for compliance", "CoA workflow in Odoo", "deviation tracking", "FDA audit trail", "supplement manufacturing compliance", "recall procedure", "Part 11 e-signatures", or any work touching regulated manufacturing in Odoo. Overlay skill — applies on top of schema decisions made in odoo-schema-design. Anchored to functional gummy / dietary supplement contract manufacturing but generalizes to any FDA-regulated CPG manufacturer.
---

# Odoo GMP Compliance

> **STATUS v0.3.0:** Released from a fully-deployed configuration at a contract dietary supplement manufacturer running Odoo 19 Online. Sections on lot strategy, Receiving QA, CoA workflow (inbound live, outbound design locked), recall procedure, deviation tracking, and Part 11 gap analysis are all encoded from real config. Two known stubs remain: outbound CoA report-template implementation (Studio Spreadsheet path) and automated chatter-discipline enforcement.

## Three operating principles before configuration

1. **Lot tracking is the spine.** Without lot tracking on every storable raw material and finished good, the rest of GMP compliance is theater. Verify with the query in §1 before touching anything else.
2. **The Quality module is necessary but not sufficient.** Stock Odoo gives you `quality.point`, `quality.check`, `quality.alert`. It does NOT give you the workflow discipline that turns those into a GMP-defensible system. That's SOP work — see `odoo-training-and-adoption`.
3. **"Standard before Studio before Custom" applies twice as hard for GMP.** Custom code is hard to validate; Studio is auditable; standard Odoo is the gold standard. Default to standard. Use Studio for fields. Avoid custom modules unless an audit specifically requires a feature stock Odoo can't provide.

---

## §1 — Lot strategy enforcement matrix

The first table that should drive every product's `tracking` field on a
GMP-regulated database. Defaults below are for a dietary supplement
manufacturer subject to Part 111; Part 117 (food) is similar but allows
more flexibility on packaging.

| Product class | tracking | Rationale |
|---|---|---|
| Raw materials | `lot` (mandatory) | Supplier CoA → lot binding is the audit trail. Part 111 §111.155 requires component traceability. |
| Finished goods | `lot` (mandatory) | Customer recall capability requires per-lot traceability. Part 111 §111.260. |
| WIP / intermediate | `lot` (recommended) | Stage-by-stage traceability. Sub-recipes (e.g., "Pectin Blend A") are lot-tracked because they're prepared and held before use; un-tracked intermediates have to be inferred from MO consumption logs, which is brittle. |
| Packaging materials | `none` (default off) | Lot-tracking every cap and label is expensive and rarely useful. Switch on by exception (specific customer demand, specific regulation, or label artwork that itself has version control needs). |
| Service products | `none` | Not stockable. |
| Serial | `none` for product flow | Reserved for unique-instance items (capital equipment maintenance per `maintenance.equipment`). Not for gummies / supplements at the inventory level. |

### Verification query — must return zero results (excluding intentional services)

```python
search_records('product.template',
  [['is_storable','=',True], ['tracking','!=','lot']],
  fields=['id','name'])
```

The lot-tracking flip runbook to *get* there is in
`odoo-data-migration/references/lot-tracking-flip-runbook.md`.

### Lot naming convention

Standardize across vendors. Recommended pattern:

| Source | Pattern | Example |
|---|---|---|
| Vendor lot supplied | use vendor's lot number verbatim | `LOT-A-25-CSL509` |
| Vendor lot missing | generate `<sku>-<vendor-code>-<YYYYMMDD>` | `CL-Y-FM-FRMX-20260510` |
| Customer-owned material | prefix `CUST-<customer-code>-<batch>` | `CUST-FC-2026-Q2-001` |
| Heron-issued opening balance | `OPENING-<sku>-<YYYYMMDD>` | `OPENING-P-E49092-C-20260509` |

The convention matters less than the consistency. Pick one and lock it in
the SDR.

---

## §2 — Warehouse layout for GMP traceability

A regulated warehouse needs explicit segregation states. The recommended
sub-location hierarchy under `WH/Stock`:

```
WH (id 1, single warehouse for most CMOs)
└── Stock (id 8)
    ├── Quarantine          ← receipts land here pending CoA review
    ├── Released            ← raw materials cleared by QA
    ├── WIP                 ← intermediate / sub-recipes
    ├── Rejected            ← failed QC; awaits disposition
    ├── Finished Goods (Held)     ← post-MO, pre-release
    ├── Finished Goods (Released) ← cleared for shipment
    └── Customer-Owned      ← tolling / consignment (use stock.quant.owner_id)
```

External IDs for each (`__heron__.location_wh_stock_quarantine` etc.) make
the configuration portable and let Studio Approval Rules reference them
by name rather than ID.

The Customer-Owned sub-location pattern is documented in
`odoo-schema-design/references/customer-owned-inventory-patterns.md`.

---

## §3 — Receiving QA: the 3-control-point pattern

A proven, deployment-ready pattern:

```
quality.alert.team:
  Receiving QA (one team for all incoming receipts)
    - Members: QA Inspector + QA Manager (escalation)
    - Scope: all incoming receipts

quality.point (3 records, all on Receipts picking types):

  1. CoA Received (universal)
     - picking_type_id: every Receipts type
     - product domain: any
     - test_type: textual confirmation that supplier CoA was received
     - failure: routes to Receiving QA team for review

  2. Visual Inspection (universal)
     - picking_type_id: every Receipts type
     - product domain: any
     - test_type: pass/fail
     - failure: routes to Receiving QA, blocks transfer to Released

  3. Identity Verification (scoped)
     - picking_type_id: every Receipts type
     - product domain: only high-risk categories (e.g. botanical
       extracts, actives where adulteration risk is documented)
     - test_type: matches CoA assay → operator confirmation
     - failure: routes to Receiving QA, blocks transfer to Released
```

The **scoping** of the third point is the load-bearing pattern. Naively
running an Identity Verification check on every receipt (including, say,
Tapioca Sugar) creates alert fatigue and trains operators to click-through.
Scoping to high-risk categories keeps the check meaningful.

### The "scoped quality.point" trick

`quality.point` records have a domain field that filters which products they
fire on:

| Want | Set `product_domain` to |
|---|---|
| Fire on every product | `[]` (empty domain) |
| Fire only on a specific category | `[('categ_id','=',<categ_id>)]` |
| Fire only on a category subtree | `[('categ_id','child_of',<root_categ_id>)]` |
| Fire on multiple categories | `[('categ_id','in',[<id1>,<id2>,...])]` |
| Fire only on a specific product | `[('id','=',<product_id>)]` |
| Skip a category | `[('categ_id','not in',[<id>])]` |

`child_of` against high-risk category roots keeps the configuration simple
as new products are added.

---

## §4 — CoA workflow

The CoA (Certificate of Analysis) workflow has two flows: **inbound**
(supplier CoA bound to received lot) and **outbound** (entity-issued CoA
generated per finished-good lot before shipment).

### 4.1 Inbound flow (deployment-ready)

```
Receipt        QA hold              Released raw materials
  WH/Input  →  WH/Stock/Quarantine  →  WH/Stock/Released
  (supplier)        ↓                       ↑
                 [QA review]             [release decision]
                    ↓                       ↑
                  CoA bound to lot ────────┘ (SOP-INV-002)
                    ↓
                  Rejected → WH/Stock/Rejected
```

**Studio fields on `stock.lot`** — six fields cover the full inbound flow:

| Field | Type | Purpose |
|---|---|---|
| `x_coa_status` | Selection: pending / received / released / rejected | Lifecycle state |
| `x_coa_attachment_id` | Many2one → `ir.attachment` | Primary CoA file linked to this lot |
| `x_coa_received_date` | Date | When the supplier CoA was attached |
| `x_coa_released_by` | Many2one → `res.users` | QA Manager who approved release |
| `x_coa_release_date` | Date | When the lot was released to production |
| `x_coa_release_notes` | Text | Release / reject rationale (required for audit) |

**Saved filters (`ir.filters`)** for QA queue management:

| Filter | Model | Domain |
|---|---|---|
| Pending CoA Release | `stock.lot` | `x_coa_status in ('pending','received')` |
| Released | `stock.lot` | `x_coa_status = 'released'` |
| Rejected | `stock.lot` | `x_coa_status = 'rejected'` |
| In Quarantine | `stock.quant` | `location_id child_of <quarantine_id>` |

**Operator workflow:** SOP-INV-001 (Receiving) + SOP-INV-002 (Releasing
Quarantined Materials). See `odoo-training-and-adoption/references/sop-template.md`
for the SOP shape. The QA Manager opens the lot, attaches the supplier
CoA via chatter, links it via `x_coa_attachment_id`, sets `x_coa_status`,
and (Phase 2) the Studio Approval Rule auto-moves quants Quarantine →
Released on release.

### 4.2 Outbound flow (design locked, implementation Phase 2)

When an MO closes and produces a finished-good lot:

1. **Heron-issued CoA** generated from a QWeb report template OR a
   Studio Spreadsheet template — pulls active mass, lot number,
   manufacture date, expiration, BOM components used, QC results from
   the MO's `quality.check` records.
2. PDF is attached to the lot's chatter and linked via
   `x_coa_attachment_id` (same Studio field as inbound).
3. **Outbound delivery picking** has a `quality.point` requiring
   `x_coa_attachment_id != False` on each lot before validation —
   ensures no FG ships without a CoA.

**Studio vs custom-module decision per phase:**

| Need | Solution | Notes |
|---|---|---|
| Status field on lot | Studio Selection field | ✓ Phase 1 |
| Attachment link on lot | Studio Many2one to `ir.attachment` | ✓ Phase 1 |
| Filter views | Saved filters | ✓ Phase 1 |
| Quality control points on receipts | Standard Quality module | ✓ already in production |
| Auto-move quants on release | Studio Approval Rule firing on `x_coa_status` change | Phase 2 UX improvement; manual UI in Phase 1 |
| Outbound CoA PDF generation | Studio Spreadsheet template (preferred on Odoo Online) OR QWeb XML (requires Odoo.sh) | Phase 2 |
| Outbound shipment gate | Quality control point on Delivery picking with `x_coa_attachment_id` requirement | Phase 2 |

**No custom module required for either phase on Odoo Online.**

### 4.3 Customer-owned material exception

Per `odoo-schema-design/references/customer-owned-inventory-patterns.md`,
customer-owned material runs the SAME inbound QC flow but with two
differences:

1. The `stock.move.line.owner_id` is set to the customer's `res.partner`
   (this is what excludes the material from inventory valuation)
2. If QC fails, the disposal path is customer-directed (return-to-customer
   or destruction-with-customer-authorization), NOT the standard
   SOP-INV-008 disposal flow

CoA can be either supplier-issued (vendor sent material to customer who
sent it to Heron) OR customer-issued (customer ran QC themselves). Either
way, the binding to `stock.lot` is the same.

---

## §5 — Deviation tracking

Stock Odoo's `quality.alert` model is the deviation record. It's already
created automatically when a `quality.check` fails — no Studio scaffolding
needed.

**What's missing from stock Odoo is the PROCESS:**

| Step | Owner | Standard Odoo support |
|---|---|---|
| Detect deviation | Any operator | `quality.check.test_type` + fail action |
| Author the alert | Reporting operator | `quality.alert.create_uid` |
| Categorize (critical / major / minor) | QA Manager | Studio Selection field on `quality.alert` |
| Assign investigation owner | QA Manager | Standard `responsible_user_id` |
| Document root cause | Investigation owner | Chatter on the alert |
| Decide disposition | QA Manager | Studio Selection: rework / use-as-is / scrap / return |
| Define CAPA (corrective + preventive action) | QA Manager | Chatter |
| Verify CAPA effectiveness | QA Manager | Chatter + a follow-up date field |
| Close | QA Manager | `quality.alert.stage_id` → Solved |

**Recommended Studio fields on `quality.alert`** (add these for any
regulated entity):

| Field | Type | Purpose |
|---|---|---|
| `x_severity` | Selection: critical / major / minor | Drives escalation |
| `x_disposition` | Selection: rework / use_as_is / scrap / return_to_vendor / destroy | Final fate decision |
| `x_capa_due_date` | Date | When the corrective action must be implemented |
| `x_capa_verified_date` | Date | When effectiveness was confirmed |

SOP wraps this: SOP-QC-003 Recording a Deviation. The escalation rule of
thumb: critical (release stops on any related lot) → notify QA Manager +
Sponsor immediately; major → next business day; minor → weekly review.

---

## §6 — Recall procedure (forward + back trace)

Stock Odoo's standard report combo for traceability:

| Report | Purpose | Where |
|---|---|---|
| Lot Traceability Report | Forward + back trace from any lot | Inventory ▸ Reporting ▸ Traceability Report |
| Stock Move History | Per-move audit (in/out, partner, picking) | On any `stock.lot` record → smart button |
| Manufacturing Order chatter | Components consumed + lot output | On any `mrp.production` record |

### 6.1 Forward trace (lot → affected customers)

> "Lot ABC of raw material X failed retroactive testing. Which customer
> shipments need to be recalled?"

```
1. Open the affected raw-material stock.lot (lot ABC of product X)
2. Smart button → Stock Moves
3. Filter: outgoing moves where lot consumed in an MO
4. For each consuming MO: identify the produced finished-good lot(s)
5. For each finished-good lot: smart button → Stock Moves
6. Filter: outgoing moves to Partners/Customers
7. For each delivery: partner = customer to notify
```

### 6.2 Back trace (finished-good lot → vendors of components)

> "Customer reports adverse event on finished-good lot DEF. Which vendor
> components contributed?"

```
1. Open the finished-good stock.lot (lot DEF)
2. Find the MO that produced it (search mrp.production where
   move_finished_ids.lot_id = DEF)
3. On the MO: smart button → Stock Moves (consumed)
4. For each consumed lot: identify the source receipt + vendor (res.partner)
5. For each vendor lot: pull the supplier CoA from x_coa_attachment_id
```

### 6.3 SOP-QC-004 (Initiating a Recall)

The report combo above is the *technical* trace. The SOP wraps it with:

- Who decides recall vs notify-only
- Customer notification template
- FDA reportable-event threshold (Part 7 §7.40 / Part 111 §111.610)
- Hold-and-segregate procedure for unshipped affected lots
- Mock-recall cadence (annual) to verify the procedure

A full SOP draft is in `odoo-training-and-adoption/references/sop-template.md`.

---

## §7 — Part 11 (e-signatures) gap analysis

21 CFR Part 11 applies when electronic records / signatures are used in
place of paper records that the FDA requires. For Part-111-regulated
dietary supplement manufacturers, Part 11 applies primarily to records
that exist solely electronically AND are used to satisfy Part 111 record
requirements.

### What stock Odoo + Studio provides

| Part 11 control | Stock Odoo coverage |
|---|---|
| §11.10(a) Validation | Partial — Odoo's standard QA program covers core; entity must validate their specific configuration |
| §11.10(b) Accurate record copies | ✓ Full — `ir.attachment` retains source PDFs; chatter retains text |
| §11.10(c) Record retention | ✓ Full — `ir.attachment` doesn't auto-delete; chatter preserved indefinitely |
| §11.10(d) Limit access to authorized users | ✓ Full — `res.users` + `res.groups` + record rules |
| §11.10(e) Audit trail | ✓ Full — chatter + Odoo's standard tracking on tracked fields |
| §11.10(f) Sequence enforcement | Partial — Studio Approval Rules enforce sequence; can be bypassed by admin |
| §11.10(g) Authority checks | ✓ Full — record rules + groups |
| §11.10(h) Device checks | ✗ Not provided |
| §11.10(i) Personnel training | Partial — out of scope for Odoo; covered by HR / training records |
| §11.10(k) Document control | Partial — Documents module + ir.attachment work; rev control needs SOP |
| §11.30 Open systems | N/A for on-prem; partial for Odoo.sh / Online (TLS only) |
| §11.50 Signature manifestation | ✗ Not provided — chatter signs the user but doesn't bind a cryptographic signature to the record hash |
| §11.70 Signature/record linking | ✗ Not provided — would need custom module |

### What's NOT provided by stock Odoo + Studio

| Gap | Implication | Closure path |
|---|---|---|
| Cryptographic signature binding | Auditor may not accept Odoo signatures as Part 11 compliant for product release | Custom module OR retain wet-ink signed copies alongside Odoo records |
| Reason-for-change as structured field | Workaround: free-text in chatter or in a Studio field like `x_change_reason` | Studio (acceptable for Part 111; weak for Part 11 strict) |
| Periodic re-authentication for high-risk ops | Operator stays logged in; release decisions don't re-prompt | Custom module OR procedural via SOP |

### Practical guidance

For a Part-111-only manufacturer (no Part 11 requirements from customers):

> Use Studio + chatter discipline as the audit trail. Retain wet-ink
> signatures on any record an FDA inspector might specifically ask about
> (final release, deviation closure, recall decision). Document the
> hybrid approach in the SDR §6 with a note that Part 11 controls are
> not implemented and are not required for Part 111 compliance.

For a manufacturer with customer audit requirements for Part 11 strict
compliance:

> Plan a custom module deployment (requires Odoo.sh or self-hosted).
> Module scope: cryptographic record-hash + signature binding,
> structured reason-for-change, periodic re-auth for high-risk ops.
> Off-the-shelf options exist (search OCA `electronic_signature` modules)
> but verify against your specific Part 11 §11.10 / §11.30 / §11.50
> requirements before deploying.

---

## §8 — 21 CFR Part 111 mapping table

Each subpart of Part 111 maps to Odoo configuration:

| Part 111 subpart | Odoo configuration |
|---|---|
| §111.8 Plant + grounds | Out of scope for Odoo (facility records) |
| §111.10 Equipment + utensils | `maintenance.equipment` + `maintenance.request` |
| §111.12 Personnel | `hr.employee` + training records via Documents module |
| §111.14 Holding + distribution | `stock.location` segregation (Quarantine / Released / Held / FG) |
| §111.20 Production + process controls | `mrp.bom` + `mrp.production` + `quality.point` |
| §111.21 Quality unit | `quality.alert.team` + role separation via `res.groups` |
| §111.27 Quality control operations | `quality.point` on receipts/MOs/deliveries |
| §111.35 Sanitation | Out of scope for Odoo (record in chatter on `maintenance.request`) |
| §111.40 Equipment used | `maintenance.equipment` per machine |
| §111.65 Components from suppliers | `res.partner.x_approved_vendor` Studio flag + CoA workflow §4 |
| §111.70 Component specifications | `product.template` + per-product spec sheet in `ir.attachment` |
| §111.95 Production records | `mrp.production` chatter + linked `quality.check` records |
| §111.110 Master Batch Record | `mrp.bom` (active version) + linked product specifications |
| §111.155 Components must meet specs before use | CoA workflow §4 (lot can't move Quarantine → Released without CoA) |
| §111.160 In-process testing | `quality.point` on Manufacturing picking type |
| §111.165 Batch production record | `mrp.production` (the MO itself, with chatter + quality.check trail) |
| §111.180 Records of components received | `stock.lot` + supplier CoA attachment |
| §111.210 Master Manufacturing Record | Same as §111.110 — the active `mrp.bom` |
| §111.260 Distribution records | `stock.picking` (outgoing) + customer `account.move` |
| §111.350 Returned products | `stock.picking` reverse + `stock.location` "Returns" sub-loc |
| §111.420 Customer complaints | `helpdesk.ticket` (if Helpdesk module enabled) OR `crm.lead` with custom category |
| §111.605 Records retention (7 years) | `ir.attachment` doesn't auto-delete; verify no IT-side retention policy is shorter |
| §111.610 Recall procedure | SOP-QC-004 + traceability report combo §6 |

---

## §9 — When to invoke this skill alongside others

| Co-invoke with | Why |
|---|---|
| `odoo-schema-design` | Before this — lot strategy + warehouse layout decisions live there; this skill ENFORCES them, doesn't make them |
| `odoo-data-migration` | When flipping `tracking='none' → 'lot'` on an active database — there's a runbook for that specific operation |
| `odoo-training-and-adoption` | For the SOP layer that wraps every Quality module workflow. Without the SOPs, the Odoo config is theater. |
| `odoo-webhooks` | If you need to surface deviations to an external system (e.g., Slack alert on critical deviation) |

---

## What this skill does not cover

- Pharmaceutical (Part 211) — different regulation, different validation, out of scope for v0.x
- Medical device (Part 820) — different regulation, out of scope
- HACCP / FSMA Preventive Controls (Part 117 Subpart C) — adjacent but separate; Part 117 §117 mapping table is a future minor release
- Cannabis state-specific compliance (Metrc, BioTrack) — see the dedicated `metrc` agent
- Software validation (CSV / GAMP 5) for Odoo itself — out of scope; assume Odoo is "GAMP 4: configurable off-the-shelf" and validate per-entity configuration not the platform

## References

- `references/coa-workflow-pattern.md` — full inbound + outbound CoA flow with example Studio scaffolding
- `references/part-111-mapping-table.md` — 21 CFR Part 111 subpart → Odoo configuration (full table)
- `odoo-data-migration/references/lot-tracking-flip-runbook.md` — getting from `tracking='none'` to `tracking='lot'` on a live database
- `odoo-schema-design/references/customer-owned-inventory-patterns.md` — customer-owned material via `stock.quant.owner_id`
- 21 CFR Part 111 (Dietary Supplement CGMP): <https://www.ecfr.gov/current/title-21/chapter-I/subchapter-B/part-111>
- 21 CFR Part 117 (Food CGMP): <https://www.ecfr.gov/current/title-21/chapter-I/subchapter-B/part-117>
- 21 CFR Part 11 (Electronic Records / Signatures): <https://www.ecfr.gov/current/title-21/chapter-I/subchapter-A/part-11>

## TODO before next minor (v0.4)

- Outbound CoA report-template implementation (Studio Spreadsheet path) — needs a real entity's CoA spec sheet authored
- Part 117 §117 mapping table (food CGMP, adjacent to but separate from Part 111)
- Validation master plan template (one-pager: scope, assumptions, IQ/OQ/PQ approach for a configurable Odoo deployment)
- Worked example: tracing a contamination event end-to-end through the Quality module
