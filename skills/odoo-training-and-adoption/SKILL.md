---
name: odoo-training-and-adoption
description: Use this skill for any work involving training a workforce on Odoo, building internal SOPs for Odoo workflows, change management, user adoption, or post-go-live support structures. Triggers include "Odoo training", "train the team on Odoo", "Odoo SOP", "Odoo workflow documentation", "change management", "user adoption", "Odoo runbook", "internal Odoo wiki", "Odoo onboarding", "SPoC training", "key user training", "end user training", "Odoo certification", "Odoo learning path", or any phrasing about getting people to actually use Odoo correctly day-to-day. Builds on top of the official Odoo Implementation Methodology — adoption is the third pillar, alongside scope and delivery. Do NOT use for technical Odoo configuration questions or schema decisions; those go to other skills in this family.
---

# Odoo Training & Adoption

This skill is about the human side of Odoo — getting the team to actually use it correctly, day after day, without external hand-holding. It is the single largest determinant of project success. Industry estimates put 75% of ERP failures down to poor user adoption rather than technical issues.

The skill produces a few categories of output:
1. **Role-based learning paths** — what the SPoC, key users, end users, and admins each need to know
2. **Internal SOPs** — workflow-specific written procedures keyed to Odoo screens
3. **Change management plan** — how the rollout is communicated, paced, and supported
4. **Post-go-live support structure** — who answers questions, how, and for how long
5. **Training assets** — quick-reference cards, video walkthroughs, screen recordings, FAQ pages

For regulated entities (e.g., dietary supplement manufacturers under 21 CFR Part 111): the training and SOP layer is also a regulatory artifact. 21 CFR Part 111 §111.12 requires written procedures for every operation that affects product quality. Done well, the Odoo SOPs *are* the GMP procedures for the operations Odoo touches.

## When to use this skill

- During Phase 2 (Kick-off) of an Odoo implementation: SPoC training kicks off here
- During Phase 3 (Implementation): key-user training runs in parallel with configuration
- During Phase 4 (Go-Live) and just after: end-user training, hyper-care, and SOP finalization
- Ongoing: when new hires join, new modules deploy, or processes change

If `odoo-implementation-methodology` hasn't been consulted yet, start there — it sets the project structure within which training happens.

## Three layers of competence

Build training around three layers, not four-letter Bloom's-taxonomy escalations:

**Layer 1 — Operate.** "I can do my daily job in Odoo without help." End users live here forever. This is the floor.

**Layer 2 — Adapt.** "When something doesn't fit the standard flow, I know where the levers are or who to ask." Key users and floor leads live here.

**Layer 3 — Configure.** "I can set up new products, change workflows, train others." SPoCs and admins live here.

