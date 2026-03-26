---
name: otterkit-tunnel
description: Expose a local port to the internet via OtterKit tunnel, or create a webhook endpoint to capture incoming HTTP requests. Use when the user asks to "tunnel", "expose", "share my localhost", "make my local server public", needs a public URL for a local service, or needs a webhook endpoint to capture requests.
---

# OtterKit Tunnel

Expose a local port to the internet instantly via a secure tunnel. Payments are handled automatically via MPP (Machine Payments Protocol) using Tempo Wallet or mppx. The CLI auto-detects which wallet is available (Tempo Wallet first, mppx fallback) and prints the wallet address when connecting (e.g. "Using Tempo Wallet (0x1e69...)"). The server accepts both USDC and pathUSD on Tempo mainnet.

## Prerequisites

The user must have a payment wallet set up. Either option works:

**Option 1: Tempo Wallet (recommended)**

```bash
curl -fsSL https://tempo.xyz/install | bash && tempo wallet login
```

Tempo Wallet uses USDC by default. Access keys are scoped to USDC.

**Option 2: mppx**

```bash
npx mppx account create
npx mppx account fund
```

If the server asks for USDC but the mppx wallet holds pathUSD, the CLI auto-swaps via Tempo DEX.

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

## Commands

### Foreground Tunnel ($0.01, alive while terminal is open)

```bash
npx otterkit tunnel <port>
```

The tunnel stays alive as long as the terminal is open. Press Ctrl+C to disconnect.

### Daemon Tunnel (background, survives terminal close)

```bash
npx otterkit tunnel <port> --daemon [--ttl <duration>]
```

Runs in the background as a detached process. Auto-expires when TTL runs out.

#### Daemon Pricing

| TTL              | Price | Flag        |
| ---------------- | ----- | ----------- |
| 1 minute         | $0.01 | `--ttl 1m`  |
| 1 hour (default) | $0.01 | `--ttl 1h`  |
| 4 hours          | $0.03 | `--ttl 4h`  |
| 12 hours         | $0.05 | `--ttl 12h` |
| 24 hours         | $0.08 | `--ttl 24h` |

### Webhook Endpoint ($0.01, captures incoming HTTP requests)

```bash
npx otterkit webhook
```

Creates a webhook endpoint that captures incoming HTTP requests without needing a local server. Same pricing as tunnels. Supports daemon mode with `--daemon` and `--ttl` flags.

```bash
npx otterkit webhook --daemon [--ttl <duration>]
```

### Intercept (capture + forward, $0.01)

```bash
npx otterkit intercept <port>    # capture + forward to local port
npx otterkit intercept           # capture only, no local server
```

Captures every HTTP request to a local JSONL file (`~/.otterkit/requests/<subdomain>.jsonl`) while optionally forwarding to your local server. Useful for debugging webhook integrations — inspect or replay any captured request later.

Supports daemon mode:

```bash
npx otterkit intercept <port> --daemon --ttl 4h
npx otterkit intercept --daemon --ttl 1h
```

### Inspect Captured Requests

```bash
npx otterkit inspect <subdomain>         # view captured requests (last 20)
npx otterkit inspect <subdomain> --json  # raw JSONL output for piping
npx otterkit inspect <subdomain> --last 50  # show last 50 requests
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

| Flag               | Description                                             | Default         |
| ------------------ | ------------------------------------------------------- | --------------- |
| `--host <host>`    | Local host to forward to                                | `127.0.0.1`     |
| `--account <name>` | mppx account name to use for payment                    | default account |
| `--wallet <name>`  | Force wallet: `tempo` or `mppx` (overrides auto-detect) | auto-detect     |
| `--daemon`         | Run tunnel in background                                | off             |
| `--ttl <duration>` | Daemon TTL: 1m, 1h, 4h, 12h, 24h                        | `1h`            |

## Examples

```bash
# Expose port 3000 (foreground)
npx otterkit tunnel 3000

# Expose port 8080 on a custom host
npx otterkit tunnel 8080 --host 0.0.0.0

# Use a specific mppx account
npx otterkit tunnel 3000 --account work

# Background tunnel for 4 hours
npx otterkit tunnel 3000 --daemon --ttl 4h

# Quick 1-minute background tunnel
npx otterkit tunnel 4008 --daemon --ttl 1m

# Create a webhook endpoint (foreground, $0.01)
npx otterkit webhook

# Create a background webhook for 4 hours ($0.03)
npx otterkit webhook --daemon --ttl 4h

# Webhook with a specific mppx account
npx otterkit webhook --account work

# Intercept: capture + forward to local server ($0.01)
npx otterkit intercept 3000

# Intercept: capture only, no local server ($0.01)
npx otterkit intercept

# Intercept daemon for 4 hours
npx otterkit intercept 3000 --daemon --ttl 4h

# View captured requests
npx otterkit inspect agent-a1b2c3d4

# View as raw JSONL (pipe-friendly)
npx otterkit inspect agent-a1b2c3d4 --json

# Check running daemons
npx otterkit status

# Stop a daemon
npx otterkit stop agent-a1b2c3d4
```

## How It Works

1. The CLI pays for the tunnel via MPP using your Tempo Wallet or mppx keychain (no private key export)
2. A public URL like `https://agent-a1b2c3d4.otterkit.app` is created
3. All HTTP traffic to that URL is forwarded to your local port
4. Foreground tunnels auto-reconnect on network interruption or laptop wake
5. Daemon tunnels run as detached processes and auto-expire after TTL

## Troubleshooting

If the command fails with "No wallet found", set up one of the supported wallets:

**Option 1: Tempo Wallet (recommended)**

```bash
curl -fsSL https://tempo.xyz/install | bash && tempo wallet login
```

**Option 2: mppx**

```bash
npx mppx account create
npx mppx account fund
```

If the user has multiple accounts and wants to list them:

```bash
npx mppx account list
```

To check pricing:

```bash
curl https://otterkit.app/api/agent/pricing
```
