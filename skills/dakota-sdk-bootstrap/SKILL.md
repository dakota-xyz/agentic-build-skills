---
name: dakota-sdk-bootstrap
description: |
  Install and configure the Dakota TypeScript SDK in a project, build the
  client singleton, and wire it for sandbox + production. Use when
  scaffolding a new Dakota integration or adding the SDK to an existing
  app. The full install + auth modes + retry policy reference lives in
  AGENTS.md inside the SDK repo and at docs.dakota.xyz — this skill is
  the pack-shaped summary.
---

# Dakota — SDK bootstrap

## Install

```bash
npm install @dakota-xyz/ts-sdk
```

Targets `^1.4.x` at time of writing.

## Get an API key

Dashboard → **API Keys** → **Create New API Key** → copy once (never
shown again). Sandbox dashboard at `platform.sandbox.dakota.xyz`,
production at `platform.dakota.xyz`. Full instructions:
[docs.dakota.xyz/documentation/authentication/api-keys-headers](https://docs.dakota.xyz/documentation/authentication/api-keys-headers).

Sandbox keys and production keys are completely separate — they live in
separate dashboards. There is no flag that switches one into the other.

## Client singleton

A singleton avoids re-issuing OAuth tokens on every request. One file
per project:

```ts
// lib/dakota.ts
import { DakotaClient, Environment } from "@dakota-xyz/ts-sdk";

let _client: DakotaClient | null = null;

export function getDakotaClient(): DakotaClient {
  if (!_client) {
    if (!process.env.DAKOTA_API_KEY) {
      throw new Error("DAKOTA_API_KEY is required");
    }
    _client = new DakotaClient({
      apiKey: process.env.DAKOTA_API_KEY,
      environment:
        process.env.DAKOTA_ENV === "production"
          ? Environment.Production
          : Environment.Sandbox,
    });
  }
  return _client;
}
```

Server-only — never construct the client in browser code.

## Required headers

The SDKs add these for you; if you're calling Dakota directly over
HTTP, every request needs them:

| Header | When | Value |
|---|---|---|
| `x-api-key` | every request | your API key |
| `Content-Type: application/json` | requests with a body | constant |
| `x-idempotency-key` | **POST requests only** | a unique UUID per logical operation |

The idempotency key is a UUID — generate one per logical operation
(not per retry). If a POST is retried with the same `x-idempotency-key`
and the same body, Dakota returns the cached response from the first
attempt instead of creating a duplicate. Do **not** send the header on
GET / PUT / PATCH / DELETE.

The TypeScript SDK auto-generates idempotency keys via
`automaticIdempotency: true` (default on). Override per call with
`{ idempotencyKey: 'your-uuid' }` if you want a stable key (e.g.
keyed to a user action so refresh-bashing is safe).

## `.env.local` template

```bash
# Required
DAKOTA_API_KEY=

# Optional — defaults to sandbox if unset or any non-"production" value
DAKOTA_ENV=sandbox

# Optional — set when wiring webhooks
DAKOTA_WEBHOOK_PUBLIC_KEY=
```

## Error handling

Catch `APIError` (HTTP non-2xx) and `TransportError` (network failure).
Always log `requestId` from `APIError` — Dakota support reads their
logs by it.

```ts
import { APIError, TransportError } from "@dakota-xyz/ts-sdk";

try {
  await dakota.customers.create({ ... });
} catch (err) {
  if (err instanceof APIError) {
    console.error(err.statusCode, err.code, err.message, err.requestId);
    if (err.retryable) {
      // safe to retry
    }
  } else if (err instanceof TransportError) {
    console.error("network", err.message, err.cause);
  } else throw err;
}
```

Full error reference: [docs.dakota.xyz/api-reference/errors](https://docs.dakota.xyz/api-reference/errors).

## What next

The next skill is
[`dakota-customer-onboarding`](../dakota-customer-onboarding/SKILL.md) —
nothing else works until you have a KYB-active customer.

## Canonical reference

This skill captures patterns. The truth source is the docs + the SDK's
own `AGENTS.md` reference (~700 lines covering every resource and
method):

- SDK list + install — [docs.dakota.xyz/documentation/sdks](https://docs.dakota.xyz/documentation/sdks)
- Authentication — [docs.dakota.xyz/documentation/authentication](https://docs.dakota.xyz/documentation/authentication)
- API key headers — [docs.dakota.xyz/documentation/authentication/api-keys-headers](https://docs.dakota.xyz/documentation/authentication/api-keys-headers)
- API errors — [docs.dakota.xyz/api-reference/errors](https://docs.dakota.xyz/api-reference/errors)
- Quickstart — [docs.dakota.xyz/documentation/try-it-now](https://docs.dakota.xyz/documentation/try-it-now)
- LLM-friendly index — [docs.dakota.xyz/llms.txt](https://docs.dakota.xyz/llms.txt)
- SDK reference (in the repo) — `dakota-ts-sdk/AGENTS.md`

Every method here maps 1:1 to a REST endpoint documented in the API
reference.
