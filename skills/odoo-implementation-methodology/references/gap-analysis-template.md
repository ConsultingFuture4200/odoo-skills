# GAP Analysis Template

Replace `{ENTITY}` with the business name. Save as `gap-analysis-{entity-slug}-v{N}.md`.

The GAP analysis is short by design. If yours is over ~10 pages, the project has scope problems, not documentation problems.

---

# GAP Analysis — {ENTITY}

**Date:**
**Author:**
**Status:** Draft | Reviewed | Signed
**Target Odoo version:**
**Target deployment:** Odoo Online | Odoo.sh | Self-hosted

## 1. Business overview (max 1 page)

What does this entity do, in plain language. What pain does Odoo need to solve. What success looks like 6 months after go-live.

## 2. Functional area summary

One paragraph per area. Touch only areas in scope.

### 2.1 Sales & CRM
### 2.2 Purchase
### 2.3 Inventory
### 2.4 Manufacturing
### 2.5 Quality (if regulated)
### 2.6 Accounting
### 2.7 HR / Payroll (if in scope)
### 2.8 Other

## 3. Requirements list

Classify every requirement. Use one of: `STANDARD`, `STUDIO`, `CUSTOM`, `DROP`, `DEFER`.

| # | Requirement | Functional area | Classification | Notes |
|---|---|---|---|---|
| 1 | Lot tracking on raw materials | Inventory | STANDARD | native Odoo |
| 2 | Customer-owned material segregation | Inventory | STUDIO | use stock.quant.owner_id, light Studio for the receipt form |
| 3 | Auto-generate CoA PDF on lot release | Quality | CUSTOM | no standard; OCA option being investigated |
| 4 | Sync historical orders from QuickBooks (3 yrs back) | Accounting | DROP | OIM principle — avoid importing data history |
| 5 | Per-customer pricing tiers with quantity breaks | Sales | DEFER | Phase II |

**Distribution check:** the OIM expectation is roughly:
- 50% STANDARD (target — push higher if possible)
- 15–20% STUDIO
- 5–10% CUSTOM (target — push lower)
- 30% DROP or DEFER

If your distribution is heavily weighted toward CUSTOM, the project is at high risk. Re-examine each CUSTOM line and ask: can this be solved by changing the *process* instead of the *software*?

## 4. SPoC nomination

**Nominated SPoC:**
**Role at company:**
**Time commitment:** % of work week dedicated to the project
**Decision authority:** [explicit statement of what the SPoC can decide without escalation]
**Backup:** [if SPoC is unavailable]
**Training plan:** [link to `odoo-training-and-adoption` SPoC pathway]

**SPoC charter signed by:** [Sponsor], [SPoC] — both must sign.

## 5. Phasing plan

**Phase I scope** (target go-live: YYYY-MM-DD):
- (list of requirement numbers from §3)

**Phase II scope** (target: YYYY-MM-DD or "after Phase I stable"):
- (list of requirement numbers)

**Out of scope** (will not be implemented):
- (list of requirement numbers, with rationale for each)

## 6. Effort and budget envelope

| Phase | Estimated weeks | Estimated hours | Rate basis | Budget envelope |
|---|---|---|---|---|
| GAP Analysis (this document) | | | | |
| Kick-off | | | | |
| Implementation | | | | |
| Go-Live + hyper-care | | | | |
| **Total Phase I** | | | | |

Include both internal time (SPoC, key users) and external time (Project Leader, etc.).

## 7. Risks and assumptions

| # | Risk or assumption | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| 1 | SPoC may not have enough time | Medium | High | Sponsor commits to backfill X% of SPoC's normal duties |
| 2 | Existing data quality unknown | High | Medium | Phase 0.5 mini-audit before kickoff |

## 8. Out-of-scope explicit statements

State clearly what this project will NOT do, to prevent scope-creep later. Example:
- "This project will not migrate historical accounting data prior to YYYY-MM-DD."
- "This project will not build a customer-facing portal in Phase I."
- "This project will not integrate with [system X] in Phase I."

## 9. Sign-off

| Role | Name | Date | Signature |
|---|---|---|---|
| Sponsor | | | |
| SPoC | | | |
| Project Leader | | | |

No sign-off, no kickoff. No exceptions.
