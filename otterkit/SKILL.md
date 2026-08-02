---
name: otterkit-tunnel
description: Expose a local port to the internet via OtterKit tunnel, or create a webhook endpoint to capture incoming HTTP requests. Captured requests can be inspected and replayed against the local server, and tunnels can be protected with HTTP Basic auth. Use when the user asks to "tunnel", "expose", "share my localhost", "make my local server public", needs a public URL for a local service, needs a webhook endpoint to capture requests, or wants to re-test a webhook handler against a previously received payload.
---

# OtterKit Tunnel

Expose a local port to the internet instantly via a secure tunnel. Paid with prepaid **OtterKit credits** (1 credit = $0.01), metered by time: **1 credit per connected hour** (first hour charged at provision), **never more than 300 credits ($3) per endpoint per rolling 30 days** - webhooks and tunnels alike. Billing pauses while disconnected, and tunnels auto-stop after a TTL (default 24h) so a forgotten tunnel stops billing. The user logs in once with `otterkit login`; after that the CLI (and any agent on the same machine) provisions automatically, debiting the user's credit balance.

## Prerequisites

The user must be logged in. One-time:

```bash
npx otterkit login
```

This opens the browser, the user approves the device, and a token is saved to `~/.otterkit/credentials.json`. Buy credits at https://console.otterkit.com. New accounts get a small free-credit grant to start.

For headless/CI agents, set `OTTERKIT_TOKEN` (create a token at console.otterkit.com → API Tokens) instead of running `otterkit login`.

Check the logged-in account and balance:

```bash
npx otterkit whoami
npx otterkit balance
```

## When to Use

- User asks to expose a local port or server to the internet
- User needs a public URL for a local development server
- User wants to share their localhost with others
- User asks to create a tunnel
- User needs a webhook endpoint pointing to their local machine
- User needs a long-running tunnel that survives terminal close
- User needs a webhook endpoint to capture incoming HTTP requests
- User wants to test webhooks without running a local server
- User needs to receive webhook callbacks from third-party services (Stripe, GitHub, Slack, etc.)
- User needs to debug webhook integrations (capture + forward)
- User wants to capture HTTP traffic passing through a tunnel for later inspection
- User needs to replay or inspect webhook payloads
- User fixed a webhook handler and wants to re-test it against a real captured payload
- User wants the public tunnel URL protected so only callers with credentials reach their server

## Commands

### Foreground Tunnel (1 credit/hour, alive while terminal is open)

```bash
npx otterkit tunnel <port>
```

The tunnel stays alive as long as the terminal is open. Press Ctrl+C to disconnect.

### Daemon Tunnel (background, survives terminal close)

```bash
npx otterkit tunnel <port> --daemon [--ttl <duration>]
```

Runs in the background as a detached process. Same hourly pricing; the TTL is an auto-stop (default 24h, max 7d), not a price.

#### Pricing (all modes)

| What                 | Cost                                            |
| -------------------- | ----------------------------------------------- |
| Per connected hour   | 1 credit ($0.01), first hour at provision       |
| Daily cap per tunnel | 10 credits - hours beyond are free              |
| While disconnected   | Free - billing pauses                           |
| Auto-stop TTL        | `--ttl 45m` / `4h` / `3d` (default 24h, max 7d) |

### Stable URL (`--subdomain`)

By default every tunnel/webhook gets a fresh random URL (`https://agent-<hex>.otterkit.app`).
To get a **persistent URL that stays the same across runs**, pass `--subdomain <name>`:

```bash
npx otterkit tunnel 3000 --subdomain myapp   # https://myapp.otterkit.app, every time
```

The name is claimed to the user's account the first time it's used and reused after that, so
the same public URL works across restarts and machines. Use this when the user needs a fixed
URL to hand to a third party (a webhook provider, an OAuth callback, a teammate). Holding a
name is free - pricing is unchanged (1 credit/hour only while connected). Works on `tunnel`
and `webhook`, foreground and `--daemon`.

If the name is already taken by another account the command fails with a "taken" error and
**no credit is charged** - pick a different name or drop the flag for an auto name. Manage
reserved names with:

```bash
npx otterkit subdomains                 # list reserved names
npx otterkit subdomains reserve myapp   # reserve without starting a tunnel
npx otterkit subdomains release myapp   # release a name
```

### Protect a Tunnel with Basic Auth (`--auth`)

```bash
npx otterkit tunnel 3000 --auth admin:s3cret
```

