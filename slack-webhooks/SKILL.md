---
name: otterkit-slack-webhooks
description: Develop and debug Slack apps locally with OtterKit - tunnel the Events API to a local handler with request capture, verify X-Slack-Signature signing secrets, replay captured events (re-signed to pass Slack's timestamp tolerance), and send synthetic signed Slack events without needing a workspace. Use when the user is building a Slack app, bot, or Events API handler, mentions ngrok for Slack, needs a public Request URL for Slack's URL verification, or wants to test Slack signature verification locally.
---

# Slack Apps with OtterKit

Local development for the Slack Events API: a tunnel with capture for the
URL-verification handshake, then verify / replay / synthetic events for the debug
loop - no interstitial page in the way (Slack marks deliveries failed when a
tunnel answers with an HTML warning page).

Prerequisites: logged in via `npx otterkit login` (see the `otterkit-tunnel` skill for
accounts, credits, and pricing).

## Fastest loop: synthetic signed events (free, no endpoint needed)

Test the handler before any Slack configuration exists:

```bash
npx otterkit send slack:event_callback.message 127.0.0.1:3000/slack/events --secret <signing-secret>
npx otterkit send --list          # all providers and event types
```

Signed with real `X-Slack-Signature` + timestamp headers using the given secret, so
the handler's verification code (e.g. Bolt's built-in check) runs for real.
`--body '<json>'` overrides the payload.

## Tunnel for the Request URL (handles url_verification)

Slack's URL verification requires echoing a dynamic `challenge`, which your handler
does - so use a capture tunnel to the local app rather than a bare endpoint:

```bash
npx otterkit tunnel 3000 --subdomain myapp-slack --log --daemon
# Request URL: https://myapp-slack.otterkit.app/slack/events
```

The reserved subdomain keeps the Request URL stable across restarts (no re-verifying
in the Slack app config every session), and `--log` captures every event Slack sends.

```bash
npx otterkit inspect myapp-slack --follow                 # live-tail events
npx otterkit await myapp-slack --count 1 --timeout 5m     # block until one arrives
```

## Verify the signing secret

```bash
npx otterkit verify myapp-slack slack --secret <signing-secret>
```

Checks `X-Slack-Signature` on captured events; per-request results
(`signature_mismatch`, `missing_signature_header`) catch signing-secret mixups
between the Slack app config and the handler's environment.

## Replay captured events

Slack signatures embed a timestamp and handlers (including Bolt) reject stale ones,
so re-sign replays with the current time:

```bash
npx otterkit replay myapp-slack --target 127.0.0.1:3000 --resign slack --secret <signing-secret>
```

Free, repeatable re-delivery of a real captured event - fix the handler, replay,
repeat, without re-triggering the action in the workspace. Combine with
`--body '<json>'` to test payload variants that still pass verification.

## Typical session

```bash
npx otterkit tunnel 3000 --subdomain myapp-slack --log --daemon   # 1. stable Request URL
# 2. set Request URL in api.slack.com app config (verification passes via your handler)
# 3. trigger a message/mention in the workspace
npx otterkit verify myapp-slack slack --secret <signing-secret>   # 4. confirm authenticity
npx otterkit replay myapp-slack --target 127.0.0.1:3000 --resign slack --secret <signing-secret>  # 5. debug loop
npx otterkit stop myapp-slack                                     # 6. done
```
