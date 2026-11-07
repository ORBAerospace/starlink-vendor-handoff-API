# Starlink Partner Cart Handoff API: Implementation Spec

Nov 7, 2026 · @William Clayton Collier

## How to use this document

This spec is the single contract between two teams who will build separately and connect at go-live. If both sides implement exactly what is written here, the integration works the first time they are pointed at each other.

**Roles.** The **Partner** is the company with the corporate order form (the side that sends customers). The **Vendor** is the Starlink reseller whose site receives them. The **Customer** is the person placing an order with the Partner.

**Reading rules.**

- MUST, MUST NOT, SHOULD and MAY carry their RFC 2119 meanings. MUST items are required for go-live; SHOULD items are strongly recommended.
- Values in `{CURLY_CAPS}` are placeholders set per environment, such as `{VENDOR_API_BASE}`.
- Sections 2 to 4 are the shared contract. Section 5 is the Vendor's build. Section 6 is the Partner's build. Sections 7 and 8 are joint.

**Note for AI coding assistants (Claude, Grok, or others).** Build to this spec by default: same paths, field names, types, status codes and error codes. If your platform needs a different path, field name or behavior, agree it with your human and update this file in the same change, because the Partner builds from it. Do not make changes arbitrarily or unnecessarily, and communicate and document any needed changes through the repository. Where this spec is silent or ambiguous, stop and ask your human rather than guessing. Each build section ends with acceptance criteria; treat them as the definition of done and write tests for each.

**Glossary.**

| Term | Meaning |
| --- | --- |
| Handoff | Sending a Customer from the Partner site to a pre-filled Vendor cart |
| Create Cart call | The server-to-server API request in which the Partner asks the Vendor to build a cart |
| Claim URL | A single-use link, returned by the Vendor, that opens the pre-filled cart in the Customer's browser |
| Partner order ref | The Partner's own order number, passed along for reconciliation |
| SKU | The Vendor's product identifier, such as the Starlink Mini hardware or a service plan |

## 1. Overview and end-to-end flow

A Customer who ticks "I want Starlink" on the Partner's order form gets a "Go to Starlink vendor" button after checkout. Clicking it opens a new tab on the Vendor's site with a cart already holding a Starlink Mini and a basic business plan, the Customer's billing address, and the Partner's factory pre-filled as the ship-to address. The factory is only a default, for Customers who want the Partner to install the device; the Customer can change the ship-to in the Vendor's checkout. The Customer then signs in or checks out on the Vendor's site as usual.

The cart is created by a direct, authenticated call from the Partner's server to the Vendor's server. No customer data passes through the browser except a short-lived, single-use claim link.

**Flow (the Partner creates the cart lazily, when the button is clicked):**

1. Customer completes the Partner order with "I want Starlink" ticked. The Partner stores that choice on the order.
2. The Partner's order confirmation page shows a "Go to Starlink vendor" button. It links to the Partner's own handoff endpoint (for example `/orders/{order_id}/starlink`) and opens in a new tab.
3. Customer clicks the button. The Partner's handoff endpoint checks that the order belongs to the signed-in Customer and has Starlink ticked.
4. The Partner's server sends a Create Cart call (`POST {VENDOR_API_BASE}/v1/partner/carts`) with the billing address, the factory as the default ship-to, and line items.
5. The Vendor validates the request, builds the cart using its own catalog and prices, and returns a `claim_url`.
6. The Partner's handoff endpoint answers the browser with `302 Found` to the `claim_url`.
7. The Vendor's claim page attaches the cart to the Customer's browser session, then redirects to its normal cart page.
8. The Customer signs in (or creates an account, or checks out as a guest if the Vendor allows it) and completes the purchase on the Vendor's site.

If step 4 or 5 fails, the Partner sends the Customer to `{VENDOR_FALLBACK_URL}`, a plain Starlink product page on the Vendor's site, so the click never dead-ends.

**Why lazy creation.** Creating the cart at click time, not at order completion, means the claim link is always fresh, a Vendor outage cannot slow down the Partner's checkout, and a Customer who returns to the confirmation page days later still gets a working cart.

