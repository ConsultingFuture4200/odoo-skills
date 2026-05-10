---
name: odoo-implementation-methodology
description: Use this skill at the start of any Odoo implementation, re-implementation, or major scope-expansion. Triggers include "implement Odoo", "Odoo project plan", "Odoo kickoff", "Odoo SPoC", "Odoo go-live", "Odoo GAP analysis", "Odoo project methodology", "phase our Odoo rollout", "Odoo project failing", "rescue Odoo project", or "what's the right way to roll out Odoo". This is the gating skill — `odoo-schema-design` and all downstream skills should be sequenced inside the phases this skill defines. Encodes the official Odoo Implementation Methodology (OIM) from Odoo S.A.: SPoC-led, four phases (GAP Analysis → Kick-off → Implementation → Go-Live), Standard-before-Studio-before-Custom, "Replace, don't replicate," and "avoid importing data history." Do NOT use for tactical configuration questions or in-Odoo how-to questions — those go to other skills or Odoo documentation.
---

# Odoo Implementation Methodology (OIM)

This skill encodes the methodology that Odoo S.A.'s own QuickStart team uses to deliver implementations on time and on budget. It is opinionated, tested across thousands of Odoo deployments, and the source of most "why is our Odoo project taking forever" failures when ignored.

The principles are uncomfortable. They will tell you to drop scope, to refuse customer requests, to skip importing your historical data, to challenge the customer's process rather than replicate it. Discomfort is the point — these principles exist because the *comfortable* path (yes-and the customer, replicate the legacy system, customize everything) is the path that produces 50%+ ERP failure rates.

## When to use this skill

Use this skill at the **start** of:
- Any new Odoo implementation
- Any major scope expansion (new entity, new module family, new country)
- A re-implementation or rescue of a stalled project
- Whenever someone asks "should we customize Odoo to do X?" — the answer process is in here

Use this **before** `odoo-schema-design`. The methodology decides who is making decisions and on what timeline; schema design happens within those decisions.

## The five non-negotiable principles

These are taken directly from OIM. Internalize them before reading anything else.

### 1. Standard before Studio before Custom

Try to solve the requirement with **standard Odoo** first. If standard can't, try **Odoo Studio** (the no-code customizer). If Studio can't, only then consider a **custom module**.

Rationale: every line of custom code is a line of upgrade pain forever. Each customization compounds — *the complexity of a project grows with the square of the number of customizations, not linearly.* A project with 20 small customizations is not 20x more complex than a project with 1; it is closer to 400x.

Action: in any conversation about a feature, the question "can we do this with standard Odoo?" must be asked and answered before "how do we build it?"

### 2. Replace, don't replicate

The customer's existing process exists because of the constraints of their old tool. New tool, new constraints, new process. **Do not faithfully replicate the old workflow in Odoo.** Look for the *intent* of the workflow and ask whether Odoo's standard approach satisfies that intent.

Rationale: replication produces Odoo instances shaped like the old system, which means the old system's pain is preserved and Odoo's value is wasted.

Action: when capturing a requirement, ask "what is this trying to accomplish?" and "if we built this from scratch today, would we still do it this way?"

### 3. SPoC, not committee

The customer designates a **Single Point of Contact (SPoC)**. The SPoC has decision-making authority. The Odoo Project Leader works with the SPoC, not with a committee.

Rationale: committees produce slow, compromise decisions that please nobody. A single empowered person produces fast, sometimes wrong, decisions that can be corrected. Speed beats correctness in iteration.

Action: refuse to start a kickoff until the SPoC is named and confirmed in writing.

### 4. Avoid importing data history

Import the **master data** (chart of accounts, products, partners, opening inventory, opening balances). Do **not** import historical transactions (closed sale orders, paid invoices, completed manufacturing orders) unless there is a specific operational or legal need.

Rationale: historical data import is expensive, error-prone, and rarely used after go-live. Most "we need our history" requests die in the cost-benefit analysis.

