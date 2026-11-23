# Web3alert MCP server

Connect an MCP client to Web3alert's hosted server for decoded on-chain events. Find a trigger in the catalog of EVM and Substrate chains, including Ethereum, Polkadot and Kusama, and subscribe to receive JSON at your webhook or in a paid workspace inbox. The build tools generate private triggers from a contract ABI, Solana IDL or Substrate metadata.

| | |
|---|---|
| Agent endpoint | `https://web3alert.io/mcp/agent` |
| Build endpoint | `https://web3alert.io/mcp/agent/build` adds trigger-building tools for Advanced and Pro accounts |
| Transport | Streamable HTTP, stateless |
| Authentication | Individual `initialize` and `tools/list` requests work without a key on both endpoints. Tool calls need `Authorization: Bearer <agent key>`. |
| MCP Registry | [`io.web3alert/agent`](https://registry.modelcontextprotocol.io/v0/servers?search=io.web3alert) |
| Docs | [Agent quickstart](https://docs.web3alert.io/en/agents) · [MCP reference](https://docs.web3alert.io/en/mcp-server) · [Webhooks](https://docs.web3alert.io/en/webhooks) |

This repository describes the hosted server and holds its registry entry ([`server.json`](server.json)). The server's source code is not public.

## Connect

1. **Create a key in the app.** Sign in, select a workspace, and create an agent key with its own scopes, optional monthly spend cap and optional expiry. Copy the key when it appears. Agent keys have `read` and can also have `manage`, `spend` and `code` (the tools on `/mcp/agent` and `/mcp/agent/build` work without `code`). `add_webhook` and `create_subscription` require the key's `manage` scope. [Agents page](https://web3alert.io/agents)
2. **Sign up with an Ethereum wallet.** From an externally owned account (EOA), call `POST /api/agent/siwe/nonce`, sign the Sign-In with Ethereum (SIWE) message, then send the message and signature to `POST /api/agent/siwe/verify`. On first sign-in, it creates a Free account and returns account and key details, the bearer `secret`, and a `claimUrl` for your human. Keys issued by wallet sign-in have `read`, `manage`, `spend` and `code`. A later sign-in can return `claimUrl: null` while a link is active. Treat `claimUrl` as an account credential and share it privately only with the intended person. Keep it out of logs and public issues. [Agent quickstart](https://docs.web3alert.io/en/agents#sign-up-with-an-ethereum-wallet)

The Account page keeps the browser session token hidden. Existing session tokens keep working until you sign out or revoke them.

**Claude Code** in Bash or Zsh prompts for the key without echoing it.

```bash
printf 'Web3alert agent key: ' && read -rs WEB3ALERT_KEY && echo && [ -n "$WEB3ALERT_KEY" ] && claude mcp add --scope user --transport http web3alert-agent https://web3alert.io/mcp/agent --header "Authorization: Bearer $WEB3ALERT_KEY"
```

Claude Code saves the header in its user configuration, so treat that file as a secret. If a `web3alert-agent` entry already exists, follow the [replacement steps in the MCP reference](https://docs.web3alert.io/en/mcp-server#upgrading-existing-clients).

**VS Code** uses `.vscode/mcp.json` with `WEB3ALERT_KEY` in the VS Code process environment.

```json
{
  "servers": {
    "web3alert-agent": {
      "type": "http",
      "url": "https://web3alert.io/mcp/agent",
      "headers": { "Authorization": "Bearer ${env:WEB3ALERT_KEY}" }
    }
  }
}
```

**Claude Desktop and Cursor** use the `mcp-remote` entry. [MCP reference](https://docs.web3alert.io/en/mcp-server#claude-desktop-and-cursor)

### Paying

Before paying, use `get_account` to confirm the key has the `spend` scope and the planned purchase fits within its remaining monthly cap, if set. Check Billing or ask the account owner whether an active auto-renewing plan (`key_not_allowed`), a higher active plan or a pending checkout (both `api_error`) blocks the purchase.

`get_topup_requirements` (needs the key's `manage` scope) returns a USDC quote and the x402 payment requirements; the agent then pays at `POST /api/billing/wallet/x402-topup`. Top-ups are €5–€500 in USDC on Base at the ECB reference rate with no markup; the quote rounds up to a USDC cent. Your agent signs the payment with its own wallet, and the facilitator pays the gas. [Agent quickstart and payment example](https://docs.web3alert.io/en/agents)

If a signed top-up gets no reply, times out, or returns `202` with `status: "pending"` or a `5xx`, resend the same request in the same workspace with the same body and saved `PAYMENT-SIGNATURE`, limiting retries to a few attempts and increasing the pause between them. Wait for the first payment's outcome to be confirmed before signing a new quote, because a fresh signature authorizes a separate payment. [x402 retry instructions](https://docs.web3alert.io/en/api-billing#post-api-billing-wallet-x402-topup)

Once the EUR balance is credited, use `buy_plan` (needs the `spend` scope) to buy Advanced or Pro without auto-renewal. The key's optional monthly spend cap applies to balance purchases made with that key. Reuse the same `requestId` if you retry the purchase.

## Tools

### `/mcp/agent`

The agent endpoint lists 14 tools.

| Tool | What it does | Kind |
|---|---|---|
| `get_account` | Shows your tier, EUR balance, tier limits, workspace subscription counts when available and key details. | read |
| `get_topup_requirements` | Returns a USDC quote and the x402 payment requirements. | read |
| `buy_plan` | Buys Advanced or Pro from your EUR balance without auto-renewal. | write |
| `find_triggers` | Searches the published catalog and your workspace for triggers, paged with `cursor`. | read |
| `get_trigger` | Shows a trigger's runtime inputs, event fields and filterable fields. | read |
| `list_webhooks` | Lists workspace webhooks and their signing status. | read |
| `add_webhook` | Registers an HTTPS webhook endpoint and returns its signing secret once. | write |
| `remove_webhook` | Deletes a workspace webhook if no live subscription in the workspace uses it. | write |
| `create_subscription` | Subscribes one trigger to a webhook or paid inbox, with required inputs and optional filters. | write |
| `list_subscriptions` | Lists subscriptions in the current workspace. | read |
| `set_subscription_state` | Pauses or resumes a subscription. | write |
| `remove_subscription` | Deletes a subscription and stops its delivery. | write |
| `get_delivery_log` | Shows delivery history, on plans that include it, for the workspace or one subscription, paged with `cursor`. | read |
| `read_events` | Reads retained events from the paid workspace inbox, oldest first. | read |

### `/mcp/agent/build`

The build endpoint lists 19 tools in total, including all 14 agent tools and the five below. Calling the additional tools requires Advanced or Pro on the caller's account.

| Tool | What it does | Kind |
|---|---|---|
| `find_contract` | Finds event signatures from a verified contract ABI, Solana IDL or Substrate metadata. | read |
| `plan_trigger` | Drafts a trigger for a selected event and returns its inputs, filterable fields and output schema. | write |
| `test_trigger` | Tests a draft with a recent sample or a supplied block or slot. | read |
| `activate_trigger` | Saves a draft whose latest test matched to the workspace's private agent project. | write |
| `remove_my_trigger` | Removes a trigger from that project if no live subscription in the workspace uses it. | write |

Every tool on these two endpoints declares an output schema, and a successful call returns `structuredContent`. A tool error returns `isError: true` with a JSON error (for example `subscription_limit_reached`, `not_found`, `rate_limited`) in its text content.

A typical first session starts with `get_account` to check the key's workspace, then `find_triggers` and `get_trigger`. For webhook delivery, use `list_webhooks` or register a public HTTPS URL without credentials with `add_webhook`, then call `create_subscription` with that webhook. For inbox delivery, pass `inbox: true` to `create_subscription` and poll with `read_events`, passing each non-null `nextCursor` as `after` on the next call. If `gap: true`, treat the event history as incomplete because previously stored events expired or were pruned. Check `body.truncated` for oversized deliveries and deduplicate by event `id`.

- Supply the trigger's required inputs and any filters. A create or resume can return `state: "blocked"` at the subscription limit. Check that the returned state is `on`. If a create gets no reply, check `list_subscriptions` before retrying to avoid a duplicate.
- Inbox access, inbox retention and delivery history use the plan of the first workspace owner. If there is no owner, access and delivery-log reads use the caller's plan, and inbox storage uses the subscription creator's plan. Inbox access and delivery-log reads return `not_in_plan` if the applicable plan lacks delivery history.

Tools act in the workspace bound to the key. The workspace roles for adding a webhook, creating a subscription, reading the inbox and reading delivery history are `owner` and `admin`.

## What agents cannot do here

`/mcp/agent` has no trigger authoring tools. `/mcp/agent/build` creates and removes triggers in the workspace's private agent project. Neither endpoint edits projects, templates or data sources, or manages delivery resources beyond webhooks and the inbox. Agents can set subscription state to `on` or `off`. The service sets `blocked`.

Agent keys cannot manage credentials, delete accounts or workspaces, open a hosted plan checkout, change renewal settings or replace an active auto-renewing account plan. They can top up over x402 and buy a plan from the EUR balance as described above.

The full authoring tool set is at `https://web3alert.io/mcp` and accepts the same bearer header. With an agent key, the tools listed under [authoring permissions](https://docs.web3alert.io/en/mcp-server#authoring-permissions) require the `code` scope to save, patch, preview or test triggers or test providers (Solana and Substrate import tools require it only when saving). Creating projects in your own workspace requires a paid plan.

## What your webhook receives

Each delivery is a JSON POST. The body's `id` (`evt_…`) equals `webhook-id` and stays the same across retries. Every attempt includes `webhook-timestamp`. Endpoints with a signing secret also receive `webhook-signature`, which is `v1,` followed by the base64 HMAC-SHA256 of `webhook-id.webhook-timestamp.raw-body`, keyed with the base64-decoded part of the `whsec_…` secret. [Standard Webhooks](https://www.standardwebhooks.com/)

- Verify the signature against the raw body before parsing JSON and reject a `webhook-timestamp` more than five minutes from your clock. If your receiver requires signatures, reject unsigned deliveries. A secret change takes about 10 seconds to apply. Moving a project to another workspace clears the secrets of the webhooks that move with it. Generate a new secret for each moved webhook in the destination workspace and configure the receiver to verify with it.
- Deduplicate by `webhook-id`. The same event can arrive more than once.
- Return `2xx` within the request timeout, which defaults to 4 seconds. HTTP `429`, `5xx`, timeouts and network errors before a response are eligible for retry. The minimum waits are 5 s, 1 min, 10 min, 1 h and 6 h. On that schedule, six attempts span about 7 hours 11 minutes plus delivery time. A full retry queue or a queued record over 256 KB prevents a retry. A sender outage can delay retries until they expire. Other responses end delivery, redirects are never followed, and test deliveries are never retried.
- After a pause or removal, pending retries stop within about 10 seconds. An attempt already due in that window can still arrive.
- An HTTP `410` blocks the subscription at once. Five consecutive undelivered events for the same subscription and webhook block it too.
- The subscription creator's plan limits the delivery rate per subscription. Events beyond the plan's queue can be dropped. [Limits](https://docs.web3alert.io/en/limits)

See the Webhooks page for the payload, verification code in Node.js and Python, a test vector and retry details. [Webhooks](https://docs.web3alert.io/en/webhooks)

## Limits

- Free accounts can create up to 5 active subscriptions to non-Free projects across all workspaces. Subscriptions to Free projects are exempt.
- The default MCP budget is 120 tool calls per minute per account on each server process, shared across `/mcp/agent`, `/mcp/agent/build` and `/mcp` and across the account's credentials. When that budget is exhausted, a tool returns `rate_limited` with `retryAfter` in seconds.
- All agent keys on an account share an API budget of 60 requests per minute on Free, 300 on Advanced or 900 on Pro. API requests made by MCP with an agent key also count. [Plan limits and retention](https://docs.web3alert.io/en/limits)

## For machines

- Connection details and docs links in [`/llms.txt`](https://web3alert.io/llms.txt).
- The agent-facing REST API in [`/api/openapi.json`](https://web3alert.io/api/openapi.json).
- The MCP Registry entry in [`server.json`](server.json).

## Issues and support

Issues in this repository are public. Never post a key, a signing secret, or a webhook URL that carries credentials. This repository holds documentation only, so pull requests that change the server cannot be accepted here.

Report a security problem privately through this repository's **Security → Report a vulnerability**, not in an issue.

## License

The text and examples in this repository are under the [MIT License](LICENSE).