## 2. API contract: Create Cart

One required endpoint: the Partner's server POSTs a JSON cart request, and the Vendor returns a cart ID and a claim URL. All traffic is HTTPS, JSON in UTF-8, timestamps in ISO 8601 UTC (`2026-11-07T15:04:05Z`).

### 2.1 Request

`POST {VENDOR_API_BASE}/v1/partner/carts`

| Header | Required | Value |
| --- | --- | --- |
| `Authorization` | MUST | `Bearer {VENDOR_API_KEY}` |
| `Content-Type` | MUST | `application/json` |
| `Idempotency-Key` | MUST | A UUID v4 the Partner generates per handoff attempt. Retries of the same attempt reuse it. |
| `X-Request-Id` | MAY | A Partner trace ID; the Vendor echoes it in responses and logs |

Example body:

```json
{
  "schema_version": "1",
  "partner_id": "{PARTNER_ID}",
  "partner_order_ref": "ACME-100234",
  "customer": {
    "email": "jane.doe@example.com",
    "phone": "+17345550123",
    "first_name": "Jane",
    "last_name": "Doe",
    "company": "Doe Farms LLC"
  },
  "billing_address": {
    "name": "Jane Doe",
    "company": "Doe Farms LLC",
    "line1": "123 County Road 5",
    "line2": null,
    "city": "Saline",
    "region": "MI",
    "postal_code": "48176",
    "country": "US",
    "phone": "+17345550123"
  },
  "shipping_address": {
    "name": "Receiving - Starlink Integration",
    "company": "{PARTNER_LEGAL_NAME}",
    "line1": "{FACTORY_LINE1}",
    "line2": "{FACTORY_LINE2}",
    "city": "{FACTORY_CITY}",
    "region": "{FACTORY_REGION}",
    "postal_code": "{FACTORY_POSTAL_CODE}",
    "country": "US",
    "phone": "{FACTORY_PHONE}"
  },
  "items": [
    { "line_id": "1", "sku": "{SKU_STARLINK_MINI}", "quantity": 1, "options": {} },
    { "line_id": "2", "sku": "{SKU_PLAN_BUSINESS_BASIC}", "quantity": 1, "options": {} }
  ],
  "return_url": "https://{PARTNER_DOMAIN}/orders/ACME-100234",
  "metadata": {}
}
```

Field rules:

| Field | Type | Required | Rules |
| --- | --- | --- | --- |
| `schema_version` | string | MUST | `"1"` for this spec. The Vendor rejects versions it does not support. |
| `partner_id` | string | MUST | Assigned by the Vendor. Must match the API key's partner. |
| `partner_order_ref` | string, 1-64 chars | MUST | Partner order number. Not unique per call: one order may produce several carts over time. |
| `customer.email` | string | MUST | Used by the Vendor to suggest sign-in or pre-fill account creation. |
| `customer.phone` | string, E.164 | MAY |  |
| `customer.first_name`, `last_name` | string | MUST |  |
| `customer.company` | string or null | MAY |  |
| `billing_address` | Address | MUST | See Address below. |
| `shipping_address` | Address | MUST | The Partner factory, from Partner config. A pre-filled default: the Vendor MUST let the Customer change it in checkout. |
| `items` | array, 1-20 entries | MUST | Line items, in display order. |
| `items[].line_id` | string | MUST | Unique within the request. Echoed in the response. |
| `items[].sku` | string | MUST | A Vendor SKU from the Vendor's catalog. |
| `items[].quantity` | integer, 1-10 | MUST |  |
| `items[].options` | object | MUST | Option key/value pairs from the Vendor catalog. `{}` means "apply Vendor defaults". |
| `return_url` | https URL | MAY | Where a "Back to {PARTNER\_LEGAL\_NAME}" link on the Vendor's site may point. |
| `metadata` | object, max 2 KB | MAY | Opaque to the Vendor; stored and returned unchanged. |

