# Instructions for coding agents

You are probably helping the **Vendor** (the Starlink reseller) build their side of this integration. The **Partner** (Orb Aerospace) builds the other side separately, from the same documents.

## Source of truth

- [`SPEC.md`](SPEC.md) is the contract. [`VENDOR_GUIDE.md`](VENDOR_GUIDE.md) explains how to work with it.
- The Vendor builds sections 2, 3 and 5 of the spec. Section 6 is the Partner's build: use it only to understand what the Partner will send.

## Rules

1. **Build to the spec by default.** Use the same paths, field names, types, status codes and error codes.
2. **Deviations are allowed, but they must be written down.** If the Vendor's platform needs something different, propose it to your human. Once they agree, update `SPEC.md` in the same change so it describes what you actually built. The Partner's code is generated from `SPEC.md`, so an unrecorded difference will break the integration.
3. **Don't silently guess.** Where the spec is silent or ambiguous, ask your human. If the answer affects the contract, record it in `SPEC.md` or open an issue.
4. **Keep the six protections in `VENDOR_GUIDE.md`** ("Please don't change these without talking to us first") unless your human has confirmed the Partner agreed to change one.
5. **Treat SPEC 5.3 as the definition of done**, and write a test for each acceptance criterion.
6. **This repo is public.** Never write secrets, API keys, real customer data or private contact details into it.

The Vendor's application code probably lives in the Vendor's own repository, not here. Use this repo for the contract, discussion and shared test fixtures.
