---
name: otterkit-github-webhooks
description: Develop and debug GitHub webhooks and GitHub Apps locally with OtterKit - a reliable smee.io alternative. Capture push/pull_request/issues deliveries at a persistent endpoint, verify X-Hub-Signature-256 secrets, replay deliveries against a local handler, and send synthetic signed GitHub events. Use when the user is building a GitHub App, GitHub Action alternative, bot, or CI integration that receives GitHub webhooks, mentions smee.io or webhook proxy, or needs to test GitHub webhook signature verification locally.
---

# GitHub Webhooks with OtterKit

Local development for GitHub Apps and repo webhooks: persistent capture endpoint
(no smee.io channels that expire or drop large payloads), signature verification,
replay, and synthetic signed events.

Prerequisites: logged in via `npx otterkit login` (see the `otterkit-tunnel` skill for
accounts, credits, and pricing).

## Fastest loop: synthetic signed events (free, no endpoint needed)

```bash
npx otterkit send github:pull_request.opened 127.0.0.1:3000/api/webhook --secret mysecret
npx otterkit send --list          # all providers and event types
```

Signed with a real `X-Hub-Signature-256` header using the given secret, so the
handler's HMAC verification runs for real. Events include `pull_request.opened` and
`issues.opened`; `--body '<json>'` overrides the generated payload.

## Capture real GitHub deliveries

```bash
npx otterkit webhook --subdomain myapp-github --daemon
# → https://myapp-github.otterkit.app
```

Use that URL as the webhook URL in the repo/org settings or the GitHub App
manifest, with a webhook secret set. The endpoint answers `200` instantly and
captures every delivery - including the large payloads smee.io is known to drop.
`--standby` keeps capturing while your laptop is closed.

```bash
npx otterkit inspect myapp-github --follow                 # live-tail deliveries
npx otterkit await myapp-github --count 1 --timeout 5m     # block until one arrives
```

## Verify the webhook secret

```bash
npx otterkit verify myapp-github github --secret mysecret
```

Checks `X-Hub-Signature-256` on captured deliveries; reports `signature_mismatch` /
`missing_signature_header` per request - the fastest way to catch a secret typo
between GitHub settings and your handler.

## Replay against the local handler

```bash
npx otterkit replay myapp-github --target 127.0.0.1:3000
```

Re-sends a captured delivery (original headers, body, signature) to local code -
free, repeatable. GitHub's signature has no timestamp component, so replays pass
verification as-is when the secret matches. To test edited payload variants and
keep the signature valid:

```bash
npx otterkit replay myapp-github --target 127.0.0.1:3000 \
  --body '{"action":"closed",...}' --resign github --secret mysecret
```

## Typical GitHub App session

```bash
npx otterkit webhook --subdomain myapp-github --daemon      # 1. endpoint up
# 2. set as the App's webhook URL + secret, open a test PR
npx otterkit await myapp-github --count 1 --timeout 10m     # 3. wait for delivery
npx otterkit verify myapp-github github --secret mysecret   # 4. secret sanity check
npx otterkit replay myapp-github --target 127.0.0.1:3000    # 5. debug loop
npx otterkit stop myapp-github                              # 6. done
```
