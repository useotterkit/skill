---
name: otterkit-shopify-webhooks
description: Develop and debug Shopify webhooks and apps locally with OtterKit - stable tunnel URLs for `shopify app dev --tunnel-url`, persistent endpoints that survive the 19-consecutive-failure subscription removal, X-Shopify-Hmac-Sha256 verification, replay of orders/create and orders/paid deliveries, and synthetic signed Shopify events. Use when the user is building a Shopify app or webhook handler, hits Cloudflare quick-tunnel errors in the Shopify CLI, or needs to test Shopify HMAC verification locally.
---

# Shopify Webhooks with OtterKit

Two Shopify-specific problems, solved: unstable dev-tunnel URLs, and Shopify
removing webhook subscriptions after 19 consecutive failed deliveries (a dead
tunnel during retries silently kills the subscription).

Prerequisites: logged in via `npx otterkit login` (see the `otterkit-tunnel` skill for
accounts, credits, and pricing).

## Stable tunnel for `shopify app dev`

The Shopify CLI accepts any tunnel provider. A reserved subdomain gives the same
URL every session, so app URLs in the Partner Dashboard stop churning:

```bash
npx otterkit tunnel 3000 --subdomain myapp --daemon
shopify app dev --tunnel-url https://myapp.otterkit.app:443
```

Add `--log` to the tunnel to capture everything Shopify sends while the app runs
(inspect later with `npx otterkit inspect myapp`).

## Fastest loop: synthetic signed events (free, no endpoint needed)

```bash
npx otterkit send shopify:orders/create 127.0.0.1:3000/webhooks --secret shpss_xxx
npx otterkit send --list          # all providers and event types
```

Signed with a real `X-Shopify-Hmac-Sha256` header, so the handler's HMAC check runs
for real. Events include `orders/create` and `orders/paid`; override the payload
with `--body '<json>'`.

## Capture real deliveries without risking the subscription

```bash
npx otterkit webhook --subdomain myapp-hooks --daemon --standby
# → https://myapp-hooks.otterkit.app
```

Register that URL for the webhook subscription. The endpoint answers `200`
immediately and captures every delivery; with `--standby` it keeps answering even
while your machine is offline - the 19-failure counter never starts.

```bash
npx otterkit inspect myapp-hooks --follow                # live-tail
npx otterkit await myapp-hooks --count 1 --timeout 10m   # block until delivery
```

## Verify HMAC

```bash
npx otterkit verify myapp-hooks shopify --secret shpss_xxx
```

The secret is the app's API secret key (webhooks signed with it) or the secret
shown when creating the webhook in admin. Reports `signature_mismatch` /
`missing_signature_header` per captured request.

## Replay against local code

```bash
npx otterkit replay myapp-hooks --target 127.0.0.1:3000
```

Free, repeatable re-delivery of a real captured order event to the local handler.
Shopify's HMAC covers only the body (no timestamp), so replays verify as-is; after
editing the body, re-sign so the handler still accepts it:

```bash
npx otterkit replay myapp-hooks --target 127.0.0.1:3000 \
  --body '{"id":999,...}' --resign shopify --secret shpss_xxx
```
