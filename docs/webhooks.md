# Webhooks

UTM.coupons can notify your endpoint when something happens in your workspace.
Endpoints are managed through the REST API; delivery is signed so you can trust
the payload.

## Managing endpoints

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/api/webhooks` | List endpoints |
| `POST` | `/api/webhooks` | Create an endpoint → **201** |
| `DELETE` | `/api/webhooks/:id` | Delete an endpoint |
| `POST` | `/api/webhooks/:id/test` | Send a `ping` through the real delivery path |

Create an endpoint by posting the URL and the events you want:

```bash
curl -X POST https://api.utm.coupons/api/webhooks \
  -H "Authorization: Bearer sk_live_…" \
  -H "Content-Type: application/json" \
  -d '{"url":"https://example.com/hooks/utm","events":["conversion.created"]}'
```

The response includes a signing secret with a `whsec_` prefix. **It is shown
once, at creation.** Store it — you cannot read it back, and you need it to
verify deliveries.

## Delivery contract

Every delivery is a POST carrying three headers:

| Header | Value |
| --- | --- |
| `x-utm-event` | The event name (for example `ping`) |
| `x-utm-signature` | HMAC-SHA256 of the **raw request body**, keyed with the endpoint's `whsec_…` secret |
| `x-utm-delivery-id` | Identifier for this delivery attempt |

## Verifying a delivery

Recompute the HMAC over the raw body — not a re-serialised JSON object, which can
reorder keys and change the digest — and compare in constant time.

```js
import { createHmac, timingSafeEqual } from 'node:crypto';

function verify(rawBody, signature, secret) {
  const expected = createHmac('sha256', secret).update(rawBody).digest('hex');
  const a = Buffer.from(expected, 'utf8');
  const b = Buffer.from(signature, 'utf8');
  return a.length === b.length && timingSafeEqual(a, b);
}
```

Reject the request unless the signature matches. Return a 2xx quickly; a slow or
failing endpoint delays retries.

## Testing

`POST /api/webhooks/:id/test` sends a `ping` event with a tiny payload through
exactly the same sign-and-record path as a real event. So a verified `ping`
proves the endpoint works, rather than merely that the URL was saved.
