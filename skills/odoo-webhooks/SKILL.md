---
name: odoo-webhooks
description: Use this skill for event-driven integration with Odoo — outbound webhooks fired by Odoo events, inbound webhook handlers that write to Odoo, or automated actions configured via base_automation. Triggers include "Odoo webhook", "trigger when {event} in Odoo", "Odoo automated action", "real-time Odoo sync", "Odoo to Slack/Zapier/Make.com", "notify when Odoo {event}". Distinct from odoo-api-sync, which is for pull-based or scheduled sync; this skill is for push/event-driven flows. Should be invoked AFTER odoo-connect.
---

# Odoo Webhooks — STUB

> **STUB:** Skeleton committed for family completeness. Flesh out before first usage. No real event-driven flow has been wired up yet to anchor patterns against.

## Scope

- `base_automation` (Odoo's native automation engine) for in-Odoo triggers
- Outbound webhook patterns: HTTP POST on record event, with retry + dead-letter
- Inbound webhook handlers: lightweight FastAPI/Flask service that validates and forwards to Odoo via api-sync
- Event taxonomy: which Odoo events are worth wiring up (state changes on sale orders, MO completion, stock moves, lot creation)
- Idempotency for webhook receivers (request-id deduplication)
- Security: signature verification, IP allowlisting

## Outputs

- `base_automation` configurations as XML or programmatic creation scripts
- Webhook receiver template (FastAPI service, Dockerfile, deployment notes)
- Event catalog: "when X happens in Odoo, what fires"

## TODO before flesh-out

- Decide whether webhook receiver is part of this skill or a separate service the skill just integrates with
- Pick payload format (Odoo native, CloudEvents, custom)
- A real first event-driven flow to anchor patterns against