Requires HTTP Basic auth on every request to the public URL. Enforced by the CLI on the
local machine **before** anything reaches the local server - requests without valid
credentials get a `401` and are never forwarded. Credentials are never sent to or stored by
OtterKit's servers, and auth checks are free. Callers authenticate the standard way:

```bash
curl -u admin:s3cret https://agent-a1b2c3d4.otterkit.app/
```

Tunnel command only (foreground and `--daemon`); webhook endpoints are capture-only and
stay open. Use this when exposing anything sensitive (admin UIs, internal APIs, demos).

### Project Config (`otterkit up` / `down`)

If the project has an `otterkit.toml`, bring every defined tunnel up with one command.
`up` is **idempotent** - already-running profiles are skipped (`already_running`), so it is
safe to run repeatedly. Each profile is a normal daemon (`status`/`inspect`/`replay`/`stop`
work on them; normal pricing per profile).

```toml
[tunnels.web]
port = 3000
subdomain = "myapp"        # optional stable URL
auth = "admin:s3cret"      # optional HTTP Basic auth
log = true                 # optional request capture
ttl = "8h"                 # optional auto-stop (default 24h)

[tunnels.hooks]
webhook = true             # capture-only endpoint
respond = 200              # optional custom auto-response
respond_body = '{"ok":true}'
```

```bash
npx otterkit up --json     # start everything, machine-readable results
npx otterkit down          # stop the daemons started from the config
```

### JSON Output (use this when scripting)

Prefer `--json` for reliable parsing - it works on `tunnel --daemon`, `webhook --daemon`,
`up`, `down`, `status`, `inspect`, `replay`, `subdomains list`, `whoami`, and `balance`.
Provision results include `{subdomain, publicUrl, target, pid, ttl, expiresAt, logPath}`.
Errors are JSON too (e.g. `{"error":"insufficient_credits","topUpUrl":"..."}`) with exit
code 1.

```bash
URL=$(npx otterkit tunnel 3000 --daemon --json | jq -r .publicUrl)
```

### Webhook Endpoint (1 credit/hour, captures incoming HTTP requests)

```bash
npx otterkit webhook
```

Default auto-response is `200 {"received":true}`. If the provider requires a specific
status/body before delivering events (challenge echo, strict 2xx), customize it:

```bash
npx otterkit webhook --respond 204
npx otterkit webhook --respond 200 --respond-body '{"challenge":"accepted"}'
```

Creates a webhook endpoint that captures incoming HTTP requests without needing a local server. Every request is saved to a local JSONL file (`~/.otterkit/requests/<subdomain>.jsonl`) for later inspection with `otterkit inspect`. Same pricing as tunnels. Supports daemon mode with `--daemon` and `--ttl` flags.

```bash
npx otterkit webhook --daemon [--ttl <duration>]
```

With `--standby`, the endpoint stays live even while the CLI is disconnected (laptop closed, daemon killed): the server answers each request with the configured auto-response and buffers up to 200 captures (64 KB bodies), replaying them into the local log on the next connect. Providers never see downtime. Billing continues while disconnected (still capped at 10 credits/day; the session still auto-stops at its `--ttl`).

```bash
npx otterkit webhook --standby [--respond <status>] [--respond-body <data>]
```

### Capture Tunnel Traffic (`--log`)

```bash
npx otterkit tunnel <port> --log
```

Forwards traffic like a normal tunnel while also capturing every request to the JSONL log. Useful for debugging webhook integrations against a real local server.

### Inspect Captured Requests

```bash
npx otterkit inspect <subdomain>         # view captured requests (last 20)
npx otterkit inspect <subdomain> --json  # raw JSONL output for piping
npx otterkit inspect <subdomain> --last 50  # show last 50 requests
npx otterkit inspect <subdomain> --follow   # live-tail new requests (Ctrl+C to stop)
npx otterkit inspect <subdomain> --method POST --status 5xx --path /hook  # filters (combinable)
npx otterkit inspect <subdomain> --har > session.har  # HAR 1.2 export for devtools/HAR viewers
```

`--status` accepts an exact code (`500`) or a class (`5xx`). Filters also apply to
`--json`, `--follow`, and `--har`.

### Replay a Captured Request

```bash
npx otterkit replay <subdomain>                          # re-send the latest captured request
npx otterkit replay <subdomain> --index 3                # re-send request #3 (1 = oldest, -1 = latest)
npx otterkit replay <subdomain> --target 127.0.0.1:3000  # explicit target (webhook captures / stopped tunnels)
npx otterkit replay <subdomain> --json                   # machine-readable: {status, headers, body (base64), durationMs}
```

