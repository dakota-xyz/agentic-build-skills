---
name: dakota-webhooks
description: |
  Receive Dakota webhooks: register a target, verify the Ed25519
  signature, dedupe, and map events to the onboarding / deposit / payout
  flows. Use when wiring app/api/webhooks/dakota, when a KYB, deposit or
  payout status should update your app without polling, or when
  webhooks never arrive or fail signature checks.
---

# Dakota: Webhooks

Every async flow in this pack (KYB approval, ACH deposits, wallet
deposits, payouts) reports progress by webhook. Polling works for a
demo. Webhooks are how a real app learns that money moved.

## The chain (2 pieces)

1. **Register a target** once per environment (`POST /webhooks/targets`).
2. **Receive and verify** each delivery in a server route.

```ts
// One-time setup (a script, or an admin route)
const target = await dakota.webhooks.createTarget({
  url: "https://your-app.com/api/webhooks/dakota", // HTTPS only
  global: true,           // every event type for your client
  // OR: global: false + event_types: [...]  (see the trap below)
});
```

## The #1 trap: `global` and `event_types`

`event_types` is an **exact-match delivery filter**. With
`global: false`, Dakota delivers only the types you list. An unlisted
type is never delivered and never retried, so there is nothing to
replay later.

A target with `global: false` and **no** `event_types` receives no
events at all. Set `global: true` while you build. Narrow to an
explicit list once you know what you consume.

```ts
await dakota.webhooks.createTarget({
  url: "https://your-app.com/api/webhooks/dakota",
  global: false,
  event_types: [
    "customer.kyb_status.created",
    "customer.kyb_status.updated",
    "transaction.one_off.updated",
    "transaction.auto.updated",
    "wallet.deposit",
    "wallet.transaction.updated",
    "deposit.held",
  ],
});
```

## Verify every delivery

Dakota signs with **Ed25519** (not HMAC) over `timestamp + body`.
Each environment has its own public key, a 64-character hex string,
listed at
[docs.dakota.xyz/documentation/webhooks](https://docs.dakota.xyz/documentation/webhooks#dakota-public-keys).
Put the one that matches your API key in `DAKOTA_WEBHOOK_PUBLIC_KEY`.
A sandbox key does not verify production deliveries.

| Header | Meaning |
|---|---|
| `X-Webhook-Signature` | Base64 Ed25519 signature over `timestamp + body` |
| `X-Webhook-Timestamp` | Unix seconds. Reject anything older than 5 minutes |
| `X-Dakota-Event-ID` | Same across retries. Use it to dedupe |
| `X-Dakota-Event-Type` | e.g. `transaction.one_off.updated` |
| `X-Dakota-Delivery-Attempt` | 1-10 |

Verify against the **raw body**. If your framework parses JSON first
and you re-serialize it, the bytes change and the signature fails.

### Next.js App Router route with the TS SDK

```ts
// app/api/webhooks/dakota/route.ts
import { WebhookHandler, WebhookEventType } from "@dakota-xyz/ts-sdk/webhook";

const handler = new WebhookHandler({
  publicKey: process.env.DAKOTA_WEBHOOK_PUBLIC_KEY!,
  // timestampTolerance: 300 (default, seconds)
  // idempotencyStore: defaults to in-memory. On serverless, pass a
  // durable store (Redis / DB) or set false and dedupe yourself.
});

handler.on(WebhookEventType.CustomerKybStatusUpdated, async (event) => {
  // event.data.object holds the resource state
});
handler.on("transaction.*", async (event) => {
  // one-off and auto transactions: read status from event.data.object
});
handler.onDefault(async (event) => {
  console.log("unhandled", event.type);
});

export async function POST(request: Request) {
  const body = await request.text(); // raw body, do NOT request.json()
  try {
    await handler.handleRequest(body, Object.fromEntries(request.headers));
    return new Response("ok", { status: 200 });
  } catch (err) {
    return new Response("invalid", { status: 400 });
  }
}
```

With Express, mount `express.raw({ type: "application/json" })` before
`handler.expressMiddleware()`. Without the SDK, call
`verifySignature(...)` from the same subpath, or use the Node, Python,
Go, Rust and Java samples in the docs. Go: the `webhook` package in
`github.com/dakota-xyz/go-sdk`.

## Delivery rules

- Return `2xx` within **20 seconds**. Do slow work after you respond
  (queue it).
- Failed deliveries retry 10 times over about 48 hours, then show as
  failed in the dashboard.
- **Order is not guaranteed.** Two events for one resource can arrive
  swapped, most often in sandbox. Each event carries the full resource
  state. Treat its `status` as the truth, and accept a "later" status
  that arrives before an "earlier" one.
- Payloads are at `event.data.object` (SDK 2.0+).

## Events per flow

| Flow (skill) | Events |
|---|---|
| KYB ([`dakota-customer-onboarding`](../dakota-customer-onboarding/SKILL.md)) | `customer.kyb_status.created` (first status), `customer.kyb_status.updated`, `customer.capability_status.updated`, `recipient.created` |
| ACH / wire deposit to an on-ramp ([`dakota-receive-usd`](../dakota-receive-usd/SKILL.md)) | `transaction.auto.created`, `transaction.auto.updated`, `deposit.held`, `deposit.returned` |
| USDC to a wallet ([`dakota-receive-usdc`](../dakota-receive-usdc/SKILL.md)) | `wallet.deposit` |
| Wallet send | `wallet.transaction.created`, `wallet.transaction.updated` (`status_code`) |
| One-off payout ([`dakota-send-payment`](../dakota-send-payment/SKILL.md)) | `transaction.one_off.created`, `transaction.one_off.updated` |

`customer.updated` is in the enum but is not emitted. The full list is
the `EventType` enum in the API reference.

## Local development

Dakota only sends to HTTPS. Expose your local server with a tunnel
(e.g. `ngrok http 3000`) and register the tunnel URL as a sandbox
target. For a quick look without code, register a
[webhook.site](https://webhook.site) URL.

## Debug and replay

```ts
const history = await dakota.webhooks.listEvents();     // delivery history
const evt = await dakota.webhooks.getEvent(eventId);
await dakota.webhooks.replayEvent(eventId);             // redeliver
```

`dakota.webhooks.updateTarget(id, { enabled: true })` re-enables a
disabled target and sends what queued while it was off.

## Common problems

| Symptom | Cause | Fix |
|---|---|---|
| No deliveries at all | `global: false` with no or missing `event_types` | Set `global: true`, or list every type you consume |
| Every signature fails | Body re-serialized, or wrong environment key | Verify the raw body; use the key for the API key's environment |
| `Invalid public key length` | Key is not the 64-char hex value | Copy the hex key from the docs table |
| Same event handled twice | Retries / replays | Dedupe on `X-Dakota-Event-ID` (the SDK does it per process) |

## Canonical reference

- Webhooks guide: [docs.dakota.xyz/documentation/webhooks](https://docs.dakota.xyz/documentation/webhooks)
- Webhook testing in sandbox: [docs.dakota.xyz/documentation/testing](https://docs.dakota.xyz/documentation/testing#webhook-testing)
- Without the SDK: `POST /webhooks/targets`, `GET /webhooks/targets`,
  `PATCH /webhooks/targets/{id}`, `GET /webhooks`,
  `POST /webhooks/events/{id}/replay`: see
  [docs.dakota.xyz/api-reference/webhooks/create-a-webhook-target](https://docs.dakota.xyz/api-reference/webhooks/create-a-webhook-target).
