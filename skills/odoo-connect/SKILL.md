---
name: odoo-connect
description: Use this skill to set up authenticated, idempotent communication with an Odoo instance — Odoo Online, Odoo.sh, or self-hosted. Triggers include "connect to Odoo", "Odoo RPC", "Odoo XML-RPC", "Odoo JSON-RPC", "Odoo MCP", "Odoo API key", "talk to Odoo from Python", or "how do I read/write Odoo records from outside the UI". Produces a working client and the patterns (auth, batching, field selection, retries, External-ID-based upsert) that downstream skills rely on. Do NOT use for full data migrations (use odoo-data-migration), event-driven sync (use odoo-api-sync), or webhook setup (use odoo-webhooks).
---

# Odoo Connect

> **STATUS v0.3.0:** Released against Odoo 19 Online. Adds a working JSON-RPC client reference implementation (proven on two production services), a three-mode auth pattern for Google-service-account–style integrations, and Odoo 19 transport quirks.

The plumbing layer underneath every other skill in this family. Three transport
choices with different trade-offs; pick once, use everywhere. The patterns
below are grounded in a real catalog migration that used the Odoo MCP
transport against an Odoo Online instance, plus two production Python services
(`heron-bom-autobuild`, `heron-drive-ingest`) running JSON-RPC against the same.

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

## Reference JSON-RPC client (Python)

A minimum-viable Odoo JSON-RPC client. ~120 lines, no dependencies beyond
`requests`. Tested against Odoo 18 and 19 Online. Both
`heron-bom-autobuild` and `heron-drive-ingest` use this exact shape.

```python
import os
from dataclasses import dataclass
from typing import Any


class OdooRPCError(RuntimeError):
    """Raised when the Odoo server returns an error envelope."""


@dataclass
class OdooClient:
    url: str
    db: str
    username: str
    api_key: str
    timeout: float = 60.0
    _uid: int | None = None

    @classmethod
    def from_env(cls) -> "OdooClient":
        url = os.environ.get("ODOO_URL", "").rstrip("/")
        db = os.environ.get("ODOO_DB", "")
        username = os.environ.get("ODOO_USER", "")
        api_key = os.environ.get("ODOO_API_KEY", "")
        missing = [k for k, v in {
            "ODOO_URL": url, "ODOO_DB": db,
            "ODOO_USER": username, "ODOO_API_KEY": api_key,
        }.items() if not v]
        if missing:
            raise RuntimeError(f"OdooClient.from_env: missing env vars: {', '.join(missing)}")
        return cls(url=url, db=db, username=username, api_key=api_key)

    def _post(self, payload: dict) -> Any:
        import requests
        resp = requests.post(f"{self.url}/jsonrpc", json=payload, timeout=self.timeout)
        resp.raise_for_status()
        body = resp.json()
        if body.get("error"):
            raise OdooRPCError(f"Odoo RPC error: {body['error']}")
        return body.get("result")

    def _authenticate(self) -> int:
        if self._uid is not None:
            return self._uid
        result = self._post({
            "jsonrpc": "2.0", "method": "call",
            "params": {"service": "common", "method": "authenticate",
                       "args": [self.db, self.username, self.api_key, {}]},
        })
        if not result:
            raise OdooRPCError("authentication failed: check user + api_key")
        self._uid = int(result)
        return self._uid

    def execute_kw(self, model: str, method: str, args: list, kwargs: dict | None = None) -> Any:
        uid = self._authenticate()
        return self._post({
            "jsonrpc": "2.0", "method": "call",
            "params": {"service": "object", "method": "execute_kw",
                       "args": [self.db, uid, self.api_key, model, method, args, kwargs or {}]},
        })

    def search_read(self, model: str, domain: list, fields: list[str] | None = None,
                    limit: int = 0, offset: int = 0, order: str | None = None) -> list[dict]:
        kwargs: dict[str, Any] = {"limit": limit, "offset": offset}
        if fields is not None: kwargs["fields"] = fields
        if order is not None: kwargs["order"] = order
        return self.execute_kw(model, "search_read", [domain], kwargs)

    def create(self, model: str, vals: dict | list[dict]) -> int | list[int]:
        # Odoo 19 normalizes create() to return a list even for single-dict input.
        # Older versions returned a bare int; handle both.
        if isinstance(vals, dict):
            result = self.execute_kw(model, "create", [vals])
            return int(result[0] if isinstance(result, list) else result)
        return [int(x) for x in self.execute_kw(model, "create", [vals])]

    def message_post(self, model: str, record_id: int, body: str,
                     subtype_xmlid: str = "mail.mt_note") -> int:
        result = self.execute_kw(
            model, "message_post", [record_id],
            {"body": body, "subtype_xmlid": subtype_xmlid},
        )
        return int(result[0] if isinstance(result, list) else result)
```

