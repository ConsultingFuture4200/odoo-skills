# SPoC Charter Template

The Single Point of Contact is the most important role in any Odoo implementation. This charter exists to make the role concrete, so it doesn't degrade into a "phantom SPoC" — named on paper, absent in practice.

Save as `spoc-charter-{entity-slug}-v{N}.md`.

---

# SPoC Charter — {ENTITY}

**Effective date:** YYYY-MM-DD
**Project phase covered:** Kickoff through Go-Live + 3 months hyper-care
**Reviewed by:** Sponsor
**Acknowledged by:** SPoC

## SPoC identity

| Field | Value |
|---|---|
| Name | |
| Title | |
| Department | |
| Reports to | |
| Backup SPoC | (must be named — projects can't run on one person who might get sick) |

## Time commitment

| Phase | % of work week | Hours per week (40-hr basis) |
|---|---|---|
| GAP Analysis | 25% | 10 |
| Kick-off | 50% | 20 |
| Implementation | 30–50% | 12–20 |
| Go-Live + hyper-care | 75% (week 1) → 30% (month 1) → 20% (steady state) | varies |

Sponsor commits to backfilling the SPoC's normal duties at the percentages above. This is **not optional** — it is the structural condition that makes the SPoC role work.

## Decision authority

The SPoC has decision-making authority on:

- Functional requirements interpretation (what does the business actually need)
- Process design within Odoo (how a workflow is structured)
- Module configuration choices (which Odoo features to use)
- Training scheduling and delivery
- Acceptance of completed implementation work for their functional areas

The SPoC must escalate to the Sponsor for:

- Scope changes that affect budget or timeline by more than 10%
- Decisions to commission custom development beyond the GAP analysis approval
- Decisions to abandon or replace standard Odoo features in favor of legacy processes
- Decisions affecting compliance posture (for regulated entities: 21 CFR Part 111 / Part 117 implications)

## Responsibilities

The SPoC is expected to:

1. **Become the internal Odoo expert.** Complete the SPoC training pathway in `odoo-training-and-adoption`. Use Odoo's own eLearning. Optionally pursue the Functional Certification.
2. **Own the GAP analysis** in collaboration with the Project Leader. Sign the scope document.
3. **Run the daily decision flow** during implementation — questions from the implementation team get a same-day or next-day answer.
4. **Conduct or coordinate end-user training.** No one trains end users better than the colleague who will work next to them after go-live.
5. **Provide first-level support** post-go-live. Project Leader is escalation, not first responder.
6. **Maintain the internal Odoo wiki** — SOPs, quick-reference cards, screen recordings.
7. **Participate in change management** — visible advocacy on the floor, not just in meetings.

## Authority and ground rules

The SPoC is a peer with the Project Leader, not a subordinate. The Project Leader does not bypass the SPoC to take direction directly from end users or department heads.

The SPoC works on the project, not just oversees it. Hands-on configuration in staging, hands-on data import, hands-on training delivery.

The SPoC will sometimes need to say "no" to colleagues — to scope creep, to last-minute requirements, to "just one small change" requests. The Sponsor visibly backs the SPoC when this happens.

## Sign-off

By signing below, the parties acknowledge their commitments under this charter.

| Role | Name | Signature | Date |
|---|---|---|---|
| Sponsor | | | |
| SPoC | | | |
| Project Leader | | | |
| Backup SPoC | | | |

This charter is reviewed at the end of each project phase and updated if circumstances have changed materially.
