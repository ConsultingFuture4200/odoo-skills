# Schema Decision Record (SDR) Template

Replace `{ENTITY}` with the business entity. Replace `{VERSION}` with the SDR version (start at v0.1.0). Save as `odoo-sdr-{entity-slug}-v{version}.md`.

---

# Odoo Schema Decision Record — {ENTITY} v{VERSION}

**Date:** YYYY-MM-DD
**Author:**
**Status:** Draft | Under Review | Approved | Superseded by v{NEXT}
**Odoo version:** (e.g., 18.0 Enterprise)
**Deployment:** Odoo Online | Odoo.sh | Self-hosted
**Database name:**
**Existing state:** Greenfield | Existing instance with data | Migrating from {prior system}

## 0. Executive summary

One paragraph. The non-obvious decisions and why. A reader who only reads this section should be able to repeat the three biggest schema choices.

## 1. Multi-company strategy

**Decision:** Single company / Multi-company single DB / Multiple databases

**Companies in scope:**

| Legal entity | Currency | Country | Tax regime | Transacts with |
|---|---|---|---|---|
| | | | | |

**Inter-company transaction patterns:**

**Rationale:**

**Risks / open questions:**

---

## 2. Chart of accounts

**Localization template:** (e.g., `l10n_us`)

**Strategy:** Use template as-is | Template + extensions | Hybrid (template for tax accounts, custom for operational)

**Mapping table** (legacy → Odoo):

| Legacy code | Legacy name | Odoo code | Odoo type | Notes |
|---|---|---|---|---|
| | | | | |

**Historical data treatment:** Opening balances only | Full GL history | Per-entity decision

**Open items:**

---

## 3. Product modeling

**For each product family, the variant decision:**

| Family | Pattern | Variant axes | SKU convention | Rationale |
|---|---|---|---|---|
| (e.g., contract-manufactured products) | Separate products | none | `{customer_code}-{sku}` | Per-customer traceability; BOM owned by customer |
| (e.g., own-brand sleep gummy) | Variants | size (30ct/60ct) | `OWN-SLEEP-{size}` | Same BOM scaled; same packaging line |

**Default code (SKU) convention:** describe globally and per family if it differs.

**External ID prefix:** (must match decision in §7)

**Open items:**

---

## 4. BOM strategy

**For each finished good or product family:**

| Product / family | BOM type | Levels | Customer-owned components? | Notes |
|---|---|---|---|---|
| | normal / phantom / kit / subcontracting | flat / multi-level | yes/no | |

**Routing decisions:** per family, are routings used? Which work centers?

**Customer-owned material handling:** describe the location pattern (cross-references §5).

**Scrap and yield handling:** standard scrap percentages, where they're configured.

**Open items:**

---

## 5. Warehouse and location hierarchy

**Warehouse(s):**

| Code | Name | Address | Notes |
|---|---|---|---|
| | | | |

**Location tree** (full ASCII tree):

```
WH
├── Stock
│   ├── ...
```

**For each non-trivial location:**

| Location | Type | Parent | Counted? | Access rule | GMP role |
|---|---|---|---|---|---|
| | internal/view/transit | | yes/no | | quarantine / release / etc. |

**Customer-owned segregation:** describe the per-customer location pattern if applicable.

**Open items:**

---

## 6. Lot/serial tracking

| Product category | Tracking | Expiration tracked? | Rationale |
|---|---|---|---|
| Raw materials (active ingredients) | lot | yes | GMP / CoA binding |
| Raw materials (excipients) | lot | yes | GMP |
| Packaging — primary | lot | no | Customer recall traceability |
| Packaging — secondary | none | n/a | Cost vs. benefit |
| WIP | lot | no | Stage traceability |
| Finished goods | lot | yes | Recall + expiration |

**Lot naming convention:**

**CoA / spec sheet binding strategy:** (cross-references `odoo-gmp-compliance` skill)

**Open items:**

---

## 7. External ID (xml_id) convention

**Namespace:** `__{entity_slug}__` (e.g., `__exampleco__`)

**Patterns by record type:**

| Record type | Pattern | Example |
|---|---|---|
| `product.template` | `__{entity}__.product_template_{customer}_{sku}` | `__exampleco__.product_template_brandX_vitaminc_60ct` |
| `product.product` | `__{entity}__.product_{customer}_{sku}` | |
| `res.partner` | `__{entity}__.partner_{type}_{slug}` | `__exampleco__.partner_customer_brandx` |
| `mrp.bom` | `__{entity}__.bom_{product_suffix}_{revision}` | |
| `stock.location` | `__{entity}__.location_{warehouse}_{path}` | |
| `account.account` | `__{entity}__.account_{code}` | |

**Rule:** every record created by automation gets an xml_id. Records created in the UI that will be referenced by automation also need xml_ids (assigned via Developer Mode).

---

## 8. Existing state assessment (only if remediating)

**Audit summary:** (link to detailed audit report)

**Material gaps from desired state:**

| # | Current state | Desired state | Cost to remediate | Risk if deferred |
|---|---|---|---|---|
| 1 | | | low/med/high | |

---

## 9. Remediation plan (only if remediating)

Sequenced list of changes. Each change has: owner, dependency on prior changes, risk of data loss, rollback plan.

| # | Change | Depends on | Owner | Risk | Rollback |
|---|---|---|---|---|---|
| | | | | | |

---

## 10. Hand-off to downstream skills

**To `odoo-data-migration`:** (which CSVs need to be produced, against which schema, in what order)

**To `odoo-api-sync`:** (which entities will be kept in sync after initial load, with what frequency, from what source)

**To `odoo-gmp-compliance`:** (which compliance overlays apply: quality checks, deviation tracking, CoA workflow)

**To `odoo-webhooks`:** (which Odoo events trigger external systems; which external events trigger Odoo writes)

---

## Appendix A — decisions deferred

| # | Question | Why deferred | Decision needed by |
|---|---|---|---|

## Appendix B — references

- (link to existing Odoo audit report)
- (link to source-system schema docs: QuickBooks COA export, etc.)
- (link to relevant skill reference files actually consulted)