### Notes

- Single instance is single-connection. For high-throughput services,
  pool 4-8 clients. Authentication is cached per-instance, so reuse.
- `from_env` is the standard factory. `.env` files via `python-dotenv` or
  shell sourcing both work; the client just reads `os.environ`.
- For multi-company contexts, add `context={"allowed_company_ids": [...]}`
  to the `execute_kw` kwargs. Default is the user's main company.

## Multi-mode auth pattern (for Google-style integrations)

When the same service needs to talk to BOTH Odoo (via API key) AND a Google
API (via service account / ADC / OAuth refresh token), keep the auth modules
parallel. Pattern from `heron-drive-ingest`:

```python
def get_credentials(scopes: list[str]):
    """Pick auth source by env var. SA > ADC > user OAuth refresh."""
    sa_path = os.environ.get("DRIVE_SA_PATH", "")
    if sa_path and Path(sa_path).exists():
        from google.oauth2.service_account import Credentials
        return Credentials.from_service_account_file(sa_path, scopes=scopes)

    if os.environ.get("USE_ADC", "").lower() in {"1", "true", "yes"}:
        import google.auth
        creds, _ = google.auth.default(scopes=scopes)
        return creds

    if os.environ.get("USE_RCLONE_CONFIG", "").lower() in {"1", "true", "yes"}:
        return _credentials_from_rclone_config()  # user OAuth refresh, stopgap

    raise RuntimeError("No auth method configured.")
```

The fallback hierarchy matters when Google Workspace org policies block
service-account key creation (`iam.disableServiceAccountKeyCreation`). The
production target is SA; ADC works when the OAuth client advertises the
right scopes; user-OAuth refresh from `rclone` config is a development
stopgap that ties to one user account. Document the path-to-production in
the project's SDR.

## Odoo 19 transport quirks

| Quirk | Surface | Workaround |
|---|---|---|
| `create()` returns a list for single-dict input | `TypeError: int() argument` | `int(result[0] if isinstance(result, list) else result)` |
| `message_post()` returns a list | Same | Same |
| `res.groups.users` renamed to `user_ids` | `Invalid field 'users'` | Use `user_ids` on 19; both work via fallback `vals.get('user_ids', vals.get('users'))` |
| `ir.filters.user_id` renamed to `user_ids` | `Invalid field 'user_id'` | Same pattern |
| Studio model ACL cache lag | 403 after creating `ir.model.access` | Wait for process reload; not a code fix |

Full list in `odoo-data-migration`'s gotcha table.

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
- Odoo official RPC docs: <https://www.odoo.com/documentation/19.0/developer/reference/external_api.html>

## Revision history

| Version | Date | Changes |
|---|---|---|
| v0.3.0 | 2026-05-11 | Add reference JSON-RPC client (Python) — 120-line OdooClient proven on two production services. Add multi-mode auth pattern for Google-style integrations (SA > ADC > user OAuth fallback). Add Odoo 19 transport quirks section. Bump RPC docs link to 19.0. |
| v0.2.1 | 2025-late | Three-transport overview (MCP/JSON-RPC/XML-RPC), external-ID upsert, batch sizing, error patterns. |