Most training programs fail by trying to push everyone to Layer 3, which wastes time, demotivates Layer 1 users (who don't need it), and dilutes the message for everyone. Build the curriculum by layer, not by module.

## Role-based learning paths

### SPoC pathway (40–80 hours over 4–8 weeks)

The SPoC is the most important Odoo learner in the project. They need to reach Layer 3 in their primary modules and Layer 2 across the rest.

**Week 1:** Odoo orientation
- Sign up for `learn.odoo.com` (Odoo's official free eLearning)
- Complete the Odoo Functional Foundations track (sales, purchase, inventory at orientation level)
- Read the OIM document — they're now part of the methodology, not a passive recipient
- Set up a personal sandbox database to play in (Odoo Online free trial works)

**Weeks 2–3:** Module deep-dive in primary functional areas
- For a manufacturing-focused SPoC: deep-dive on Inventory, Manufacturing (MRP), Quality, Purchase
- Each module: read official docs end-to-end, then build a working example in the sandbox
- Pair with the Project Leader for at least 4 hours per module

**Weeks 4–5:** Cross-module flows
- Procure-to-Pay end-to-end
- Order-to-Cash end-to-end
- Receive-to-Ship end-to-end (manufacturing flow)
- Walk these flows in the project's actual staging environment

**Weeks 6–7:** Configuration and admin
- User roles, access rules, multi-company access
- Studio basics — building forms, reports, automated actions
- Data import via UI and via XML-RPC at conceptual level
- Backup and restore procedures (Odoo Online vs Odoo.sh vs self-hosted)

**Week 8:** Train-the-trainer prep
- Build the materials they'll use to train key users and end users
- Practice the training sessions on the Project Leader as a dry run

**Optional capstone:** sit for the Odoo Functional Certification exam (1.5 hours, 120+ questions, 70% to pass). Not required, but a useful forcing function and gives the SPoC a portable credential.

### Key user pathway (16–24 hours over 2–3 weeks)

Key users target Layer 2 in their own area, Layer 1 in adjacent areas.

**Session 1 (4 hours):** Odoo orientation + their own module's daily flows.
**Session 2 (4 hours):** Edge cases in their module — returns, corrections, exceptions, how to escalate.
**Session 3 (4 hours):** Adjacent modules at orientation level — what the next-station downstream sees from their work.
**Self-paced (4–8 hours):** working through `learn.odoo.com` modules relevant to their role.
**On-the-job (ongoing):** running test scenarios in staging with the SPoC.

### End user pathway (2–6 hours, role-specific)

End users target Layer 1 only. Keep it tight, keep it daily-job-relevant, do not lecture about "the philosophy of ERP."

**Format:** 1–2 short sessions (1–2 hours each), screen-share or in-person, focused on the 5–10 things this person does every day.

**Materials:** quick-reference card (one page per task), screen recording (90 seconds per task), and access to the SOP wiki.

**Timing:** within 1 week of go-live, not earlier (they'll forget) and not later (they'll be unprepared).

### Admin / IT pathway (variable; depends on deployment)

For Odoo Online: minimal — credential management, billing, support escalation.
For Odoo.sh: build/branch model, staging→production promotion, log access, custom module install.
For self-hosted: PostgreSQL, Odoo upgrade procedure, backup/restore, server tuning, OCA module installation.

## Internal SOP system

The SOP is the written procedure for a workflow. For regulated entities (e.g., supplement manufacturers under Part 111), SOPs are mandatory. For everyone else, they're the difference between "trained team" and "trained team that survives turnover."

### SOP structure (template at `references/sop-template.md`)

Every SOP has these sections:

1. **Header:** SOP number, title, version, effective date, owner, approver
2. **Purpose:** what this SOP is for, in 2 sentences
3. **Scope:** who follows it, when
4. **Definitions:** terms used in the SOP
5. **Roles & responsibilities:** who does what
6. **Procedure:** numbered steps, with screenshots where helpful
7. **Records:** what gets logged where (Odoo records this auto-generates)
8. **References:** related SOPs, regulations, work instructions
9. **Revision history:** what changed and when

### SOP catalog (Phase I scope, illustrative for a regulated manufacturer)

```
SOP-INV-001  Receiving Raw Materials (with QA hold)
SOP-INV-002  Releasing Quarantined Materials
SOP-INV-003  Customer-Owned Material Receipt (using owner_id)
SOP-MFG-001  Initiating a Manufacturing Order
SOP-MFG-002  Recording In-Process Quality Checks
SOP-MFG-003  Closing a Manufacturing Order and Lot Creation
SOP-QC-001   Performing Incoming QC Inspection
SOP-QC-002   Issuing a Certificate of Analysis
SOP-QC-003   Recording a Deviation
SOP-QC-004   Initiating a Recall (forward and back tracing)
SOP-PUR-001  Placing a Purchase Order
SOP-PUR-002  Receiving and Three-Way Matching
SOP-SAL-001  Creating a Sales Order from a Contract Customer
SOP-SAL-002  Releasing Finished Goods for Shipment
SOP-ACC-001  Period Close Procedure
```

These should be authored *during* implementation (when the workflows are being built), not after. The team that builds the workflow writes the SOP for it. SOP authoring is part of the definition of done for each module.

## Change management plan

The plan is not a slide deck — it's a sequence of communication and feedback loops, sized to the org.

### SME-scale change management (5–25 employees)

**T-8 weeks (kickoff):** all-hands meeting. Sponsor announces the project, names the SPoC, explains why Odoo and what the team will see. Take questions. Acknowledge that change is hard.

**T-6 weeks:** SPoC starts informal "what do you wish your tools did?" conversations with each functional team. This is not requirements gathering — it's making people feel heard.

**T-4 weeks:** key user training begins. Each module's key user gets to be the "expert" for their team — gives them ownership.

**T-2 weeks:** end-user training scheduled. Communicated repeatedly so it's expected.

**T-1 week:** training sessions. Quick-reference materials distributed.

**T-0 (go-live):** all-hands check-in. SPoC visible on the floor. Project Leader on-call.

**T+1 week:** retrospective with key users. What's working, what's not. Adjust quickly.

**T+1 month:** retrospective with end users. Same questions.

**T+3 months:** transition to steady-state. SPoC is now self-sufficient. Project Leader stepping back.

### Larger orgs

Add: dedicated project comms channel (Slack, Teams), weekly project newsletters, named change champions per department, executive steering committee meeting cadence.

## Common adoption failures and counters

| Failure | Counter |
|---|---|
| Users abandon Odoo for spreadsheets within 60 days | Make Odoo's reports better than spreadsheets *during* implementation; ban spreadsheet workarounds explicitly post-go-live; SPoC investigates every "I have to use a spreadsheet because..." complaint |
| New hires aren't trained, drift to bad habits | New-hire onboarding includes an Odoo session in week 1; SPoC owns this process |
| Knowledge centralizes in one person; they leave; everything breaks | Always have backup SPoC; document everything; rotate "expert of the month" roles for training resilience |
| Training happens once, never refreshes; team forgets | Quarterly 30-minute refreshers per role; "Odoo tip of the week" in the company chat |
| End users mock the system; resistance becomes culture | Sponsor must visibly use Odoo themselves; if leadership doesn't use it, the team won't |

## Training asset library structure

Build a single internal wiki (Odoo's own Knowledge module works, or Notion / Confluence). Structure:

```
/Odoo-Wiki
├── Start-Here              (orientation for new users)
├── Quick-Reference-Cards   (1-page-per-task PDFs)
├── Screen-Recordings       (90-second clips per task)
├── SOPs                    (formal procedures, for regulated workflows)
├── Module-Guides           (one per module: Sales, Purchase, etc.)
├── Troubleshooting         (FAQ + known issues + workarounds)
├── Release-Notes           (what changed in our Odoo when, who to contact)
└── Training-Calendar       (upcoming sessions, recordings of past sessions)
```

The SPoC owns this wiki. It's their "second job" forever.

## What this skill does not do

- Replace `learn.odoo.com` — point people there for general Odoo training. This skill is about *your specific* entity's training, not generic Odoo training.
- Configure Odoo for training purposes — that's `odoo-schema-design` plus the implementation skills.
- Solve the underlying methodology — that's `odoo-implementation-methodology`. If the project doesn't have a SPoC and a phased plan, training won't save it.

## References

- `references/sop-template.md` — formal SOP template for regulated entities
- `references/spoc-charter-template.md` — SPoC role definition template
- `learn.odoo.com` for official Odoo training
- Odoo Functional Certification: <https://www.odoo.com/slides/odoo-19-functional-certification-502>
