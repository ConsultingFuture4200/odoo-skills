# OIM Anti-Patterns

Failure patterns observed across thousands of Odoo implementations. When you see one of these in flight, name it and stop. Each is named so the team has shared vocabulary.

## 1. The Comprehensive Replication

**Pattern:** customer says "we want Odoo to do everything our old system does, exactly the same way."

**Why it happens:** comfort. The old way is known. The new way is scary.

**Why it kills projects:** Odoo is a different shape from the old tool. Replicating the shape produces a Frankenstein — Odoo bent into the old tool's silhouette, losing its own value, gaining the old tool's pain.

**Counter:** "Replace, don't replicate." Ask "what is this trying to accomplish?" not "how do we make Odoo do this?"

---

## 2. The Committee

**Pattern:** instead of a single SPoC, the customer designates a committee of 4–8 people who must agree on every decision.

**Why it happens:** politically safer than naming one person who decides.

**Why it kills projects:** decision velocity drops by an order of magnitude. Every decision becomes a calendar negotiation. Three months in, the project has made fewer decisions than a SPoC-led project makes in three weeks.

**Counter:** refuse to start without a single named SPoC with explicit decision authority. The committee can become a steering committee that meets monthly to review progress, not approve decisions.

---

## 3. The Historical Hoard

**Pattern:** "we need 5 years of historical data in Odoo."

**Why it happens:** sunk cost fallacy; vague fear of losing access; sometimes a real (smaller) need is generalized.

**Why it kills projects:** historical data import takes a multiple of the time of master data import. It's full of dead references (vendors no longer used, employees who left, products discontinued). It rarely gets queried after go-live. The cost is paid up front in calendar time.

**Counter:** ask "in the last 6 months, how often did you actually look at data older than 1 year in your current system?" Usually the honest answer is "never" or "for one tax thing." Plan for that one tax thing — keep the legacy system read-only for the next 7 years, or export the relevant subset to a flat file.

---

## 4. The Customization Cascade

**Pattern:** every requirement that doesn't exactly match standard Odoo gets a custom development. After 18 months, the implementation has 40 custom modules and the next version upgrade is impossible.

**Why it happens:** path of least resistance for both customer and consultant. Customer gets "yes" answers. Consultant bills more hours.

**Why it kills projects:** complexity grows with the square of customizations. Maintenance cost grows. Upgrade cost grows. Eventually the customer is locked into a specific Odoo version forever.

**Counter:** rigorously enforce Standard > Studio > Custom. Every Custom needs a written justification approved by the SPoC and the Sponsor — not just the requesting key user.

---

## 5. The Specification Sponge

**Pattern:** GAP analysis runs for 4 months, produces a 200-page document, and still doesn't quite cover everything.

**Why it happens:** trying to eliminate ambiguity instead of accepting and managing it. Often driven by a customer who has been burned by a previous ERP project.

**Why it kills projects:** the longer the spec phase, the more reality drifts (people leave, business shifts, market changes). By the time the spec is signed, parts of it are obsolete.

**Counter:** time-box GAP analysis (1–3 weeks for SMEs, 4–6 for large accounts). Accept that 20% of details will be discovered in implementation; budget for that. The spec is a starting framework, not a contract.

---

## 6. The Demo-Driven Development

**Pattern:** the implementation team builds something just well enough to demo, then moves on. Three months in, none of the demos add up to a working system.

**Why it happens:** weekly demos create pressure to show *something*. Shortcuts compound.

**Why it kills projects:** at go-live, the system that worked in demos doesn't work in production because none of the demos were end-to-end.

**Counter:** demos should walk through real end-to-end flows, with real data, in the staging environment that will become production. UAT is run by SPoC + key users with written test scripts, not click-around.

---

## 7. The Big Bang Without Buy-In

**Pattern:** team spends 6 months in a back room building Odoo. Two weeks before go-live, end users see it for the first time. Resistance erupts.

**Why it happens:** consultants find it efficient to work in isolation. Customer SPoC doesn't push back because the demos look fine.

**Why it kills projects:** end users feel imposed-upon. Adoption fails. Even technically correct systems get rejected because the people using them weren't part of building them.

**Counter:** key users involved early and continuously. End-user previews scheduled into the implementation timeline, not stacked at the end. Change management starts at kickoff, not at go-live.

---

## 8. The Yes Reflex

**Pattern:** Project Leader says yes to every customer request, no matter how unreasonable.

**Why it happens:** wanting to please. Confusing satisfaction with success.

**Why it kills projects:** scope explodes. Budget blows up. Eventually the Project Leader has to start saying no, but now the customer expects yes — and is angry when it stops.

**Counter:** practice the "no" conversation early. The Odoo source material is explicit: "You're not there to say yes." The customer hires an Odoo professional to be told *how* to solve their problem, not to have every demand transcribed into a build order.

---

## 9. The Phantom SPoC

**Pattern:** a SPoC is named on paper but spends 5% of their week on the project because they have a day job.

**Why it happens:** SPoC role is volunteered or assigned without the day job being backfilled.

**Why it kills projects:** decisions queue. Project Leader spins. Implementation pace drops to whatever the SPoC's spare-time allows.

**Counter:** the SPoC charter must commit a real percentage of their work week (typically 30%+ during active phases) and the Sponsor must commit to backfilling their other duties. Without this, the project is structurally broken from day one.

---

## 10. The Standard-Avoidance Reflex

**Pattern:** team rejects standard Odoo because "it doesn't quite fit how we work" without examining whether their way of working is good or bad.

**Why it happens:** familiarity bias. The standard process feels "wrong" because it's not what they do.

**Why it kills projects:** every standard rejected is a customization added. See #4.

**Counter:** when standard is rejected, demand a written explanation of why the standard process is worse than the customer's existing process — *for the business*, not for the people who happen to be doing it today. Often the standard is actually better and the team just hasn't lived with it yet.

---

## How to use this list in conversation

When you see an anti-pattern, name it. "I think we're heading into a Customization Cascade — let me walk us through it." Naming creates shared awareness and lets the team stop the slide without anyone feeling individually called out.

The anti-patterns are documented as a system, not as someone's fault. That's intentional and matters.
