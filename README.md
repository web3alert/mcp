# Web3alert MCP server

Decoded on-chain events for AI agents. Connect an MCP client to Web3alert's hosted server, find a trigger in the catalog of EVM and Substrate chains (Ethereum, Polkadot, Kusama and more), subscribe your webhook to it, and Web3alert sends each matching event to it as JSON, signed whenever the webhook has a signing secret (every new webhook gets one). Nothing to install: the server is hosted.

| | |
|---|---|
| Endpoint | `https://web3alert.io/mcp/agent` |
| Transport | Streamable HTTP, stateless |
| Authentication | `initialize` and `tools/list` work without a token; tool calls need `Authorization: Bearer <personal access token>` |
| MCP Registry | [`io.web3alert/agent`](https://registry.modelcontextprotocol.io/v0/servers?search=io.web3alert) |
| Docs | [Agent quickstart](https://docs.web3alert.io/en/agents) · [MCP reference](https://docs.web3alert.io/en/mcp-server) · [Webhooks](https://docs.web3alert.io/en/webhooks) |

This repository describes the server and holds its registry entry ([`server.json`](server.json)). The server's source code is not public.

## Connect

1. A person signs in at [web3alert.io](https://web3alert.io) and copies the personal access token from the account page. Agents cannot open their own accounts yet.
2. Point your MCP client at `https://web3alert.io/mcp/agent` and send the token in the `Authorization` header. The server accepts the token only in that header. You can list the tools without a token; a tool call without one gets HTTP `401` with instructions, and a wrong token gets `401` on every request.

**Claude Code**: this reads the token without echoing it or keeping it in shell history. Claude Code saves the header in its user configuration, so treat that file as a secret.

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
| `get_trigger` | A trigger's runtime inputs, event fields and filterable fields: use it to learn which inputs a subscription must supply. | read |
| `list_webhooks` | Webhook endpoints in the current workspace, with their signing status. | read |
| `add_webhook` | Registers an HTTPS webhook endpoint and returns its signing secret once. | write |
| `create_subscription` | Delivers one trigger to one webhook, with the trigger's required inputs and optional filter conditions. | write |
| `list_subscriptions` | Subscriptions in the current workspace. | read |
| `set_subscription_state` | Pauses or resumes a subscription. | write |
| `remove_subscription` | Deletes a subscription and stops its delivery. | write |
| `get_delivery_log` | Delivery history for the workspace or one subscription; paged with `cursor`. | read |

Every tool declares an output schema, and a successful call returns `structuredContent`. A tool error returns `isError: true` with a JSON error (for example `subscription_limit_reached`, `not_found`, `rate_limited`) in its text content.

A typical first session: `get_account` → `find_triggers` → `get_trigger` → `add_webhook` → `create_subscription` → `get_delivery_log`. To receive events you need a public HTTPS endpoint; `add_webhook` refuses a URL that carries a username or password.

- `create_subscription` takes the trigger's required inputs and optional filter conditions. A create or a resume can succeed with `state: "blocked"` at the plan's subscription limit, so check that the returned `state` is `on` before you expect events. If a create got no reply, call `list_subscriptions` before retrying, or you may create a duplicate.
- After a pause or removal, pending retries stop within about 10 seconds; an attempt already due in that window can still arrive.
- `get_delivery_log` follows the plan of the first account with the `owner` role in the workspace (your own plan only when there is none), not necessarily your tier from `get_account`; on a plan without delivery history it returns `not_in_plan`, so treat the last step of the session as optional.

Tools act on the account's current workspace, which you choose in the app. Adding a webhook, creating a subscription and reading delivery history need the `owner` or `admin` role in that workspace.

## What agents cannot do here

Agents on this endpoint cannot author or change triggers, projects, templates or data sources; change billing, plans or payments; manage tokens; delete accounts or workspaces; block subscriptions; or manage resources other than webhook endpoints.

People who build catalog triggers use the full authoring server at `https://web3alert.io/mcp` with the same token. Creating projects there needs a paid plan.

## What your webhook receives

Each delivery is a JSON body whose `id` (`evt_…`) stays the same across retries and equals the `webhook-id` header. Every delivery carries `webhook-id` and `webhook-timestamp` headers; an endpoint that currently has a signing secret also gets `webhook-signature` per [Standard Webhooks](https://www.standardwebhooks.com/): `v1,` followed by the base64 HMAC-SHA256 of `webhook-id.webhook-timestamp.raw-body`, keyed with the base64-decoded part of the `whsec_…` secret.

- On a signed endpoint, verify the signature against the raw body before parsing JSON, and reject a `webhook-timestamp` more than five minutes from your clock. If your receiver requires signatures, also reject unsigned deliveries. A secret change takes about 10 seconds to apply, and moving a project to another workspace removes the secrets of the webhooks that move with it; generate a new secret for those.
- Delivery is at least once: deduplicate by `webhook-id`.
- Answer with a `2xx` quickly: requests time out after 4 seconds by default. HTTP `429`, `5xx`, timeouts and network errors before a response are retried, waiting at least 5 s, 1 min, 10 min, 1 h and 6 h between attempts (at most six attempts; the minimum schedule alone spans about 7 hours 11 minutes, and an outage on Web3alert's side can delay or expire a retry). A failed delivery that cannot be queued for retry (the queue is full, or the queued record with its event data exceeds 256 KB) is not retried. Other responses end the delivery, redirects are not followed, and test deliveries are never retried.
- An HTTP `410` blocks the subscription at once. Five consecutive undelivered events for the same subscription and webhook block it too.
- Each plan limits the delivery rate per subscription; events beyond the plan's queue can be dropped (see the [limits page](https://docs.web3alert.io/en/limits)).

The [Webhooks page](https://docs.web3alert.io/en/webhooks) has the payload, verification code in Node.js and Python, a test vector, and which responses are retried.

## Limits

- Free: 5 active subscriptions to non-Free projects per account, across all workspaces; subscriptions to Free projects are exempt. Paid plans and their delivery-history retention are on the [limits page](https://docs.web3alert.io/en/limits).
- A default budget of 120 tool calls per minute per account on each server process, shared by both MCP endpoints and all of the account's tokens. Over the budget, a tool returns `rate_limited` with `retryAfter` in seconds.

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
