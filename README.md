# Starlink Partner Cart Handoff API

This repo is where Orb Aerospace (the **Partner**) and our Starlink reseller (the **Vendor**) work together on one small integration. When a customer orders from Orb and ticks "I want Starlink", they get a button that opens the Vendor's site with a cart already filled in: a Starlink Mini, a basic business plan, their billing details, and our factory pre-filled as the ship-to address. The factory address is only a default, for customers who want us to install the device; they can change it at checkout.

The Partner's server creates the cart with one authenticated server-to-server call. Only a short-lived, single-use claim link ever reaches the customer's browser.

## What's here

| File | For | Purpose |
| --- | --- | --- |
| [`SPEC.md`](SPEC.md) | Both teams | The API contract: endpoints, fields, errors, security, acceptance tests, go-live. This is the source of truth. |
| [`VENDOR_GUIDE.md`](VENDOR_GUIDE.md) | Vendor developers | Where to start, what you can change, and how to tell us. |
| [`AGENTS.md`](AGENTS.md) | Coding agents (Claude, Codex, Grok, Cursor, etc.) | Ground rules for an AI assistant working from this repo. |

## How we're running this

1. **The Vendor builds first.** You own the Create Cart API and the claim page (SPEC sections 2, 3 and 5). Build them the way your platform works best.
2. **The Partner fits to you.** The spec is our starting proposal, not a rule you have to follow word for word. If something doesn't fit your platform, change it. Just record the change here (see below) so we build against what you actually shipped.
3. **The spec stays true.** Any agreed change goes into `SPEC.md` through a pull request. When both sides start testing together, `SPEC.md` should describe the running API exactly.
4. **We connect in your sandbox** and run the joint tests in SPEC section 7.

## How to talk to us

- **Questions and open decisions:** open a GitHub issue. SPEC section 8 lists the decisions we need from you.
- **Changes to the contract:** open a pull request against `SPEC.md` that explains why. Small changes are fine to just make and tell us about.
- **Secrets never go in this repo.** It is public. API keys, production credentials, real customer data and internal contact numbers go through a secure channel we agree privately.

## Status

- [x] Spec published
- [ ] Vendor answers the open decisions (SPEC section 8)
- [ ] Vendor sandbox up, with the section 5.2 details shared
- [ ] Partner integration built against the sandbox
- [ ] Joint sandbox tests passed (SPEC section 7.1)
- [ ] Production go-live (SPEC section 7.2)
