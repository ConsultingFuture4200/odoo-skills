---
name: odoo-connect
description: Use this skill to set up authenticated, idempotent communication with an Odoo instance — Odoo Online, Odoo.sh, or self-hosted. Triggers include "connect to Odoo", "Odoo RPC", "Odoo XML-RPC", "Odoo JSON-RPC", "Odoo MCP", "Odoo API key", "talk to Odoo from Python", or "how do I read/write Odoo records from outside the UI". Produces a working client and the patterns (auth, batching, field selection, retries, External-ID-based upsert) that downstream skills rely on. Do NOT use for full data migrations (use odoo-data-migration), event-driven sync (use odoo-api-sync), or webhook setup (use odoo-webhooks).
---

# Odoo Connect

The plumbing layer underneath every other skill in this family. Three transport
choices with different trade-offs; pick once, use everywhere. The patterns
below are grounded in a real catalog migration that used the Odoo MCP
transport against an Odoo Online instance.

## When to use this skill

- Bootstrapping a new project that needs to read/write Odoo from outside the UI
- Choosing between MCP, JSON-RPC, and XML-RPC for a specific use case
- Hardening an existing client (auth, retries, error handling)
- Reproducing a known-good migration pattern on a second entity

Do **not** use this skill for the actual data movement — that's
`odoo-data-migration`. This skill produces the client and the conventions; the
client is then used by every other skill.

## The three transports

| Transport | When to use | Trade-offs |
|---|---|---|
| **MCP (Model Context Protocol)** | Claude Code / Claude Desktop sessions, agentic workflows | Fastest path inside an LLM session; rich tool surface; one connection per session; not suitable for unattended cron jobs |
| **JSON-RPC** (`/web/dataset/call_kw`, `/jsonrpc`) | Programmatic clients, services, cron jobs | Modern Odoo default; works against all deployments; cleaner error shapes than XML-RPC |
| **XML-RPC** (`/xmlrpc/2/object`, `/xmlrpc/2/common`) | Legacy clients, language SDKs that haven't moved on | Universal compatibility; verbose; slightly slower |

Choose **MCP** for any work happening inside a Claude session — it gives you
`find_skill`, smart field selection, and idempotent helpers (`import_records`)
without writing a client.

Choose **JSON-RPC** for everything else. XML-RPC is only justified when the
language ecosystem you're in lacks a JSON-RPC client.

## Authentication

| Deployment | Recommended | Notes |
|---|---|---|
| Odoo Online | API key (Settings → Users → Developer Mode → API Keys) | Username + DB name still required alongside |
| Odoo.sh | API key, scoped per branch if possible | Staging vs production keys must be distinct |
| Self-hosted | API key | Falls back to username/password but **never** in production |

API keys are per-user and inherit the user's access rights. Create a
service-account user (e.g. `claude-code@<entity>`) with explicit access groups
rather than reusing a human's key. For MCP-driven sessions, the proxy holds
the API key out-of-band.

**Never** commit an API key to source control. Use environment variables or
the platform's secret manager.

## Connection check

Before doing real work, verify three things:

```
1. server_info()
   → returns version, git_commit, api_version, odoo_url, connected
   → if `connected: false`, stop and fix auth before continuing

2. list_models()
   → returns models the connection is authorized for
   → if your target model is missing, the user lacks group membership

3. read one known record (e.g. res.partner id 1)
   → smoke test that read access works
```

A typical MCP `server_info` response:

```
{"version":"1.5.0","odoo_url":"https://example-co.odoo.com",
 "connected":true,"runtime_id":"<runtime>",
 "companies":[{"id":1,"name":"Example Co"}]}
```

The `companies` field tells you about multi-company context. A connection
authenticated against company 1 will not see company-2 records by default.

## Smart field selection

Odoo records can have hundreds of fields. Reading them all serializes large
binary fields and can crash on `ir.attachment` references. Three modes:

| Call shape | Returns | When |
|---|---|---|
| `get_record(model, id)` | Smart default selection (~15 common fields) | Most reads — quick exploration |
| `get_record(model, id, fields=[...])` | Only the named fields | Production code — be explicit |
| `get_record(model, id, fields=["__all__"])` | Every field | Field discovery only — never in a loop |

For field discovery, prefer `read("odoo://<model>/fields")` over `__all__` —
it returns metadata without serializing values.

**Rule:** in any code path that runs more than once or against more than one
record, name your fields explicitly. Smart defaults are for humans exploring;
they are not a contract.