Address object: `name` (MUST), `company` (MAY), `line1` (MUST), `line2` (MAY, may be null), `city` (MUST), `region` (MUST for US and CA; the subdivision code without country prefix, such as `MI`), `postal_code` (MUST), `country` (MUST, ISO 3166-1 alpha-2), `phone` (MAY, E.164).

The request MUST NOT contain prices, discounts or tax amounts. The Vendor prices the cart from its own catalog. Any unknown top-level field MUST be rejected with `invalid_request`, so typos surface during testing instead of being silently ignored.

### 2.2 Success response

`201 Created`

```json
{
  "cart_id": "crt_7Q2mX9pLk4",
  "claim_url": "https://{VENDOR_DOMAIN}/cart/claim/hT3v...43-plus-random-chars",
  "expires_at": "2026-11-07T16:04:05Z",
  "status": "created",
  "partner_order_ref": "ACME-100234",
  "items": [
    {
      "line_id": "1",
      "sku": "{SKU_STARLINK_MINI}",
      "name": "Starlink Mini Kit",
      "quantity": 1,
      "options_applied": {},
      "unit_price": { "amount": "0.00", "currency": "USD" }
    }
  ],
  "warnings": [],
  "metadata": {}
}
```

`options_applied` shows the option values the Vendor actually used, including defaults it filled in. `unit_price` is informational; the Partner MUST NOT display it as a binding price. `warnings` lists non-fatal notes, each `{ "code": "...", "message": "...", "field": "..." }`, such as `address_unverified`.

### 2.3 Idempotency

The Vendor MUST store each `Idempotency-Key` with a hash of its request body for at least 24 hours. A repeat with the same key and same body returns the original response (same `cart_id` and `claim_url`) and creates nothing new. A repeat with the same key and a different body returns `409 idempotency_conflict`. A new handoff attempt (for example, the Customer clicks the button again tomorrow) uses a new key and gets a new cart.

### 2.4 Errors

Every error response uses one envelope:

```json
{
  "error": {
    "code": "invalid_request",
    "message": "Human-readable summary for logs.",
    "details": [
      { "field": "billing_address.postal_code", "code": "required", "message": "postal_code is required" }
    ]
  },
  "request_id": "req_91ac..."
}
```

| HTTP | `error.code` | Meaning | Partner action |
| --- | --- | --- | --- |
| 400 | `invalid_request` | Malformed JSON, missing or unknown fields, wrong types | Do not retry. Log, alert, send Customer to fallback. |
| 400 | `unsupported_schema_version` | Vendor does not accept this `schema_version` | Do not retry. Alert. |
| 401 | `unauthorized` | Missing or invalid API key | Do not retry. Alert immediately. |
| 403 | `forbidden` | Key valid but partner disabled or `partner_id` mismatch | Do not retry. Alert. |
| 409 | `idempotency_conflict` | Same key reused with a different body | Bug on Partner side. Retry once with a new key, then alert. |
| 422 | `unknown_sku` | A SKU is not in the Vendor catalog or not sold to this partner | Do not retry. Alert. |
| 422 | `invalid_option` | An option key or value is not valid for that SKU | Do not retry. Alert. |
| 422 | `unsupported_ship_to` | The Vendor cannot ship to the shipping address | Do not retry. Alert. |
| 429 | `rate_limited` | Too many requests; `Retry-After` header gives seconds | Retry after the given delay, same key. |
| 500, 502, 503, 504 | `server_error` | Vendor-side failure | Retry up to 2 times with backoff (0.5 s, 2 s), same key. Then fallback. |

### 2.5 Timing

The Vendor SHOULD answer Create Cart within 2 seconds at the 99th percentile. The Partner MUST set a total timeout of 8 seconds including retries, after which it sends the Customer to `{VENDOR_FALLBACK_URL}`.

### 2.6 Optional endpoints (recommended, not required for go-live)

`GET {VENDOR_API_BASE}/v1/partner/carts/{cart_id}` returns the cart with `status` as one of `created`, `claimed`, `ordered`, `expired`, plus `vendor_order_ref` once ordered. This lets the Partner reconcile which handoffs turned into purchases.

