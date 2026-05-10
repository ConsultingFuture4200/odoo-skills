# SOP Template (regulatory-grade)

For 21 CFR Part 111 / 117 regulated workflows. Non-regulated entities can use a lighter version — drop the Records, Approvals, and full Revision History sections if not needed.

Save as `SOP-{AREA}-{NUMBER}-v{VERSION}.md` (e.g., `SOP-INV-001-v1.0.md`).

---

# SOP-{AREA}-{NUMBER}: {TITLE}

|  |  |
|---|---|
| **Document number** | SOP-{AREA}-{NUMBER} |
| **Version** | v{N.N} |
| **Effective date** | YYYY-MM-DD |
| **Supersedes** | SOP-{AREA}-{NUMBER} v{N-1.N} (if applicable) |
| **Document owner** | (Role, not person — e.g., "Quality Manager") |
| **Approver** | (Role) |
| **Review cycle** | Annual (or trigger-based — e.g., "on Odoo major version upgrade") |
| **Applies to Odoo version** | (e.g., 18.0) |

## 1. Purpose

Two sentences. Why does this SOP exist. What does it ensure.

> Example: "This SOP defines the procedure for receiving raw materials and placing them on Quality hold pending QA release. It ensures compliance with 21 CFR Part 111 §111.155 (written procedures for components) and prevents unreleased material from being used in production."

## 2. Scope

Who follows this SOP and when. Be specific about boundaries.

> Example: "This SOP applies to all incoming receipts of active ingredients, excipients, and packaging materials handled at the primary manufacturing facility. It does not apply to customer-owned materials (see SOP-INV-003)."

## 3. Definitions

Define any term used in the SOP that a new employee might not know.

| Term | Definition |
|---|---|
| Lot | A batch of material from a single vendor receipt that shares an identical CoA. |
| Quarantine | The physical and logical state of material that has been received but not released for use. |
| Released | QA-approved for use in production. |

## 4. Roles & responsibilities

| Role | Responsibility |
|---|---|
| Receiving Operator | Performs the receipt; assigns lot number; moves material to Quarantine. |
| QA Inspector | Reviews CoA; samples per spec; performs QC; records results in Odoo. |
| QA Manager | Reviews QC results; releases or rejects lot; signs off in Odoo. |

## 5. Procedure

Numbered steps. Include screenshots or screen-recording links where helpful. Reference Odoo screens by their menu path.

### 5.1 Receiving Operator: physical receipt
1. Inspect the shipment for visible damage. If damaged, follow SOP-INV-006 (Damaged Receipt).
2. Locate the matching Purchase Order in Odoo: **Purchase ▸ Orders ▸ Purchase Orders**, search by vendor and PO number.
3. Click **Receive Products**.
4. For each line, enter the actual received quantity. If different from PO, follow SOP-INV-007 (Quantity Discrepancy).
5. Assign a lot number using the convention: `{VENDOR-CODE}-{YYYYMMDD}-{SEQ}`.
6. Confirm the destination location is `WH/Stock/Receiving/Quarantine`.
7. Click **Validate**.
8. Print the lot label and affix to the material. Move material to the Quarantine area.

### 5.2 QA Inspector: incoming QC
1. Open **Quality ▸ Quality Checks**, filter to Pending.
2. (continue with steps...)

### 5.3 QA Manager: release decision
1. (continue with steps...)

## 6. Records

What gets logged where. For Part 111 compliance, every record must be retrievable and tied to a specific lot.

| Record | Where it lives | Retention |
|---|---|---|
| Receipt transaction | Odoo `stock.picking` + `stock.move.line` | 7 years (Part 111 §111.605) |
| Lot creation | Odoo `stock.lot` | 7 years |
| QC results | Odoo Quality Check + linked CoA in Documents | 7 years |
| QA Manager release signature | Odoo chatter + electronic signature module | 7 years |

## 7. References

- 21 CFR Part 111 §111.155 (Components)
- 21 CFR Part 111 §111.605 (Records retention)
- SOP-INV-003 (Customer-Owned Material Receipt)
- SOP-INV-006 (Damaged Receipt)
- SOP-INV-007 (Quantity Discrepancy)
- SOP-QC-001 (Incoming QC Inspection)

## 8. Revision history

| Version | Date | Author | Approver | Changes |
|---|---|---|---|---|
| v1.0 | YYYY-MM-DD | | | Initial release |
| v1.1 | YYYY-MM-DD | | | Updated lot numbering convention; aligned with Odoo version-X lot field changes |

## 9. Approval signatures

| Role | Name | Signature | Date |
|---|---|---|---|
| Document owner | | | |
| Approver | | | |
| QA Manager (independent review) | | | |

---

**Note on electronic signatures (21 CFR Part 11):** if signatures are captured electronically in Odoo, the system must meet Part 11 controls (audit trail, identity verification, intent statement). For a supplement manufacturer (Part 111), Part 11 applies only to records that exist *only* electronically and are used to satisfy Part 111 record requirements. A wet-ink approval scanned and stored alongside the Odoo record is the simplest way to sidestep Part 11 controls if the e-signature infrastructure isn't ready yet.
