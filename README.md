# Web3alert MCP server

Decoded on-chain events for AI agents. Connect an MCP client to Web3alert's hosted server, find a trigger in the catalog of EVM and Substrate chains (Ethereum, Polkadot, Kusama and more), subscribe your webhook to it, and receive each matching event as signed JSON. Nothing to install: the server is hosted.

| | |
|---|---|
| Endpoint | `https://web3alert.io/mcp/agent` |
| Transport | Streamable HTTP, stateless |
| Authentication | `Authorization: Bearer <personal access token>` on every request |
| MCP Registry | [`io.web3alert/agent`](https://registry.modelcontextprotocol.io/v0/servers?search=io.web3alert) |
| Docs | [Agent quickstart](https://docs.web3alert.io/en/agents) · [MCP reference](https://docs.web3alert.io/en/mcp-server) · [Webhooks](https://docs.web3alert.io/en/webhooks) |

This repository describes the server and holds its registry entry ([`server.json`](server.json)). The server's source code is not public.

## Connect

1. A person signs in at [web3alert.io](https://web3alert.io) and copies the personal access token from the account page. Agents cannot open their own accounts yet.
2. Point your MCP client at `https://web3alert.io/mcp/agent` and send the token in the `Authorization` header. The server accepts the token only in that header.

**Claude Code**: this reads the token without echoing it or keeping it in shell history.

```bash
printf 'Web3alert token: ' && read -rs WEB3ALERT_TOKEN && echo && [ -n "$WEB3ALERT_TOKEN" ] && claude mcp add --scope user --transport http web3alert-agent https://web3alert.io/mcp/agent --header "Authorization: Bearer $WEB3ALERT_TOKEN"
```

**VS Code** (`.vscode/mcp.json`, with `WEB3ALERT_TOKEN` in the VS Code process environment):

```json
{
  "servers": {
    "web3alert-agent": {
      "type": "http",
      "url": "https://web3alert.io/mcp/agent",
      "headers": { "Authorization": "Bearer ${env:WEB3ALERT_TOKEN}" }
    }
  }
}
```

**Claude Desktop and Cursor**: see the [MCP reference](https://docs.web3alert.io/en/mcp-server) for the `mcp-remote` entry.

## Tools

| Tool | What it does | Kind |
|---|---|---|
| `get_account` | Your account, tier, EUR wallet balance, tier limits, and subscription counts in the current workspace. | read |
| `find_triggers` | Searches the published catalog and your workspace for triggers; paged with `cursor`. | read |
| `get_trigger` | A trigger's runtime inputs, event fields and filterable fields, needed before subscribing. | read |
| `list_webhooks` | Webhook endpoints in the current workspace, with their signing status. | read |
| `add_webhook` | Registers an HTTPS webhook endpoint and returns its signing secret once. | write |
| `create_subscription` | Delivers one trigger to one webhook, with optional inputs and filter conditions. | write |
| `list_subscriptions` | Subscriptions in the current workspace. | read |
| `set_subscription_state` | Pauses or resumes a subscription. | write |
| `remove_subscription` | Deletes a subscription and stops its delivery. | write |
| `get_delivery_log` | Delivery history for the workspace or one subscription; paged with `cursor`. | read |

Every tool returns structured content with an output schema. A typical first session: `get_account` → `find_triggers` → `get_trigger` → `add_webhook` → `create_subscription` → `get_delivery_log`.

Tools act on the account's current workspace, which you choose in the app. Adding a webhook, creating a subscription and reading delivery history need the `owner` or `admin` role in that workspace.

## What agents cannot do here

Agents on this endpoint cannot author or change triggers, projects, templates or data sources; change billing, plans or payments; manage tokens; delete accounts or workspaces; block subscriptions; or manage resources other than webhook endpoints.

People who build catalog triggers use the full authoring server at `https://web3alert.io/mcp` with the same token. Creating projects there needs a paid plan.

## What your webhook receives

Each delivery is a JSON body whose `id` (`evt_…`) stays the same across retries and equals the `webhook-id` header. Every delivery carries `webhook-id` and `webhook-timestamp` headers; endpoints with a signing secret also get `webhook-signature` per [Standard Webhooks](https://www.standardwebhooks.com/): `v1,` followed by the base64 HMAC-SHA256 of `webhook-id.webhook-timestamp.raw-body`, keyed with the base64-decoded part of the `whsec_…` secret.

- Delivery is at least once: deduplicate by `webhook-id`.
- Answer with a `2xx` within 4 seconds. HTTP `429`, `5xx`, timeouts and network errors are retried after at least 5 s, 1 min, 10 min, 1 h and 6 h; other responses end the delivery, and redirects are not followed.
- Five consecutive undelivered events block the subscription.

The [Webhooks page](https://docs.web3alert.io/en/webhooks) has the payload, verification code in Node.js and Python, a test vector, and which responses are retried.

## Limits

- Free: 5 active subscriptions to non-Free projects per account, across all workspaces; subscriptions to Free projects are exempt. Paid plans and their delivery-history retention are on the [limits page](https://docs.web3alert.io/en/limits).
- 120 tool calls per minute per account on each server process. Over the budget, a tool returns `rate_limited` with `retryAfter` in seconds.

## Coming soon

- Wallet sign-in, so agents can open their own accounts.
- USDC top-ups over x402.
- An inbox to catch up on missed events.

## For machines

- [`/llms.txt`](https://web3alert.io/llms.txt): what Web3alert is for agents, in one page.
- [`/api/openapi.json`](https://web3alert.io/api/openapi.json): the agent-facing REST API.
- [`server.json`](server.json): the MCP Registry entry.

## Issues and support

Issues in this repository are public. Never post a token, a signing secret, or a webhook URL that carries credentials. This repository holds documentation only, so pull requests that change the server cannot be accepted here.

Report a security problem privately through this repository's **Security → Report a vulnerability**, not in an issue.

## License

The text and examples in this repository are under the [MIT License](LICENSE). The Web3alert service is governed by its own terms.
