---
name: odoo-schema-design
description: Use this skill whenever the user is structuring an Odoo instance, mapping existing business data into Odoo's data model, or making schema decisions in Odoo. Triggers include any mention of "set up Odoo", "structure Odoo", "map our data to Odoo", "Odoo chart of accounts", "product variants vs. separate products", "BOM structure", "warehouse/location hierarchy", "multi-company Odoo", "lot tracking", "Odoo schema", or "how should we model X in Odoo". Also trigger for adjacent phrasings like "we have an existing Odoo instance and need to clean it up", "migrating from QuickBooks/NetSuite to Odoo", or "Odoo isn't fitting our business correctly". Anchored to manufacturing/CPG use cases (contract manufacturing, GMP-regulated supplements) but the decision frameworks generalize. Do NOT trigger for pure Odoo development (custom modules in Python), Odoo UI/UX questions unrelated to data model, or end-user "how do I click X" questions.
---

# Odoo Schema Design

Decision frameworks for structuring an Odoo instance correctly the first time, or remediating one that wasn't. The wrong Odoo schema costs months of cleanup; this skill exists to slow you down at the decision points that matter and speed you through the ones that don't.

This skill produces **decisions and configuration plans**, not code. The output is typically a markdown decision record that downstream skills (`odoo-data-migration`, `odoo-api-sync`) consume.

## When to use this skill

Use this skill before:
- Loading any data into Odoo for the first time
- Adding a new business entity, brand, or product line to an existing instance
- Major Odoo migrations (QuickBooks → Odoo, NetSuite → Odoo, version upgrades that change data model)
- Resolving "Odoo doesn't fit our business" frustrations — usually a schema problem, not a software problem

Do not use this skill for tactical "how do I configure X menu" questions — those are Odoo documentation lookups.

## Workflow

1. **Inventory the business.** Before recommending anything, understand the entity: how products are sold, manufactured, accounted for. Use the questionnaire in `references/business-inventory.md`.
2. **Inventory the existing Odoo state** (if one exists). What's already configured, what's loaded, what's broken. Use `references/odoo-audit-checklist.md`.
3. **Walk the seven decision frameworks** below in order. Each gates the next. Skipping ahead causes rework.
4. **Produce a Schema Decision Record** (template at `references/sdr-template.md`) — one markdown file the team can review before any data movement happens.
5. **Hand off to `odoo-data-migration` or `odoo-api-sync`** with the SDR as input.

## The seven decision frameworks

These are the high-leverage decisions. Get these right and the rest is execution. Get any of them wrong and you'll feel it for years.

### 1. Multi-company strategy

**The question:** when a business has multiple legal entities, do they live in one Odoo company, multiple companies in one database, or multiple databases?

**The three options:**

| Pattern | When it fits | Cost |
|---|---|---|
| **Single company, analytic tags** | Entities are operational departments, not legal entities. Shared chart of accounts. | Low setup; loses legal/tax separation; can't run separate financials. |
| **Multi-company, single database** | Separate legal entities that share customers/vendors/inventory. Inter-company transactions are common. | Medium setup; requires careful access rules; supports inter-company moves natively. |
| **Multiple databases** | Entities are operationally independent. Separate teams. Different countries/currencies. M&A in progress. | High setup; no inter-company automation; but full isolation. |

**Decision artifact:** list every legal entity, its currency, its country, its tax regime, and which other entities it transacts with. Inter-entity transaction frequency is the deciding factor.

See `references/multi-company-deep-dive.md` for inter-company configuration patterns.

### 2. Chart of accounts

**The question:** localized template, custom-built, or migrated-as-is from the prior accounting system?

**Default recommendation:** start with Odoo's localized template for the country (US: l10n_us), then **add** accounts your business needs, but never **remove** accounts the localization expects (you'll break tax reports). Map prior-system accounts to localization accounts wherever the semantics align.