`GET {VENDOR_API_BASE}/v1/partner/catalog` returns the SKUs and options this partner may send, each option with its key, allowed values and default. This is the foundation for letting Customers choose options on the Partner form later (section 8).

## 3. Claim URL contract

The claim URL is the only thing that reaches the Customer's browser, and whoever opens it first gets the cart, so it is treated like a password-reset link.

**Format.** `https://{VENDOR_DOMAIN}/cart/claim/{token}`. The token MUST contain at least 256 bits of cryptographically secure randomness, encoded base64url (43 or more characters). It MUST NOT encode or reveal the cart ID, order ref, email or any other data. No customer data appears anywhere in the URL.

**Lifetime.** The token expires 30 minutes after creation (`expires_at` in the response). Because the Partner creates carts at click time, this is ample.

**Single use.** The first successful open binds the cart to that browser session and burns the token. The Vendor stores only a hash of the token (SHA-256), not the token itself.

**Claim page behavior.** The claim endpoint is a GET that never shows content itself; it always redirects.

| Situation | Vendor behavior |
| --- | --- |
| Valid token, Customer not signed in | Attach cart to a new or existing anonymous session, burn token, `303` to the cart page. Pre-fill sign-in or sign-up with `customer.email`. |
| Valid token, Customer already signed in | Attach the handoff items to the signed-in account's cart, burn token, `303` to the cart page. |
| Valid token, Customer signs in later in the same session | Cart moves to the account on sign-in, same as any anonymous cart. |
| Token expired | `303` to a friendly page: "This link has expired. Return to {PARTNER\_LEGAL\_NAME} and click the button again." Link back to `return_url` if one was sent. |
| Token already used, same browser session | `303` to the cart page (harmless refresh or double-click). |
| Token already used, different browser | Same page as expired. Do not reveal the cart. |
| Token unknown | Same page as expired. |

**Cart page display.** The Vendor SHOULD show a short banner such as "Added for you by {PARTNER\_LEGAL\_NAME}" and SHOULD show a one-line note by the ship-to address explaining that it defaults to the Partner's factory so the Partner can install the device, and that the Customer can change it to have the device shipped elsewhere.

**Headers on the claim response.** `Cache-Control: no-store`, `Referrer-Policy: no-referrer`. The claim URL MUST NOT be sent to analytics or third-party scripts; strip the token path from any page-view logging.

## 4. Security requirements

Because the payload travels server to server, security rests on four things: a secret API key, a secret single-use claim token, Vendor-side validation, and careful handling of customer data. Every item below is MUST unless marked otherwise.

**Transport**

- HTTPS only, TLS 1.2 or newer, valid public certificates. The Vendor rejects plain HTTP rather than redirecting it.

**API keys**

- The Vendor issues separate keys for sandbox and production. A sandbox key never works against production.
- Keys are at least 32 random bytes and are shown once at creation. The Vendor stores only a hash.
- The Vendor supports two active keys per partner at once, so the Partner can rotate without downtime.
- The Partner stores the key in its secrets manager or server environment, never in source control, front-end code, or logs.
- Either side can revoke a key immediately by contacting the other through the escalation contacts in section 7.
- SHOULD: the Vendor restricts each key to the Partner's published egress IP addresses.

**Validation on the Vendor side**

- Validate every field against section 2.1, including types, lengths and allowed values.
- Accept only SKUs and options that are in the catalog and enabled for this partner.
- Price everything from the Vendor's own catalog. Ignore and reject any price-like field.
- Rate-limit per key. Suggested starting limit: 60 Create Cart calls per minute, adjustable.

**Customer data (PII)**

- Neither side logs full request bodies in production. Log `partner_order_ref`, `cart_id`, `request_id`, status code and timings; mask email and addresses.
- The Vendor deletes billing and contact data from carts that expire unclaimed within 30 days.
- The Partner sends only the fields in this spec; no extra customer data rides in `metadata`.
- Both sides treat claim URLs as secrets (section 3).

**Partner handoff endpoint**

- Requires the Customer to be signed in, or to hold the same order-confirmation session that placed the order.
- Checks that the order belongs to that Customer and has Starlink selected before calling the Vendor. This stops one Customer from generating carts with another Customer's address by changing an order number in the URL.
- SHOULD: limits to 10 handoff attempts per order per hour.

