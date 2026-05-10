---
name: odoo-api-sync
description: Use this skill for ongoing programmatic reads and writes to Odoo via XML-RPC or JSON-RPC. Triggers include "sync to Odoo", "push X to Odoo", "pull X from Odoo", "Odoo upsert", "create Odoo record from {Shopify/Amazon/spreadsheet}", "Odoo batch update", "scheduled Odoo sync", or any non-bulk-import write operation. Implements idempotent upsert via external IDs, retry-with-backoff, batch chunking for large writes, and rate-limit-aware patterns. Should be invoked AFTER odoo-connect (for the client) and AFTER odoo-schema-design (for the external ID convention).
---

# Odoo API Sync — STUB

> **STUB:** Skeleton committed for family completeness. Flesh out before first usage. The patterns below are sketched — `odoo-connect` already covers the lower-level transport details.

## Scope

- Idempotent upsert pattern: `xmlid_to_res_id` then `write` or `create`, never blind `create`
- Batch operations: `create` array form, `write` with multiple ids, chunking for >1000 records
- Retry strategy: exponential backoff on `ConnectionError`, no retry on validation errors
- Read patterns: `search_read` with field whitelisting (don't fetch what you won't use)
- Write conflict handling (last-writer-wins vs optimistic concurrency via `__last_update`)
- Common entity patterns: partner upsert, product upsert, sale order creation, manufacturing order creation, stock move creation
- Logging convention: structured logs with external ID, operation, latency, outcome

## Outputs

- Reusable Python module with sync primitives
- Per-entity sync recipes (e.g., "sync Shopify orders to Odoo sale orders")
- Sync run report: created/updated/skipped/failed counts with sample failures

## TODO before flesh-out

- Library choice (same decision as `odoo-connect`)
- Settle on logging stack
- Decide where sync state lives (Odoo's own log? external Postgres? both?)
- A real first integration to anchor the patterns against (e.g., a Shopify-to-Odoo or Amazon-to-Odoo flow)
