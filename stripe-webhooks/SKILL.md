---
name: otterkit-stripe-webhooks
description: Develop and debug Stripe webhooks locally with OtterKit - capture real Stripe events at a persistent endpoint, verify Stripe-Signature headers, replay captured events against a local handler, and send synthetic signed Stripe events (payment_intent.succeeded, checkout.session.completed, invoice.paid) without triggering real payments. Use when the user is building or debugging a Stripe webhook handler, testing Stripe signature verification, needs a webhook URL for the Stripe Dashboard, or wants to test payment/checkout/invoice/subscription event handling locally.
---

# Stripe Webhooks with OtterKit

Develop a Stripe webhook handler end-to-end without deploying: capture real events,
verify signatures, replay until the handler passes, and generate signed test events.

Prerequisites: logged in via `npx otterkit login` (see the `otterkit-tunnel` skill for
accounts, credits, and pricing - webhooks bill like tunnels: 1 credit/hour, capped).

## Fastest loop: synthetic signed events (free, no endpoint needed)

Test a local handler directly - no tunnel, no Stripe account interaction, no charge:

```bash
npx otterkit send stripe:payment_intent.succeeded 127.0.0.1:3000/webhooks/stripe --secret whsec_xxx
npx otterkit send --list          # all providers and event types
```

The event is signed with a real `Stripe-Signature` header (`t=...,v1=...`) using the
given secret, so the handler's signature verification code runs for real. Use the same
secret the handler checks against. Available Stripe events include
`payment_intent.succeeded`, `checkout.session.completed`, `invoice.paid`, and
`customer.subscription.deleted`; override the generated payload with `--body '<json>'`.

## Capture real Stripe events

Create an endpoint and register it in the Stripe Dashboard (Developers → Webhooks →
Add endpoint), or via the Stripe API:

```bash
npx otterkit webhook --subdomain myapp-stripe --daemon
# → https://myapp-stripe.otterkit.app  (stable across restarts)
```

Stripe requires a 2xx response quickly; the endpoint auto-responds `200` immediately
and captures everything, so deliveries never fail while your code is broken - Stripe
won't disable the endpoint for repeated failures. With `--standby` it keeps answering
and buffering even while your machine is offline.

Watch events arrive:

```bash
npx otterkit inspect myapp-stripe --follow
npx otterkit await myapp-stripe --path /webhooks --count 1 --timeout 5m --json   # block until one arrives
```

## Verify signatures on captured events

Confirm captured deliveries are really from Stripe (secret is the endpoint's signing
secret, `whsec_...`, from the Dashboard):

```bash
npx otterkit verify myapp-stripe stripe --secret whsec_xxx
```

Reports per request: valid, or why not (`missing_signature_header`,
`signature_mismatch`, `malformed_signature_header`).

## Replay against the local handler until it passes

```bash
npx otterkit replay myapp-stripe --target 127.0.0.1:3000
```

Re-sends the captured event (original headers and body, including `Stripe-Signature`)
straight to the local server - free, no tunnel round-trip. Fix code, replay, repeat.
Note: Stripe signatures embed a timestamp, so handlers enforcing timestamp tolerance
(as the Stripe SDKs do by default) will reject old captures - re-sign the replay so
verification passes with the current time:

```bash
npx otterkit replay myapp-stripe --target 127.0.0.1:3000 --resign stripe --secret whsec_xxx
```

`--resign` also works after editing the body (`--body '<json>'`) to test variants of a
real event.

## Typical session

```bash
npx otterkit webhook --subdomain myapp-stripe --daemon        # 1. endpoint up
# 2. register URL in Stripe Dashboard, trigger a test payment
npx otterkit await myapp-stripe --count 1 --timeout 10m       # 3. wait for delivery
npx otterkit verify myapp-stripe stripe --secret whsec_xxx    # 4. confirm authenticity
npx otterkit replay myapp-stripe --target 127.0.0.1:3000 --resign stripe --secret whsec_xxx  # 5. debug loop
npx otterkit stop myapp-stripe                                # 6. done
```