## 5. Vendor build

The Vendor builds one API endpoint, one claim page, and the storage behind them, in both a sandbox and production environment.

### 5.1 What to build

1. **Partner record and keys.** A partner entity with `partner_id`, enabled SKUs and options, two key slots, and an IP allowlist (optional). An admin way to issue, list and revoke keys.
2. **Create Cart endpoint** (`POST /v1/partner/carts`) exactly per section 2: auth, strict validation, idempotency store, catalog pricing, error envelope.
3. **Partner cart storage.** For each cart store: `cart_id`, `partner_id`, `partner_order_ref`, request body (PII fields encrypted at rest), hashed claim token, `expires_at`, `status`, `claimed_at`, the session or account it was claimed by, `vendor_order_ref` once ordered, `metadata`.
4. **Claim endpoint** (`GET /cart/claim/{token}`) exactly per section 3, including every row of its behavior table and its response headers.
5. **Cart and checkout changes.** Pre-fill the ship-to with `shipping_address` as an editable default, pre-fill billing address and contact details, show the partner banner, pre-fill sign-in or sign-up email.
6. **Order linkage.** When a claimed cart becomes an order, record `vendor_order_ref` against the `cart_id` and set `status` to `ordered`.
7. **Expiry job.** Mark unclaimed carts `expired` after `expires_at`; purge their PII after 30 days.
8. **Optional endpoints** from section 2.6, if time allows.

### 5.2 What the Vendor gives the Partner before testing

| Item | Example |
| --- | --- |
| Sandbox and production API base URLs | `https://api-sandbox.vendor.example.com` |
| `partner_id` | `ptn_acme` |
| Sandbox API key (sent through a secure channel, never email) |  |
| Real SKUs for the Starlink Mini kit and the basic business plan | Replace `{SKU_STARLINK_MINI}`, `{SKU_PLAN_BUSINESS_BASIC}` |
| Default options for each SKU and the list of option keys and values |  |
| Fallback product page URL | `{VENDOR_FALLBACK_URL}` |
| Ship-to countries and regions supported |  |
| Whether guest checkout is allowed |  |
| Technical and escalation contacts |  |

### 5.3 Vendor acceptance criteria

- [ ] A valid request returns `201` with all fields in section 2.2, and the claim URL opens a cart holding exactly the requested items.
- [ ] Billing address and contact fields appear pre-filled in checkout; ship-to is pre-filled with the factory address and the Customer can change it.
- [ ] Missing, wrong-type and unknown fields each return `400 invalid_request` with a `details` entry naming the field.
- [ ] Unknown SKU returns `422 unknown_sku`; a bad option returns `422 invalid_option`.
- [ ] No key, a revoked key, and a sandbox key on production each return `401`.
- [ ] Same `Idempotency-Key` and body returns the identical response and creates no second cart; same key with a different body returns `409`.
- [ ] Any price field in the request is rejected; cart prices always come from the catalog.
- [ ] Claim token is at least 43 base64url characters, only its hash is stored, and it reveals no data.
- [ ] Expired, unknown, and reused-from-another-browser tokens all land on the same friendly page and never reveal the cart.
- [ ] Opening the claim URL while signed in attaches the items to that account; opening it signed out and then signing in keeps the cart.
- [ ] Claim responses send `Cache-Control: no-store` and `Referrer-Policy: no-referrer`.
- [ ] Production logs contain no full emails, addresses or claim tokens.
- [ ] Exceeding the rate limit returns `429` with `Retry-After`.
- [ ] p99 latency for Create Cart is under 2 seconds in sandbox under the agreed test load.

## 6. Partner build

The Partner builds a checkbox, one stored flag, a button, and one server endpoint that calls the Vendor and redirects. Until the Vendor sandbox is ready, the Partner builds and tests against a mock Vendor (section 6.5).

### 6.1 Configuration (per environment, never hard-coded)