Exceptions: open AR/AP at cutover (must be imported), tax-relevant journal entries for the current fiscal year (sometimes), open sales orders / open POs / open MOs (yes — they're operationally needed).

Action: at kickoff, draw a line — only data on the right side of this line gets imported. Everything else stays in the legacy system as a read-only archive.

### 5. Customer satisfaction is not a useful KPI during implementation

Satisfaction naturally declines from the kickoff to the go-live as scope is challenged, requests are refused, and reality sets in. Disagreement is healthy. The KPI is **on time, on budget, in production.**

Rationale: implementations optimized for satisfaction at every step end up with bloated scope, happy users during configuration, and a failed go-live. Implementations optimized for delivery have grumpy stakeholders for three months and grateful ones for the next decade.

Action: track velocity, scope adherence, and budget consumption — not satisfaction surveys.

## The four phases

### Phase 1: GAP Analysis (typically 1–3 weeks)

**Purpose:** validate that Odoo can solve the customer's problem at acceptable cost; identify the gaps between standard Odoo and customer requirements; produce a written scope.

**Activities:**
- Workshops with key users from each functional area
- Map current processes (lightly — don't get sucked into months of process documentation)
- For each requirement, classify: standard / Studio / custom / drop / defer
- Build the GAP analysis document
- Identify the SPoC candidate
- Estimate effort and produce phased plan

**Gate to next phase:** signed scope document. No verbal scope. No "we'll figure it out as we go."

**Common failure:** trying to satisfy 100% of requirements. The OIM expectation is that **30% of requirements get dropped**, **20% defer to second deployment**, and roughly **50% land in Phase I**. If the gap analysis is closing at 95% requirement coverage, you are not gap-analyzing — you are scope-creeping.

### Phase 2: Project Kick-off (~1 week)

**Purpose:** align the team, confirm the SPoC, agree on the methodology, build buy-in.

**Activities:**
- SPoC training (point them at learn.odoo.com immediately, see `odoo-training-and-adoption`)
- Steering committee formation (if project size warrants; small projects skip this)
- Kickoff workshop with key users — this is the change-management moment, not a technical meeting
- Project plan finalized

**Gate to next phase:** SPoC is operational, plan is signed, team knows who decides what.

**Common failure:** treating kickoff as a kick-the-tires demo. The Odoo source material is explicit: at least 10% of total project time goes here, because the project's success depends on the kick-off.

### Phase 3: Implementation (the longest phase — weeks to a few months)

Iterative cycles. Each cycle: configure → import → validate → train. Demo to SPoC weekly. SPoC decides go/no-go on each module.

**Activities:**
- Configuration in a staging environment (never directly in production)
- Data import: master data first, opening balances last
- Specific developments only when GAP analysis approved them
- End-user training led by SPoC with Project Leader support
- UAT with **written test scripts**, not click-around

**Gate to next phase:** SPoC sign-off on each module, UAT pass.

**Common failures:**
- Skipping staging
- Letting scope grow because "while we're at it..."
- Accepting verbal sign-off
- Configuring in parallel with demos so the demo doesn't match what's deployed

### Phase 4: Go-Live (1–2 weeks, plus support tail)

**Purpose:** cut over to production; provide intense support during the early days.

**Activities:**
- Final master data import to production
- Opening balances import
- Cutover (typically over a weekend; resume operations Monday)
- Hyper-care: Project Leader and SPoC available in real time for the first 1–2 weeks
- Issue triage and rapid fix
- Post-go-live retrospective

**Gate to "done":** stable operations, customer team operating without daily hand-holding, retrospective conducted, documented learnings filed for the next phase or entity.

### Phase 5 (optional): Second Deployment

If the GAP analysis deferred 20% of requirements to a second phase, that work becomes Phase 5 and runs as a smaller version of phases 1–4. Most projects benefit from this — first deployment proves Odoo works, removes anxiety, and the customer becomes much more pragmatic about what is actually needed in Phase 5.

## Roles

### Customer-side

- **Sponsor** — typically CEO/CFO. Funds the project, sets strategic objectives, sometimes sits on steering committee.
- **SPoC (Single Point of Contact)** — the most important role. Internal Odoo expert in training; decision-maker for "what does the business need." Trains end-users post-go-live. Must be available and authorized.
- **Key users** — domain experts (warehouse manager, head of QA, AP clerk lead). Help SPoC define requirements, test, validate.
- **End users** — everyone else who will use Odoo daily.

### Implementer-side

- **Project Director** — senior PM, oversees multiple projects, sits on steering committee for large projects.
- **Project Leader** — the doer. PM + business analyst + product expert. Decides "how" Odoo implements the customer's "what." For small projects this is the only role.
- **App Expert** — pulled in for module-specific deep dives (e.g., a manufacturing specialist for MRP-heavy projects).
- **Developer** — only when custom development is approved. Writes the code per spec.

For small implementations a single internal hybrid (sponsor + project leader) is common. The SPoC is whichever client team member can be embedded in the project.

## How this skill orchestrates the others

```
Phase 1: GAP Analysis
└─► [no other skills yet — this is the strategic decision phase]

Phase 2: Kick-off
├─► odoo-training-and-adoption  (SPoC training kicks off here)
└─► odoo-schema-design           (begin SDR)

Phase 3: Implementation (iterative)
├─► odoo-schema-design           (finalize SDR)
├─► odoo-connect                 (working RPC client)
├─► odoo-data-migration          (master data load)
├─► odoo-api-sync                (ongoing integrations)
├─► odoo-webhooks                (event-driven flows)
├─► odoo-gmp-compliance          (regulated-industry overlay)
└─► odoo-training-and-adoption   (key-user training, end-user training)

Phase 4: Go-Live
├─► odoo-data-migration          (final cutover loads)
├─► odoo-training-and-adoption   (hyper-care + refresh sessions)
└─► odoo-api-sync                (production sync flows enabled)

Phase 5: Second Deployment
└─► [start over at Phase 1 for the deferred scope]
```

## Producing the GAP Analysis document

Every project starts with this single deliverable. The template is in `references/gap-analysis-template.md`. It is short by design — long GAP analyses indicate scope problems.

A complete GAP analysis has:
1. Business overview (one page max)
2. Functional area summary (one paragraph per area)
3. Requirements list with classification (standard / Studio / custom / drop / defer)
4. SPoC nomination
5. Phasing plan (what's in Phase I, what's deferred to Phase II)
6. Estimated effort and budget envelope
7. Risks and assumptions

## What this skill does not do

- Configure Odoo (that's downstream skills)
- Make schema decisions (that's `odoo-schema-design`)
- Train users (that's `odoo-training-and-adoption`)
- Decide which Odoo modules to install — although the GAP analysis surfaces this, the actual module selection is part of schema design

## References

- Odoo Implementation Methodology by Catherine Vieslet (Odoo S.A.) — see `research/research-bibliography.md`
- `references/gap-analysis-template.md`
- `references/oim-anti-patterns.md`
- `references/rescue-case-study.md` — what happens when you skip Phase 1 and try to recover; documents the retroactive backfill pattern