## Batch sizing

Empirically validated batch sizes:

| Operation | Batch size | Notes |
|---|---|---|
| `update_records` (single field, e.g. `tracking='lot'`) | 100 | ~5 seconds at this size |
| `create_records` (full product creates) | 50 | Larger payloads → smaller batches |
| `import_records` (with External IDs) | 100 | Idempotent; safe to re-run |
| `search_records` (read) | 100 default, 250 max | Pagination via `offset` |
| `update_record` (single record) | n/a | Acceptable for ad-hoc fixes; never for bulk |

Larger batches sometimes succeed but timeouts on Odoo Online become more
likely past 200 records per call. Smaller batches are also more debuggable
when one record fails.

## External-ID-based upsert (idempotency)

The `import_records` call in Odoo's RPC accepts an `id` column containing an
External ID. Behavior:

| External ID resolves to | Behavior |
|---|---|
| Existing `ir.model.data` row → existing record | UPDATE |
| Existing `ir.model.data` row → record was deleted | Re-CREATE with same External ID |
| No matching `ir.model.data` row | CREATE + register new External ID |

This is the foundation of idempotent migration. As long as your External ID
generation is deterministic for a given source row, re-running the import
produces zero duplicates. Convention: `<namespace>.<entity>_<stable_business_key>`,
e.g. `__import__.ing_ascorbic_acid_vendora` for ingredients or
`__exampleco__.product_template_brandx_vitaminc_60ct` for ongoing automation.

For models that don't go through `import_records` directly (e.g. one-off
writes via `create_record`), set the External ID after creation:

```
create the record → capture the returned id
create an ir.model.data row pointing at (model, res_id) with the desired external_id
```

## Error patterns + retries

Common error shapes and how to react:

| Error | Cause | Action |
|---|---|---|
| `AccessError` | Service account lacks group membership | Add group; do not retry |
| `ValidationError` (single message) | Domain rule rejected | Fix payload; do not retry |
| `UserError: You cannot change a tracking mode while there are existing quants` | Lot-flip on a non-zero product | Run the lot-tracking flip runbook (see `odoo-data-migration`) |
| `MissingError: Record does not exist or has been deleted` | Foreign-key target was removed/archived | Refresh the lookup; check `active=False` records |
| `ConcurrentUpdateError` | Optimistic locking conflict | Retry with backoff (250ms, 1s, 4s); if it persists, the record is hot — serialize that path |
| Timeout (504, network reset) | Odoo Online cold start, large batch, or slow query | Reduce batch size; retry once; if it keeps timing out, the operation is too large for one RPC call |

**Retry rule:** retry only on transient errors (timeouts, 5xx, concurrent
update). Never retry on validation errors — they will fail the same way.

## Multi-company contexts

When the database has multiple companies, almost every read/write needs an
explicit company context. RPC pattern:

```
context = {"allowed_company_ids": [<id1>, <id2>], "company_id": <primary>}
```

For a single-company DB the context can be omitted. When a sister entity is
added to the database, every call must set `allowed_company_ids` explicitly
to avoid silent cross-company reads.

## Deployment-specific notes

### Odoo Online

- Database name is in the URL: `https://<entity>.odoo.com` → DB name `<entity>`
- API endpoint: `https://<entity>.odoo.com/jsonrpc` or `/xmlrpc/2/object`
- Database snapshots via UI; no `pg_dump` access
- Cold-start latency on the first call after idle (~2–5s); plan for it
- Custom modules cannot be installed — Studio + standard apps only
- Version upgrades happen on Odoo's schedule; pin tests to a known version

### Odoo.sh

- Per-branch databases; staging and production are siblings
- Build pipeline runs on push; you get a fresh DB each promote
- Custom modules deploy via git; no UI install
- `pg_dump` available via the .sh shell

### Self-hosted

- Full PostgreSQL access
- You own the upgrade cadence — both a feature and a footgun
- Backup/restore is your problem; test it

## Hand-off

Once `server_info()` returns `connected: true` and the smoke-test read
succeeds, hand off to:

- `odoo-data-migration` — bulk loads with the SDR
- `odoo-api-sync` — ongoing sync with external systems
- `odoo-webhooks` — event-driven flows
- `odoo-gmp-compliance` — Quality module config

## References

- `references/mcp-client-patterns.md` — MCP usage patterns
  (find_skill, smart field selection, batch helpers)
- Odoo official RPC docs: <https://www.odoo.com/documentation/18.0/developer/reference/external_api.html>