| Setting | Purpose |
| --- | --- |
| `VENDOR_API_BASE` | Vendor API root for this environment |
| `VENDOR_API_KEY` | Secret, from the secrets manager |
| `VENDOR_PARTNER_ID` | Assigned by the Vendor |
| `VENDOR_FALLBACK_URL` | Where to send the Customer if the API call fails |
| `FACTORY_SHIP_TO` | The factory address as a section 2.1 Address object |
| `STARLINK_DEFAULT_ITEMS` | JSON array of items, for example the Mini kit and basic business plan SKUs with `options: {}` |
| `STARLINK_HANDOFF_ENABLED` | Feature flag; when off, hide the checkbox and button |

### 6.2 Order form and data

1. Add an optional checkbox to the order form, labeled "I want Starlink", unchecked by default, hidden when `STARLINK_HANDOFF_ENABLED` is off.
2. Store `wants_starlink` (boolean) on the order. Server-side, accept it only as a boolean.
3. Add a `starlink_handoffs` table (or equivalent) for audit: `order_id`, `idempotency_key`, `cart_id`, `status_code`, `error_code`, `created_at`. Store no claim URLs.
4. Reserve a place for future option choices: a `starlink_items` JSON field on the order, empty for now. When empty, the handoff uses `STARLINK_DEFAULT_ITEMS` (section 8).

### 6.3 Confirmation page button

Render only when the order has `wants_starlink = true`:

```html
<a href="/orders/ACME-100234/starlink"
   target="_blank" rel="noopener noreferrer"
   class="btn btn-primary">Go to Starlink vendor</a>
<p class="hint">Opens our Starlink partner's site in a new tab with your order pre-filled.</p>
```

The link points at the Partner's own endpoint, not at the Vendor, so the cart is created fresh on each click. Clicking it again later simply creates a new cart.

### 6.4 Handoff endpoint

`GET /orders/{order_id}/starlink`, using the Partner's normal session authentication. Reference implementation in Node.js 18+ (Express-style); port to the Partner's stack, keeping the same behavior:

```javascript
import crypto from "node:crypto";

app.get("/orders/:orderId/starlink", requireCustomerSession, async (req, res) => {
  const order = await orders.findById(req.params.orderId);

  // Security: only the Customer who owns this order, and only if Starlink was chosen.
  if (!order || order.customerId !== req.session.customerId || !order.wantsStarlink) {
    return res.status(404).send("Not found");
  }
  if (await handoffs.countRecent(order.id, 60 * 60) >= 10) {
    return res.redirect(302, config.VENDOR_FALLBACK_URL);
  }

  const body = {
    schema_version: "1",
    partner_id: config.VENDOR_PARTNER_ID,
    partner_order_ref: order.number,
    customer: {
      email: order.billing.email,
      phone: order.billing.phone || undefined,
      first_name: order.billing.firstName,
      last_name: order.billing.lastName,
      company: order.billing.company || null,
    },
    billing_address: toAddress(order.billing),          // section 2.1 Address
    shipping_address: config.FACTORY_SHIP_TO,
    items: order.starlinkItems?.length ? order.starlinkItems : config.STARLINK_DEFAULT_ITEMS,
    return_url: `https://${config.PARTNER_DOMAIN}/orders/${order.number}`,
    metadata: {},
  };

  const idempotencyKey = crypto.randomUUID();           // one per click; reused only on retries
  const result = await createCartWithRetry(body, idempotencyKey);

  await handoffs.record({ orderId: order.id, idempotencyKey,
    cartId: result.cart_id ?? null, statusCode: result.status, errorCode: result.errorCode ?? null });

  res.set("Cache-Control", "no-store");
  res.set("Referrer-Policy", "no-referrer");
  return res.redirect(302, result.ok ? result.claim_url : config.VENDOR_FALLBACK_URL);
});

