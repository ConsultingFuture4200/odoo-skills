# Odoo MCP Client Patterns

A real catalog migration was driven entirely through the Odoo MCP proxy.
This file collects the patterns that worked.

## Connection profile

```
Server:   <runtime ID — changes per session>
URL:      https://<entity>.odoo.com
DB:       <entity> (derived from subdomain)
Version:  Odoo 18 (server reports 1.5.0 for the MCP layer)
Company:  id=1, "<Entity>"
Auth:     API key held server-side; not exposed to the agent
```

## Tool surface

| Tool | Use when | Notes |
|---|---|---|
| `server_info` | Connection check | First call of every session |
| `list_models` | Discover what's available | Returns models + allowed operations |
| `find_skill` | Orient before a domain workflow | Keyword-matches skills, returns full SKILL.md |
| `get_skill` | Direct fetch by name | Use when find_skill missed |
| `search_records` | Read with a domain filter | Default limit 100, max 250 — paginate via offset |
| `get_record` | Read one record | Smart field selection by default |
| `create_record` | Create one | Captures returned id |
| `create_records` | Create many | Batch sizes up to ~50 for full payloads |
| `update_record` | Update one field on one record | Ad-hoc fixes |
| `update_records` | Update one field on many records | Single-field batch — fast |
| `import_records` | Idempotent upsert via External IDs | The bread-and-butter migration tool |
| `delete_record` / `delete_records` | Hard delete | Prefer archive (`active=False`) — deletes break FKs |
| `set_binary_field` | Upload an attachment / image | Avoid for large blobs in loops |
| `post_message` | Add a chatter note | Provenance for any data change |
| `list_resource_templates` | Discover Odoo MCP resources | `odoo://<model>/fields` for field metadata |

## Field discovery without serialization risk

Reading `__all__` on a record with image fields, html fields, or large
binaries can crash the response serializer. Use the resource API instead:

```
read("odoo://res.partner/fields")
```

Returns field metadata (type, relation, required, store) without serializing
any record data. Use this to plan an explicit fields=[...] list, then call
`get_record` or `search_records` with that list.

## The find_skill / get_skill workflow

At the start of any domain workflow:

```
1. find_skill(question="how do I receive a customer-owned material in our Odoo")
   → returns the most relevant skill's full content + alternatives list

2. If no match: get_skill(name="<best guess>")

3. Read the returned SKILL.md before executing
```

This is the LLM-equivalent of "RTFM before you act." Skills returned by
`find_skill` are authored to be the operational reference.

## Idempotent upsert pattern

```
import_records('product.template', [
  {
    'id': '__exampleco__.product_template_brandx_vitaminc_60ct',
    'name': 'Brand X - Vitamin C - 60ct',
    'default_code': '[BX-VC-60]',
    'categ_id/id': 'product.product_category_all',
    'uom_id/id': 'uom.product_uom_unit',
    'tracking': 'lot',
    'is_storable': True,
  },
  ...
])
```

Notes:
- `id` column is the External ID (Odoo expects `module.name` shape; for ad-hoc
  imports, `__import__` or a custom namespace like `__<entity>__` works)
- `<field>/id` columns reference other records by External ID (cleaner than
  numeric IDs)
- Re-running the same import is safe — Odoo matches on External ID and updates

## Batch sizing in practice

Numbers from a real Phase 1–7 migration:

| Operation | Records per batch | Wall-clock |
|---|---|---|
| Phase 3 re-categorize (single field) | 100 | ~3s |
| Phase 5 product creates (full payload) | 50 | ~12s |
| Phase 6 supplierinfo (full payload) | 50 | ~10s |
| Phase 7 GREEN flip (single field) | ~95 | ~5s |
| Phase 7 YELLOW per-product (4 calls) | 1 | ~1.5s |

When in doubt, smaller batches. A 200-record batch that fails halfway is much
harder to debug than two 100-record batches where you know which one failed.

## Multi-step workflows that aren't single calls

Some Odoo state changes need orchestration. Examples:

| Workflow | Steps |
|---|---|
| Cancel a stock.picking with non-`done` moves | 1) update_record picking state='cancel'; 2) search children stock.moves; 3) update_records moves state='cancel' |
| Create a product with an immediate vendor pricelist | 1) import_records product.template; 2) capture id; 3) create_records product.supplierinfo with partner_id + product_tmpl_id |
| Migrate a product to lot tracking with on-hand stock | See `odoo-data-migration/references/lot-tracking-flip-runbook.md` |

The MCP doesn't expose every Odoo workflow method; for those that aren't
exposed, decompose into the underlying state writes.

## Provenance via chatter

For any data change you want auditable, post a chatter message:

```
post_message(
  model='product.template',
  res_id=<id>,
  body='Cost basis updated to $0.044053/g per VendorA quote (sample-only — '
       'flag for substitution at next bulk PO). Source: 04_cost_basis.csv '
       'row 47, vendor VendorA 100g sample @ $4.41.'
)
```

Phase 4 of a real migration used this pattern on 9 products to record
vendor-quote provenance even where the price field was unchanged. The
chatter becomes the audit trail Odoo's standard logging doesn't capture.

## What MCP doesn't give you (and what to do instead)

| Need | MCP gap | Workaround |
|---|---|---|
| Unattended cron job | MCP needs an interactive session | Use direct JSON-RPC client |
| Webhook receipt | MCP is request-response | Use Odoo's `base_automation` to push outbound |
| Long-running report generation | MCP timeouts on slow queries | Write the report logic into a `quality.spreadsheet` or use direct SQL on self-hosted |
| File downloads (attachments at scale) | `set_binary_field` is upload-only | Use the `/web/content/<attachment_id>` HTTP endpoint with the API key |