Re-sends a captured request straight to the local server - no tunnel round-trip, **no
credits spent**. Ideal loop for debugging a webhook handler: capture the real payload once,
fix the code, `replay` until it returns 200. The target defaults to the running daemon's
`host:port` for that subdomain; pass `--target` otherwise. The replayed exchange is appended
to the capture log, so `inspect` shows it. Exit code 0 whenever the local server responded
(even 4xx/5xx), 1 if it was unreachable.

The request can be modified before re-sending:

```bash
npx otterkit replay <subdomain> --method PUT --path /v2/hook \
  -H "X-Debug: 1" --body '{"event":"payment.failed"}'
```

### Account

```bash
npx otterkit login    # browser device-flow login (one-time)
npx otterkit whoami   # show account + balance
npx otterkit balance  # show credit balance
npx otterkit logout   # remove the saved token
```

### Check Running Daemons

```bash
npx otterkit status
```

Shows all running daemon tunnels with their public URL, target, TTL remaining, and PID.

### Stop a Daemon

```bash
npx otterkit stop <subdomain>
```

Stops a running daemon tunnel by its subdomain (e.g., `agent-a1b2c3d4`).

## Options

| Flag                 | Description                                             | Default             |
| -------------------- | ------------------------------------------------------- | ------------------- |
| `--host <host>`      | Local host to forward to                                | `127.0.0.1`         |
| `--log`              | Capture requests to a JSONL log (tunnel only)           | off                 |
| `--daemon`           | Run tunnel in background                                | off                 |
| `--ttl <duration>`   | Auto-stop after duration, e.g. 4h, 3d (max 7d)          | `24h`               |
| `--subdomain <name>` | Use a stable reserved URL (claimed on first use)        | auto                |
| `--auth <user:pass>` | Require HTTP Basic auth (tunnel only, enforced locally) | off                 |
| `--respond <status>` | Webhook auto-response status (webhook only)             | `200`               |
| `--respond-body <d>` | Webhook auto-response body (webhook only)               | `{"received":true}` |
| `--standby`          | Server answers + buffers while disconnected (webhook)   | off                 |
| `--json`             | Machine-readable output (daemon/read commands)          | off                 |

## Examples

```bash
# Expose port 3000 (foreground)
npx otterkit tunnel 3000

# Expose port 8080 on a custom host
npx otterkit tunnel 8080 --host 0.0.0.0

# Background tunnel for 4 hours
npx otterkit tunnel 3000 --daemon --ttl 4h

# Short-lived background tunnel (auto-stops after 1 minute)
npx otterkit tunnel 4008 --daemon --ttl 1m

# Create a webhook endpoint (foreground)
npx otterkit webhook

# Create a background webhook, auto-stops after 4 hours
npx otterkit webhook --daemon --ttl 4h

# Tunnel with request capture (view later with `inspect`)
npx otterkit tunnel 3000 --log

# Background capture tunnel for 4 hours
npx otterkit tunnel 3000 --log --daemon --ttl 4h

# View captured requests
npx otterkit inspect agent-a1b2c3d4

# View as raw JSONL (pipe-friendly)
npx otterkit inspect agent-a1b2c3d4 --json

# Re-send the latest captured request to the local server
npx otterkit replay agent-a1b2c3d4

# Re-send with a modified body, machine-readable result
npx otterkit replay agent-a1b2c3d4 --body '{"event":"retry"}' --json

# Tunnel protected with Basic auth
npx otterkit tunnel 3000 --auth admin:s3cret

# Check running daemons
npx otterkit status

# Stop a daemon
npx otterkit stop agent-a1b2c3d4
```

## How It Works

1. `otterkit login` runs a browser device-flow and saves an API token to `~/.otterkit/credentials.json`.
2. The CLI sends that token when provisioning; the server debits the user's credit balance.
3. A public URL like `https://agent-a1b2c3d4.otterkit.app` is created.
4. All HTTP traffic to that URL is forwarded to your local port.
5. Foreground tunnels auto-reconnect on network interruption or laptop wake.
6. Daemon tunnels run as detached processes and auto-expire after TTL.

## Troubleshooting

If the command fails with "Not logged in", log in (one-time):

```bash
npx otterkit login
```

If it fails with "Out of credits" / "insufficient_credits", top up at https://console.otterkit.com/billing.

To check pricing:

```bash
curl https://otterkit.app/api/agent/pricing
```