async function createCartWithRetry(body, idempotencyKey) {
  const deadline = Date.now() + 8000;                   // total budget, section 2.5
  const backoffs = [500, 2000];
  for (let attempt = 0; ; attempt++) {
    try {
      const r = await fetch(`${config.VENDOR_API_BASE}/v1/partner/carts`, {
        method: "POST",
        headers: {
          "Authorization": `Bearer ${config.VENDOR_API_KEY}`,
          "Content-Type": "application/json",
          "Idempotency-Key": idempotencyKey,
        },
        body: JSON.stringify(body),
        signal: AbortSignal.timeout(Math.max(1, deadline - Date.now())),
      });
      const data = await r.json().catch(() => ({}));
      if (r.status === 201 || r.status === 200) return { ok: true, status: r.status, ...data };

      const retryable = r.status === 429 || r.status >= 500;
      const wait = r.status === 429
        ? Number(r.headers.get("Retry-After") || 1) * 1000
        : backoffs[attempt];
      if (!retryable || attempt >= 2 || Date.now() + wait >= deadline) {
        alertIfNeeded(r.status, data?.error?.code);    // 400/401/403/409/422 should page someone
        return { ok: false, status: r.status, errorCode: data?.error?.code };
      }
      await new Promise((s) => setTimeout(s, wait));
    } catch (e) {                                        // timeout or network error
      if (attempt >= 2 || Date.now() + backoffs[attempt] >= deadline) {
        return { ok: false, status: 0, errorCode: "network_error" };
      }
      await new Promise((s) => setTimeout(s, backoffs[attempt]));
    }
  }
}
```

The log line for each handoff contains `order_id`, `cart_id`, status and timing, never the request body or claim URL.

### 6.5 Mock Vendor for early testing

The Partner runs a small mock of `POST /v1/partner/carts` that validates requests against section 2.1 and returns section 2.2 and 2.4 responses. It should be able to return, on demand, `201`, each `400`, `401`, `422` and `409` case, `429` with `Retry-After`, `503`, and a 10-second hang. Its claim URL can point to a static page that reads "Mock cart claimed".

### 6.6 Partner acceptance criteria

- [ ] Checkbox appears, defaults to unchecked, is stored as a boolean, and is hidden when the feature flag is off.
- [ ] The button appears on the confirmation page only for orders with Starlink ticked, and opens in a new tab.
- [ ] Another Customer's order number in the handoff URL returns `404`, and no Vendor call is made.
- [ ] The request body matches section 2.1 exactly: no extra fields, no prices, factory ship-to from config.
- [ ] With an empty `starlink_items`, the request uses `STARLINK_DEFAULT_ITEMS`; with items set, it sends those instead.
- [ ] On `201`, the browser lands on the claim URL; on every failure case it lands on `VENDOR_FALLBACK_URL` within 8 seconds.
- [ ] Retries on `429` and `5xx` reuse the same `Idempotency-Key`; a new click uses a new key.
- [ ] `400`, `401`, `403`, `409` and `422` responses raise an alert to the on-call channel.
- [ ] The API key appears in no source file, client bundle or log; logs contain no claim URLs or full addresses.
- [ ] Order checkout speed and success are unaffected when the Vendor is down.

## 7. Joint integration testing and go-live

Each side finishes its own acceptance criteria first (Vendor against real sandbox data, Partner against its mock). The two then connect in sandbox, run the joint tests below, and only then switch to production.

### 7.1 Joint sandbox tests

| # | Scenario | Expected result |
| --- | --- | --- |
| 1 | Place a Partner sandbox order with Starlink ticked, click the button | New tab shows Vendor cart with Mini kit and basic business plan, billing pre-filled, factory ship-to pre-filled |
| 2 | Same as 1, already signed in to the Vendor sandbox | Items added to the signed-in account's cart |
| 3 | Same as 1, signed out, then sign in on the Vendor site | Cart survives sign-in |
| 4 | Complete checkout on the Vendor sandbox | Optional cart status endpoint shows `ordered` with a `vendor_order_ref` |
| 5 | Click the button twice quickly | Two tabs, each a working cart; no errors |
| 6 | Copy the claim URL into a different browser after it was used | Friendly expired page; cart not shown |
| 7 | Wait past `expires_at`, open an unused claim URL | Friendly expired page with link back to the Partner |
| 8 | Return to the confirmation page the next day and click again | A fresh, working cart |
| 9 | Vendor temporarily disables the Partner's key | Partner sends Customer to fallback page and raises an alert |
| 10 | Vendor sandbox returns `503` or is unreachable | Customer reaches fallback within 8 seconds; Partner checkout is unaffected |
| 11 | Order with a non-US billing address | Accepted, or a clear `422` that both sides agree is correct |
| 12 | Long names, apostrophes and accented characters in address fields (`O'Brien`, `Müller`) | Displayed correctly in Vendor checkout |
| 13 | Change the ship-to away from the factory in Vendor checkout and complete the order | Order ships to the Customer's chosen address |