**The trap:** importing the QuickBooks COA verbatim. QuickBooks COAs are usually flat, often have duplicate accounts created by accident, and use numbering schemes that don't match Odoo's structure (header accounts vs. detail accounts, account types like "Income" vs. Odoo's `income`/`income_other` distinction).

**Required output of this decision:**
- Localization template selected
- A mapping table: `legacy_account_code → odoo_account_code | odoo_account_type | notes`
- Decisions on account hierarchy (Odoo uses prefix-based grouping)
- Decisions on whether to migrate historical journal entries or only opening balances

See `references/coa-mapping-patterns.md` for QuickBooks→Odoo and NetSuite→Odoo mapping templates.

### 3. Product modeling: variants vs. separate products

**The question:** "Brand X Sleep Gummy 60ct" and "Brand X Sleep Gummy 30ct" — are these (a) two variants of the same product template, or (b) two separate product templates?

**This decision affects:**
- BOM structure (one BOM per template with variant attributes, vs. one BOM per product)
- Inventory reports
- Sales reports
- Pricing rules
- E-commerce display
- Customer-facing SKU strategy

**Decision rule:** use variants when the items share manufacturing process, packaging line, and customer perception of "same product, different size/flavor." Use separate products when the items have meaningfully different BOMs, different regulatory treatment, or are sold to different customers.

**For contract manufacturing specifically:** the right answer is usually **separate products per customer per SKU**. Reason: the contract manufacturer is making "Brand X's vitamin C gummy" — the BOM is owned by Brand X, the customer ID is part of the product's identity, and aggregating across customers in reports is rarely useful. Variants would muddy traceability.

**For an own brand:** variants make sense for a flagship SKU with size variants `30ct` / `60ct` if the BOM scales linearly. If the 60ct uses a different bottle vendor, separate products.

**Decision artifact:** for each product family, document:
- Variant axes (size, flavor, customer, formulation revision) — or "none, separate products"
- Whether attributes are display-only or affect BOM/cost
- SKU naming convention (Odoo `default_code`)
- External ID convention (`xml_id` — critical for idempotent imports later)

See `references/product-variants-decision-tree.md` for the full decision tree with worked examples.

### 4. BOM strategy

**The question:** flat BOMs, multi-level BOMs, phantom BOMs, or kit BOMs — and how do you handle customer-supplied versus in-house raw materials?

**The four BOM patterns in Odoo:**

- **Manufacturing BOM (normal):** consumes components, produces a manufactured product. Default for production runs.
- **Phantom BOM (set):** consumed at sale time, no separate manufacturing order. Useful for assemblies that aren't really manufactured ("variety pack of 5 flavors").
- **Kit:** similar to phantom — sold as one SKU, fulfilled as components.
- **Subcontracting BOM:** Odoo's native model for "we send raw materials to a subcontractor, they make and ship the finished good." Different mechanics from a normal MO.

**For contract manufacturing — direction matters:** a contract manufacturer (CMO) is the *subcontractor* from the brand customer's perspective. **In the CMO's own Odoo, that means normal Manufacturing BOMs, not Subcontracting BOMs.** Subcontracting routes in Odoo are designed for the *contracting company's* view ("we outsource production to a vendor"); the CMO is the vendor in that picture, so those routes do not apply to the CMO's own database. This is a frequent and expensive misunderstanding — getting it wrong leads to inventory that sits in confusing virtual locations and PO/MO mechanics that fight the actual physical process.

**Customer-supplied raw materials — two valid patterns.** When a brand customer supplies their own ingredients for the CMO to manufacture with, those materials must be physically present and traceable but should not appear on the CMO's books as the CMO's inventory. Two Odoo-native approaches:

- **Pattern A (recommended default): `stock.quant.owner_id`.** Receive the material into a normal warehouse location, but on the receipt set the Owner field to the customer. The material is physically managed but financially belongs to the customer; Odoo's reports cleanly distinguish "owned by us" vs "owned by Brand Y." This is the lighter-weight approach and works for most cases.
- **Pattern B (when complexity warrants): dedicated customer-owned location.** Create a child location under Raw Materials per customer (e.g., `WH/Stock/RawMaterials/CustomerOwned/BrandX`). This gives stronger physical/logical separation and clearer reports, at the cost of more locations to manage and more careful access rules. Use this when (a) multiple brand customers' materials might be physically commingled and need software-level segregation, or (b) the audit trail needs to be airtight for regulatory or contractual reasons.

Pattern A is recommended unless there's a specific reason to choose B. Both are documented in detail in `references/customer-owned-inventory-patterns.md`.

**Multi-level vs. flat:** if you have intermediate WIP that's stocked, tested, or held (e.g., "gummy slab pre-cut" → "cut gummies" → "bottled finished good"), use multi-level. If everything happens in one continuous run, flat is fine. Multi-level supports better lot tracking through stages.

**Decision artifact:** for each finished good:
- BOM type (normal / phantom / kit / subcontracting)
- Number of levels
- Components, with for each: customer-owned or in-house, lot-tracked or not, expected scrap percentage, routing operation
- Routing/work centers if using MRP routings

See `references/bom-patterns-contract-manufacturing.md`.

### 5. Warehouse and location hierarchy

**The question:** what stock locations exist, and how is material moved between them?

**Required locations for any GMP-regulated manufacturer:**

```
WH (Warehouse)
├── Stock
│   ├── Receiving                  (inbound, awaiting QA release)
│   ├── Quarantine                 (failed QA or under investigation — REQUIRED for GMP)
│   ├── Raw Materials (released)   (QA-released, available for production)
│   │   ├── Customer-Owned         (segregated by customer — see BOM strategy)
│   │   └── In-House
│   ├── WIP                        (in-process)
│   ├── Finished Goods (held)      (manufactured, awaiting QA release for shipping)
│   ├── Finished Goods (released)  (released, available for shipping)
│   └── Rejected/Disposal          (failed final QA)
└── Output
    └── Shipping Dock
```

**Why this matters:** the FDA expects you to demonstrate that quarantined material cannot be inadvertently used in production. In Odoo, that means real `stock.location` records with access rules — not a status field on a product.

**Customer-owned segregation:** within Raw Materials (released), create a child location per customer if customer-owned inventory exists. This is the cleanest way to report on "what does Brand Y own that we're holding?" without mixing it with the CMO's books.

**Decision artifact:** the full location tree, with for each location: parent, type (`internal` / `view` / `transit` / `customer` / `supplier`), whether it's a counted location, and access rules.

See `references/warehouse-location-patterns.md` for the canonical contract-manufacturing tree and variants for B2B ingredient suppliers and Amazon-fulfilled brands.

### 6. Lot/serial tracking strategy

**The question:** which products require lot tracking, which require serial tracking, and which require neither?

**Defaults for a GMP-regulated supplement manufacturer:**

- **Raw materials:** lot tracking **mandatory**. CoA from supplier maps to lot.
- **Finished goods:** lot tracking **mandatory**. Customer recall capability requires it.
- **WIP / intermediate products:** lot tracking strongly recommended for traceability through stages.
- **Packaging materials (bottles, caps, labels):** lot tracking by exception — only if regulatory or specific customer requirement. Default off; tracking every cap by lot is expensive.
- **Serial tracking:** essentially never for gummies. Reserved for unique-instance items (capital equipment, regulated devices).

**The cost of lot tracking:** every receipt, every move, every production output requires a lot value. This is real friction at the warehouse floor. Don't apply it where you don't need it; apply it rigorously where you do.

**Decision artifact:** for each product category, document `tracking` value (`none` / `lot` / `serial`) and the rationale (regulatory / traceability / customer requirement / off).

See `odoo-gmp-compliance` skill for the full lot strategy including expiration date handling and CoA-to-lot binding.

### 7. External ID (xml_id) convention

**The question:** the most boring decision; the one that hurts most when skipped.

**What it is:** every record in Odoo can have a stable text identifier (`__import__.product_brand_x_sleep_60ct`) that survives database moves, refreshes, and re-imports. Without it, a re-run of a CSV import creates duplicates instead of updating.

**The convention:**

```
{namespace}.{entity}_{stable_business_key}
```

For a typical project:

```
__{entity}__.product_{customer_code}_{sku}
__{entity}__.partner_{partner_type}_{slug}
__{entity}__.bom_{product_xml_id_suffix}_{revision}
__{entity}__.location_{warehouse}_{location_path}
```

**The rule:** every record created by automation (migration, sync, agent) gets an external ID. Records created manually in the UI get one too if they'll ever be referenced by automation.

**Critical constraint that breaks naive schemes:** Odoo External IDs must be **unique across all models**, not just unique within their own model. You cannot have `partner_1` for a partner and `product_1` for a product — they collide because Odoo stores them in a single `ir.model.data` table. The convention above prevents this by including the model in the entity portion (`partner_customer_brandx` vs `product_template_brandx_vitaminc_60ct` — these can never collide because they have different prefixes regardless of slug).

**Why this is in the schema skill, not the migration skill:** because choosing the convention is a one-time schema decision. Once you start importing with one pattern and switch midstream, you've created two duplicate sets.

See `references/external-id-conventions.md`.

### Pre-decision: methodology and scope

This skill assumes `odoo-implementation-methodology` has been consulted first and a GAP analysis exists. Schema design happens *within* the scope set by the GAP analysis — you decide schema for the modules and processes that are in scope, not for everything Odoo can do.

If no GAP analysis exists, stop and produce one. Schema decisions made without scope are guesses.

## The Schema Decision Record (SDR)

Output of this skill is always a single markdown file: the SDR.

**Naming:** `odoo-sdr-{entity}-v{version}.md`

**Required sections:** see `references/sdr-template.md`. The template walks through all seven frameworks and forces a decision (or a documented "deferred — see issue #N") on each.

The SDR is the single source of truth that downstream skills consume. Never start data movement without one.

## Working with an existing Odoo instance

When the instance already has data, adapt the workflow:

1. Run the audit (`references/odoo-audit-checklist.md`) and produce an **Existing State Report**.
2. Walk the seven frameworks in **gap analysis mode**: for each, determine "what's the current state, what's the right state, what's the migration path?"
3. The SDR has an extra section: **Remediation Plan** — sequence of changes, dependencies between them, and risk assessment for each.
4. Some changes are cheap (rename a location). Some are expensive (split a product into separate products per customer — breaks all existing MOs). The remediation plan must call out which is which.

## What this skill does not do

- It does not write the configuration files or CSVs. That's `odoo-data-migration`.
- It does not connect to Odoo. That's `odoo-connect`.
- It does not configure GMP-specific quality flows. That's `odoo-gmp-compliance` — though decisions made here (lot tracking, location hierarchy) feed into it.
- It does not customize Odoo with new modules. Schema design works within stock Odoo's data model. If a decision points to "we need a custom module," document it in the SDR but treat the custom-module work as out of scope for this skill family.

## References

- `references/sdr-template.md` — the SDR template every project produces
- `references/worked-examples.md` — concrete answers a contract manufacturer of dietary supplements gave to each of the seven frameworks during a real catalog migration; use as a worked example when answering the same question for another entity
