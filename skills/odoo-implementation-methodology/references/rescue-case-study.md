# Rescue / Retroactive Backfill Case Study

A real-world variant of the OIM where the implementation work happens first
and the methodology artifacts are authored after. Useful as a worked example
of "what does it look like when you skip Phase 1 and try to recover."

The pattern below is anonymized from a live engagement: a contract
manufacturer of dietary supplements (anonymized as "ExampleCo"), GMP-regulated
under 21 CFR Part 111, with ~200 product templates and a multi-customer
catalog that needed an urgent lot-tracking remediation.

## What happened

| Step | OIM ideal | What ExampleCo actually did |
|---|---|---|
| Phase 1 — GAP Analysis | Workshops, requirements classification, signed scope | Skipped — work began with an ingredient-catalog import from a Google Sheet |
| SPoC nomination | Named in writing before kickoff | Not done at the time of the work |
| Phase 2 — Kick-off | All-hands kickoff workshop, SPoC training begins | Skipped |
| Phase 3 — Implementation | Iterative configure → import → validate → train cycles | Done — a 7-phase catalog migration shipped: 188 products migrated to lot tracking, 44 created, 147 vendor pricelist rows |
| Phase 4 — Go-Live | Cutover with hyper-care | Not yet — mid-Phase-3 still |
| SOP catalog | Authored during workflow buildout | Not yet — flagged for backfill |

The build was real and the data was good. What was missing was the
methodology artifacts that future-you (or a new SPoC, or an auditor) will need.

## What it costs to skip and backfill vs to do upfront

| Cost | Doing upfront | Backfilling |
|---|---|---|
| Time to GAP analysis | 1–3 weeks (workshops are part of discovery) | 1 week (most info is now in your head; workshops collapse into 1–2 sessions) |
| Time to SDR | 2–5 days | 2 days (decisions exist in execution; you're transcribing) |
| Cost of wrong scope decisions | Caught early, cheap to fix | Already shipped; expensive or impossible to undo |
| Cost of missing SPoC | Recognized at kickoff; can recruit | Recognized post-go-live; team has no internal expert |
| Stakeholder confidence | Built incrementally through workshops | Built post-hoc, often with skepticism ("why are we doing this now?") |

**Backfilling is cheaper than not having the artifact, but more expensive
than doing it the right time. The choice "skip and backfill" should be a
deliberate cost-benefit decision, not a default.**

## When skipping is justified

The work had urgent operational value: a regulated production line with
products lacking lot tracking. Waiting 4–6 weeks for a formal GAP analysis
would have meant 4–6 more weeks of non-compliant operation. The "fix the
urgent regulatory gap first, then backfill the methodology" trade-off was
reasonable in this specific case.

It would NOT have been reasonable for:
- A multi-company implementation (the schema mistakes would compound across entities)
- A custom-module-heavy project (no GAP means no scope to gate against)
- A team-wide rollout (no SPoC means no internal expert to support end users)

## Backfill order of operations

When recovering after the fact:

```
Step 1: Audit what shipped
  - Run odoo-schema-design's audit checklist against the live instance
  - Document the 7 framework decisions that are de facto answered (the
    schema-design worked-examples reference is the template)

Step 2: Author the SDR retroactively
  - Use sdr-template.md
  - Each section: "Decision: <what was actually built>" + rationale +
    "Remediation: <what to change>" if the executed answer is wrong

Step 3: Author the GAP analysis retroactively
  - Use gap-analysis-template.md
  - Requirements list reflects what's already built (label as "DONE") plus
    what's still in scope
  - Phasing plan: Phase I = what shipped, Phase II = what's queued

Step 4: Name the SPoC NOW
  - Even retroactively, name a SPoC. The project cannot enter Phase 2
    without one, and you're already past Phase 2.
  - The SPoC immediately starts the training pathway (40–80h / 4–8 weeks)

Step 5: Identify the SOP gaps
  - Every workflow already built that doesn't have a written SOP is a
    GMP gap (for regulated entities) or a knowledge-transfer gap
    (for everyone else)
  - Author SOPs in priority order: regulated workflows first, then
    high-frequency operational workflows

Step 6: Re-enter the OIM at the appropriate phase
  - For an ExampleCo-style situation: late Phase 3. Iterate normally
    from here.
```

## Anti-pattern: "we'll backfill later"

The most common variant of skipping the methodology is also the most
dangerous: telling yourself you'll write the GAP / SDR / SPoC charter
"once things settle down." Things don't settle down — new work piles on,
the executed decisions calcify, and the artifacts never get written.

The forcing function that worked: tying the methodology backfill to a
visible work tracker (Linear, Jira, GitHub Issues) with explicit issues
per artifact. "Write the GAP analysis" / "Write the SDR" / "Name the SPoC"
become blocking dependencies for downstream workstreams. Without that, the
backfill stays indefinitely "next."

## Reusable rescue checklist

For any "Odoo project that started without methodology":

- [ ] Audit what's in the live instance (schema-design audit checklist)
- [ ] Author SDR retroactively, decision by decision
- [ ] Author GAP analysis retroactively, requirements as DONE / IN-PROGRESS / DEFERRED
- [ ] Name SPoC in writing today, even if work is mid-flight
- [ ] Open tickets per missing artifact in your tracker — the visible queue is the forcing function
- [ ] Identify SOP gaps for any built workflow
- [ ] Re-enter OIM at the appropriate phase (usually late Phase 3 or Phase 4)
- [ ] Run a retrospective: was the skip justified? What would you have caught earlier with the methodology?
- [ ] Codify "retro-only OIM" as a documented option for future urgent regulatory gaps — but not as a default