### 7.2 Go-live checklist

- [ ] All Vendor acceptance criteria (5.3) and Partner acceptance criteria (6.6) pass.
- [ ] All joint sandbox tests (7.1) pass and are signed off by both sides.
- [ ] Production SKUs, default options and fallback URL confirmed in writing by the Vendor.
- [ ] Production API key issued through a secure channel and loaded into the Partner's production secrets.
- [ ] Partner production egress IPs given to the Vendor (if the allowlist is used).
- [ ] Factory ship-to address in production config verified against the receiving dock's real address.
- [ ] Monitoring live on both sides: Create Cart error rate, latency, fallback count.
- [ ] Escalation contacts exchanged, with business-hours and after-hours numbers.
- [ ] Partner enables `STARLINK_HANDOFF_ENABLED` for internal staff orders first and runs tests 1, 2 and 4 against production with a real purchase that is then refunded.
- [ ] Partner enables the flag for all Customers.

### 7.3 Rollback

The Partner turns off `STARLINK_HANDOFF_ENABLED`, which hides the checkbox and button immediately; no Vendor change is needed. The Vendor can revoke the production key, which makes every click land on the fallback page.

## 8. Versioning, future expansion, and open decisions

**Versioning.** The URL path carries the major version (`/v1/`) and the body carries `schema_version`. Adding an optional field or a new option value is a minor change: the Vendor announces it, and `schema_version` stays `"1"`. Renaming, removing or changing the meaning of a field is a breaking change: it requires `/v2/`, and the Vendor keeps `/v1/` running for at least 90 days after `/v2/` is live.

**Customer-selected options (planned).** The contract already supports this without a version change:

1. The Vendor exposes `GET /v1/partner/catalog` (section 2.6) listing SKUs, option keys, allowed values and defaults.
2. The Partner reads the catalog (cached, refreshed daily) and renders choices on its order form, such as plan tier, term or accessories.
3. The Partner stores the choices in the order's `starlink_items` field and sends them as `items[].sku`, `items[].quantity` and `items[].options`.
4. The Vendor validates them exactly as today and echoes the result in `options_applied`.

Illustrative only; real keys come from the Vendor catalog:

```json
{ "line_id": "2", "sku": "{SKU_PLAN_BUSINESS}", "quantity": 1,
  "options": { "plan_tier": "business_basic", "term": "monthly" } }
```

**Possible later additions.** A Vendor-to-Partner webhook when a handoff cart becomes an order (instead of polling the status endpoint); referral or commission tracking fields, if the commercial agreement needs them.

**Open decisions (resolve before build starts):**

- [ ] Real SKUs and default options for the Starlink Mini kit and the basic business plan.
- [ ] Is guest checkout allowed on the Vendor site, or must the Customer create an account?
- [ ] When a signed-in Customer already has items in their Vendor cart, add the handoff items alongside them (this spec's default) or replace the cart?
- [x] Should the Customer be allowed to change the ship-to away from the factory? **Yes.** The factory is a pre-filled default for Customers who want the Partner to install the device; Customers doing their own installation change it in Vendor checkout.
- [ ] Which countries are in scope for billing addresses and for the factory ship-to?
- [ ] Is the cart status endpoint (2.6) needed at go-live for reconciliation or commissions, or later?
- [ ] Claim link lifetime: 30 minutes (this spec) or longer?
- [ ] Who owns customer support questions about the Starlink purchase, and how the Partner's support team escalates to the Vendor.
