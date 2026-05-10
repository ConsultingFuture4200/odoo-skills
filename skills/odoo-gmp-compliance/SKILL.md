---
name: odoo-gmp-compliance
description: Use this skill for configuring Odoo to meet 21 CFR Part 111 (dietary supplement GMP) and Part 117 (food GMP) requirements, or analogous regulatory frameworks. Triggers include "GMP in Odoo", "Odoo quality module", "21 CFR Part 111", "lot tracking for compliance", "CoA workflow in Odoo", "deviation tracking", "FDA audit trail", "supplement manufacturing compliance", or any work touching regulated manufacturing in Odoo. Overlay skill — applies on top of schema decisions made in odoo-schema-design. Anchored to functional gummy / dietary supplement contract manufacturing but generalizes to any FDA-regulated CPG manufacturer.
---

# Odoo GMP Compliance

> **STATUS:** Partial. The lot-strategy enforcement matrix and the 3-control-point Receiving QA pattern are documented from a live config. The CoA workflow, recall procedure, deviation tracking, and Part 11 e-signature gap remain stubs and will be fleshed out in a later release once a complete CoA workflow has been built end-to-end against a real entity.

## What's solid in this release

- Lot strategy enforcement matrix (executed across ~190 storable products)
- Receiving QA scaffolding pattern (1 team + 3 control points)
- Quality `point` scoping by product category (universal vs high-risk only)
- The "scoped quality.point" trick for avoiding alert fatigue

## What's still a stub

- CoA workflow end-to-end (supplier inbound + entity outbound)
- Recall procedure (forward + back trace) with exact Odoo report combo
- Deviation tracking via `quality.alert` workflow
- Part 11 (e-signature) gap analysis
- 21 CFR Part 111 §111 mapping table

## Lot strategy enforcement matrix

This is the table that should drive every product's `tracking` field on a
GMP-regulated database. Defaults below are for a dietary supplement
manufacturer subject to Part 111.

| Product class | tracking | Rationale |
|---|---|---|
| Raw materials | `lot` (mandatory) | Supplier CoA → lot binding is the audit trail. Required. |
| Finished goods | `lot` (mandatory) | Customer recall capability requires per-lot traceability. Required. |
| WIP / intermediate | `lot` (recommended) | Stage-by-stage traceability. Sub-recipes (e.g., "Pectin Blend A") are lot-tracked because they're prepared and held before use. |
| Packaging materials | `none` (default off) | Lot-tracking every cap is expensive and rarely useful. Switch on by exception (specific customer, specific regulation). |
| Service products | `none` | Not stockable. |
| Serial | `none` | Reserved for unique-instance items (capital equipment). Not for gummies. |

Verification query — must return zero results (excluding intentional services):

```
search_records('product.template',
  [['is_storable','=',True], ['tracking','!=','lot']],
  fields=['id','name'])
```

The lot-tracking flip runbook to *get* there is in
`odoo-data-migration/references/lot-tracking-flip-runbook.md`.

## Receiving QA — the 3-control-point pattern

A proven, deployment-ready pattern:

```
quality.alert.team:
  Receiving QA (id 2)
    - Members: TBD (assigned to operators per shift)
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

## The "scoped quality.point" trick

`quality.point` records have a domain field that filters which products they
fire on. The pattern:

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

## What's missing (deferred)

### CoA workflow

Inbound: supplier CoA attached to receipt → bound to `stock.lot` →
release decision moves quant from Quarantine to Released.

Outbound: entity-issued CoA generated per finished-good lot at MO close →
attached to `stock.lot` → required attachment on outbound delivery.

Both flows need:
- A real Quarantine location (often missing in greenfield setups)
- Studio vs custom-module decision per OIM "Standard before Studio before
  Custom"
- SOP authoring (SOP-INV-002 Releasing Quarantined Materials, SOP-QC-002
  Issuing a Certificate of Analysis)

### Recall procedure

Forward trace: from raw material lot → all finished good lots that consumed
it → all customer shipments containing those finished goods.

Back trace: from a finished good lot → all raw material lots that went into
it → all suppliers of those raw materials.

Odoo's standard reports cover most of this; the gap is the documented
SOP that links the report combo to the recall decision.

### Deviation tracking

The `quality.alert` workflow is built into the Quality module. The gap is
process: who authors a deviation, who reviews, who closes, what triggers
escalation. This is SOP work (SOP-QC-003 Recording a Deviation).

### Part 11 (e-signatures)

Stock Odoo provides:
- User authentication on every record change (via `res.users`)
- Chatter audit trail (timestamped, user-attributed)
- Workflow-based approvals (via `studio.approval.rule`)

Stock Odoo does NOT provide:
- Cryptographic signature binding (record hash + user signature)
- Reason-for-change capture as a structured field
- Periodic re-authentication for high-risk operations

The gap analysis needs to enumerate which Part 11 §11.10 / §11.30 / §11.50
controls are covered, partially covered, or not covered by stock Odoo,
then propose the closure plan (Studio approvals + chatter discipline for
the partial cases; custom module or third-party integration for the rest).

## Hand-off

When a finished CoA workflow is built and SOPs are authored, this skill
moves to its next minor release with the full recall + deviation +
e-signature section.

## References

- `odoo-data-migration/references/lot-tracking-flip-runbook.md` — getting from
  `tracking='none'` to `tracking='lot'` on a live database
- 21 CFR Part 111 (Dietary Supplement CGMP):
  <https://www.ecfr.gov/current/title-21/chapter-I/subchapter-B/part-111>
- 21 CFR Part 117 (Food CGMP):
  <https://www.ecfr.gov/current/title-21/chapter-I/subchapter-B/part-117>

## TODO before next minor

- CoA workflow (supplier inbound + entity outbound) end-to-end
- Recall procedure with the exact Odoo report combo
- Deviation tracking SOP + escalation rules
- Part 11 §11.10 / §11.30 / §11.50 mapping table: requirement → Odoo
  feature → SOP
- Quarantine location config + access rules
- Document control (Documents module vs external Knowledge module)
