# Guide for the Vendor's developers

Thanks for building this. This page tells you where to start, what you can change freely, and what we need from you at each stage. The full contract is in [`SPEC.md`](SPEC.md).

## What to read

- **Read closely:** sections 1 to 5 of `SPEC.md` (flow, API contract, claim URL, security, Vendor build) and section 7 (joint testing).
- **Skim:** section 6 is the Partner's own build. It shows what we'll send and how we'll handle your responses, but you don't build any of it.
- **Answer:** the open decisions at the end of section 8.

## You lead; we adapt

You're building first, and we'll fit our side to what you build. The spec is our best guess at a clean contract. It isn't meant to override how your platform already works. Some examples of changes we expect and are happy with:

- A different path, auth scheme or header name that matches your existing APIs
- Different field names, if your system already has them (for example `zip` instead of `postal_code`)
- A different claim-link lifetime or token format, provided it is still single-use and unguessable
- Adding to a signed-in customer's existing cart, or replacing it, whichever your cart supports
- Dropping or deferring any SHOULD or MAY item

**The only rule is to write the change down.** Update `SPEC.md` in a pull request, or open an issue if you'd like to talk it over first. Our team and our coding agent build directly from that file, so anything not written there won't be built on our side.

## Please don't change these without talking to us first

These protect our customers and our pricing, so we'd want a conversation before they change:

1. **Customer data travels server to server only.** No personal data in URLs, query strings or the browser, apart from the opaque claim token.
2. **The claim link is single-use, short-lived and unguessable**, and it never reveals the cart to a second browser.
3. **The ship-to address is pre-filled with our factory and stays editable.** Customers who want us to install the device ship it to us; customers doing their own installation change it at checkout.
4. **Prices come from your catalog only.** We never send prices, and you don't accept them.
5. **Retries are safe.** A repeated request with the same idempotency key (or your equivalent) must not create a second cart.
6. **Separate sandbox and production**, with separate keys.

## Suggested build order

1. Partner record, API keys, and a sandbox environment (SPEC 5.1 items 1 and 3)
2. Create Cart endpoint with validation, idempotency and error envelope (SPEC 2, 5.1 item 2)
3. Claim endpoint, including every row of the behavior table (SPEC 3, 5.1 item 4)
4. Cart and checkout changes: pre-fill, editable factory ship-to, banner (SPEC 5.1 item 5)
5. Order linkage and the expiry job (SPEC 5.1 items 6 and 7)
6. Optional: cart status and catalog endpoints (SPEC 2.6)

The Vendor acceptance criteria in SPEC 5.3 are the definition of done. We'd appreciate automated tests for each one.

## What we need from you, and when

| When | What | How |
| --- | --- | --- |
| Before you start | Answers to the open decisions in SPEC section 8 | GitHub issue or PR |
| As you build | Any contract changes | PR to `SPEC.md` |
| When the sandbox is ready | Sandbox base URL, `partner_id`, real SKUs, default options, fallback URL, supported ship-to regions, guest checkout yes or no (SPEC 5.2) | PR or issue. These are not secret. |
| When the sandbox is ready | Sandbox API key | **Secure channel only**, never this repo, an issue or email |
| Before go-live | Production details and key, technical and escalation contacts | Secure channel |

When your sandbox is up and `SPEC.md` matches it, open an issue titled **"Sandbox ready"** and we'll start building our side against it.

## This repo is public

Never commit or post API keys, credentials, internal hostnames you'd rather keep private, real customer data, or personal phone numbers. Use placeholders such as `{VENDOR_API_KEY}` in examples.
