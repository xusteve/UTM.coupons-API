# UTM.coupons API

Public reference for the [UTM.coupons](https://utm.coupons/) REST API — manage
coupons and collection pages, read attribution reports, and receive signed
webhooks.

- **API reference:** https://utm.coupons/api/
- **Integration guide:** https://utm.coupons/docs/wp-integrations/
- **Dashboard:** https://app.utm.coupons
- **WordPress plugin:** https://github.com/xusteve/UTM.coupons-for-WP

## Base URL

```
https://api.utm.coupons
```

Every data route is mounted under `/api`. Two endpoints are public and need no
credentials:

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/health` | Liveness: `{ ok, service, env, time }` |
| `GET` | `/api/status` | Same payload, CORS-enabled so a browser can probe it |

## Authentication

All `/api/*` data routes require a bearer token:

```
Authorization: Bearer <token>
```

Two token kinds are accepted:

| Token | Format | Use |
| --- | --- | --- |
| API key | `sk_live_…` | Server-to-server integrations. Created in the dashboard under **API keys**. |
| Session token | Clerk JWT | Used by the dashboard itself. |

API keys carry a scope:

- `read` — GET/HEAD/OPTIONS only. A `read` key used on a write method returns
  **403** with `this API key is read-only; create a read_write key`.
- `read_write` — full access.

The workspace is derived from the credential, never claimed by the caller — you
cannot address another workspace by passing an id.

## Errors

Errors are JSON. A validation failure also returns the offending fields.

```json
{ "error": "invalid_payload", "issues": [ … ] }
```

| Status | `error` | Meaning |
| --- | --- | --- |
| 400 | `invalid_json`, `workspace_required` | Malformed body, or missing context |
| 401 | `unauthorized`, `invalid_token` | No token, or a token that does not verify |
| 403 | `forbidden` | Authenticated but not allowed (e.g. read-only key on a write) |
| 404 | `not_found` | No such route or object |
| 422 | `invalid_payload` | Body failed schema validation |
| 500 | `internal_error` | Server-side failure |

## Endpoints

### Session & workspace

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/api/me` | Current user, workspace and role |
| `GET` | `/api/workspace` | Workspace settings |
| `PATCH` | `/api/workspace` | Update workspace settings |
| `PUT` | `/api/workspace/logo` | Upload workspace logo |
| `DELETE` | `/api/workspace/logo` | Remove workspace logo |

### Coupons

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/api/coupons` | List (`status`, `cursor`, `limit`) → `{ data, nextCursor }` |
| `POST` | `/api/coupons` | Create a coupon |
| `GET` | `/api/coupons/:code` | Fetch one |
| `PATCH` | `/api/coupons/:code` | Update one |
| `DELETE` | `/api/coupons/:code` | Delete one |
| `GET` | `/api/coupons/:code/store-config` | Store-side setup card |

`POST /api/coupons` requires `code` and `title`. Everything else is optional and
defaults server-side: `type`, `variant`, `buttonStyle`, `themeVars`, `affiliate`,
`discountType`, `discountValue`, `rules`, `startsAt`, `expiresAt`,
`autoGeneratePage`, `shortLink`, `embedCode`, `storeSyncStatus`. If you set
`discountValue` you must also set `discountType` — a value with no type is
rejected rather than guessed.

```bash
curl -X POST https://api.utm.coupons/api/coupons \
  -H "Authorization: Bearer sk_live_…" \
  -H "Content-Type: application/json" \
  -d '{"code":"SUMMER20","title":"20% off summer","discountType":"percentage","discountValue":20}'
```

### Single-use code pools

| Method | Path | Purpose |
| --- | --- | --- |
| `POST` | `/api/coupons/:code/codes/generate` | Generate a batch of unique codes |
| `GET` | `/api/coupons/:code/codes` | List allocated codes |
| `POST` | `/api/coupons/:code/codes/import` | Import your own codes |
| `GET` | `/api/coupons/:code/codes/export` | Export allocated codes |
| `PATCH` | `/api/coupons/:code/codes/:codeValue` | Update one allocated code |

### Promoter reference codes

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` / `POST` | `/api/code-refs` | List / create a promoter reference |
| `GET` / `PATCH` / `DELETE` | `/api/code-refs/:slug` | Read, update, delete one |

### Reports

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/api/reports/overview` | Headline metrics |
| `GET` | `/api/reports/timeseries` | Metrics over time |
| `GET` | `/api/reports/conversions` | Conversion list |
| `GET` | `/api/reports/coupons/:code` | Per-coupon report |
| `GET` | `/api/reports/share/export` | Export share events |

### Collection pages (`/s/` builder)

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/api/pages` | List pages |
| `POST` | `/api/pages` | Create a page |
| `GET` | `/api/pages/:slug` | Fetch one |
| `PUT` | `/api/pages/:slug` | Replace one (coupon list included) |
| `PUT` | `/api/pages/:slug/logo` | Page-level logo |

### Webhooks

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/api/webhooks` | List endpoints |
| `POST` | `/api/webhooks` | Create an endpoint → **201** |
| `DELETE` | `/api/webhooks/:id` | Delete an endpoint |
| `POST` | `/api/webhooks/:id/test` | Send a `ping` through the real delivery path |

See [docs/webhooks.md](docs/webhooks.md) for the delivery contract.

### Other

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` / `POST` / `DELETE /:id` | `/api/keys` | Manage API keys |
| — | `/api/protection` | Blacklist / protection switches |
| — | `/api/rewards` | Reward ledger |
| — | `/api/domains` | Custom domains (Cloudflare for SaaS) |
| — | `/api/ab-tests` | A/B experiments |
| `GET` / `POST` | `/api/graphql` | Read-only GraphQL query surface |
| — | `/api/billing` | Checkout + billing webhook |
| — | `/api/push` | Web push subscriptions |
| — | `/api/integrations` | Integration waitlist |

`POST /api/coupons/:code/share` is a declared stub and returns **501**
`not_implemented`.

## Reporting conversions

Conversions and refunds do **not** go through this host. They are signed and
POSTed to the separate ingest endpoint:

```
POST https://hooks.utm.coupons/v1/conversions
x-utm-signature: <HMAC-SHA256 of the raw request body>
```

The signature is computed with your HMAC secret, so forged reports are rejected.
Accepted payloads carry the order id, amount, currency, coupon code and the
optional click id used for attribution. No customer PII is required or expected.

The WordPress plugin does this automatically; a manual integration can post to
the same endpoint directly.

## CORS

Browser calls are allow-listed per origin. An origin that is not on the list
receives no CORS header at all, so a page you do not control cannot read your
workspace data. Server-to-server calls are unaffected.

## License

MIT. See [LICENSE](LICENSE).
