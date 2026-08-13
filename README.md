# OtterKit Agent Skills

Skills that teach AI coding agents (Claude Code, Cursor, Copilot — anything that
reads the [Agent Skills](https://code.claude.com/docs/en/skills) format) to drive
[OtterKit](https://www.otterkit.com): tunnels and serverless webhook endpoints with
capture, signature verification, and replay.

| Skill | What the agent learns |
|---|---|
| [`otterkit`](otterkit/SKILL.md) | Core: tunnels, webhook endpoints, capture, inspect, replay, daemons, pricing |
| [`stripe-webhooks`](stripe-webhooks/SKILL.md) | Stripe: signed synthetic events, `Stripe-Signature` verification, re-signed replay |
| [`github-webhooks`](github-webhooks/SKILL.md) | GitHub Apps: persistent capture endpoint, `X-Hub-Signature-256` verification, replay |
| [`shopify-webhooks`](shopify-webhooks/SKILL.md) | Shopify: stable `--tunnel-url` for the CLI, HMAC verification, standby endpoints |
| [`slack-webhooks`](slack-webhooks/SKILL.md) | Slack: Events API via capture tunnel, signing-secret verification, re-signed replay |

## Install (Claude Code)

Copy any skill directory into `.claude/skills/` in your project (or `~/.claude/skills/`
for all projects):

```bash
git clone https://github.com/useotterkit/skill && cp -r skill/stripe-webhooks .claude/skills/
```

Or add the OtterKit MCP server, which exposes the same capabilities as tools:

```bash
claude mcp add otterkit -- npx otterkit mcp
```

## What's different from a plain tunnel

Every skill leans on three things ngrok-class tools don't do:

- **`otterkit send`** — synthetic provider events signed with real signature headers
  (`stripe:payment_intent.succeeded`, `github:pull_request.opened`, …) against a local
  handler, free, no provider account interaction.
- **`otterkit verify`** — check captured deliveries against a signing secret, per
  request, with the failure reason.
- **`otterkit replay --resign`** — re-deliver a captured event to local code, re-signed
  with a current timestamp so timestamp-tolerant handlers (Stripe, Slack) still accept it.

Skills are generated from the CLI's actual implementation and updated alongside it.
