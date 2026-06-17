---
name: dakota-sandbox-testing
description: |
  All Dakota sandbox simulation paths in one place — inbound deposits,
  outbound returns, KYB approval, the amount cap, the indexing quirks,
  and how to actually drive a payout to completed. Use when integration
  tests stall, when a tx is stuck pending, or when "Provider Error 502"
  shows up out of nowhere.
---

# Dakota — Sandbox testing

The sandbox runs the same production code paths — same orchestration,
webhooks, event pipeline — but replaces external providers (Lead Bank,
Privy) with in-process mocks. This skill is the field guide to what
works, what doesn't, and what the error messages actually mean.

Base URL: `https://api.platform.sandbox.dakota.xyz`

## The simulation endpoints

All sandbox is gated by `IS_SANDBOX=true` on the tenant. Calling these
against a production key returns 403.

### Drive KYB to active

```ts
await dakota.sandbox.simulateOnboarding({
  type: "kyb_approve",
  applicant_id: customer.application_id, // NOT customer.id
  simulation_id: `sim_kyb_${customer.id}`,
});
```

Types: `kyb_approve | kyb_reject | kyb_info_request | kyc_approve |
kyc_reject | kyc_info_request | applicant_activate | applicant_suspend`.

Most of the time `kyb_approve` alone is enough — `applicant_activate` is
auto-folded in.

### Receive USD via ACH

```ts
await dakota.sandbox.simulateInbound({
  simulation_id: `sim_dep_${Date.now()}`,
  type: "ach_inbound",
  account_id: onramp.id,
  amount: "2.00",     // sandbox cap is $2
  currency: "USD",
});
```

`type: "wire_inbound"` does the same for Fedwire.

### Receive USDC on-chain (`crypto_inbound`)

```ts
await dakota.sandbox.simulateInbound({
  simulation_id: `sim_crypto_${Date.now()}`,
  type: "crypto_inbound",
  wallet_id: wallet.id,        // or wallet_address
  amount: "100.00",
  currency: "USDC",
});
```

**Tenant flag required:** `USE_MOCK_PRIVY=true`. Without it, the Provider
sandbox service rejects with `FailedPrecondition` and the platform
returns `502 provider sandbox service returned an error`.

The error message in `provider@v1.0.84/internal/sandbox/grpc_server.go`
spells it out:

> *"crypto-rail simulations require USE_MOCK_PRIVY=true; with real Privy,
> deposits arrive via real on-chain webhooks"*

If your tenant has it off, three paths:

1. Ask Dakota Support to flip the flag on your sandbox tenant.
2. Send real Sepolia USDC from a faucet-funded EOA to the wallet's
   address — Dakota's webhooks watch the testnet just like mainnet.
3. Route the on-ramp's crypto destination at the wallet's address and
   use `ach_inbound` simulation instead — works on every tenant.

### Mark an outbound as returned / failed / rejected

```ts
await dakota.sandbox.simulateInbound({
  simulation_id: `sim_ret_${Date.now()}`,
  type: "ach_outbound_returned",
  movement_id: tx.id,            // the transaction KSUID
  amount: "100.00",
  currency: "USD",
});
```

Other outbound types: `ach_outbound_failed`, `ach_outbound_rejected`,
`ach_outbound_returned`, `ach_reversal`, plus wire equivalents.

These let you test failure paths without waiting for real ACH returns.

### What's NOT in the dashboard menu

`ach_outbound_settled` exists in the API but **isn't exposed in
Dakota's own simulation UI**. The canonical sandbox completion path for
a one-off payout is *not* "force settle"; it's:

1. Create the payout (`transactions.create`) — returns `crypto_address`.
2. Send USDC from a wallet (yours, real Sepolia, or `sandbox.simulateInbound(crypto_inbound)` to the wallet first) to the
   `crypto_address`.
3. The platform detects the USDC arrival → fires ACH out → tx flips to
   `completed`.

See [`dakota-receive-usdc`](../dakota-receive-usdc/SKILL.md) for the
signed-intent wallet transaction shape.

## The hard limits

### $2 amount cap per simulation

```
400 Invalid Request: amount 5000 exceeds sandbox cap of 2; reduce the amount and retry
```

Applies to every `simulateInbound` regardless of type. Workaround: use
$1.50 / $2.00 for demos.

This is `enforceSandboxAmountCap` in the platform — it applies before
any DB lookups. You can stack multiple simulations to fake higher
volumes ("3 × $2 deposits = $6") but no single sim can exceed the cap.

### Recipient name uniqueness per customer

```
409 Recipient Conflict: A recipient with this name already exists
```

KYB auto-creates one recipient named after the customer. Don't fight
it; reuse it for treasury / on-ramp work, and pick distinct names for
actual payees.

### Customer / sandbox reset

**No customer DELETE endpoint exists.** Confirmed empirically and via
the OpenAPI spec — `/customers/{id}` only exposes GET. There's no
sandbox-wide reset either. Workarounds:

- Rotate `external_id` per test run so test customers are
  distinguishable.
- Filter your UI to only show customers your app provisioned (use a
  side-table keyed by Dakota customer id).
- Ask Dakota Support for a sandbox tenant reset.

## The known quirks

### `transactions.list({customer_id})` is unreliable

The customer-id filter on `transactions.list` returns 0 for some valid
customer ids whose transactions are visible in an unfiltered call:

```ts
// Returns 0 for some customers, even when txs exist:
await dakota.transactions.list({ customer_id }).toArray();

// Always works:
const all = await dakota.transactions.list().toArray();
const mine = all.filter(t => (t as any).customer_id === customer_id);
```

Workaround everywhere in your app until Dakota fixes the index. Not a
production issue — sandbox-only.

### Pending one-off transactions omit `amount`

`response.Amount` is only set when `SendAmount != nil` (post-quote).
Pending payouts come back without it. **Cache the submitted amount in
your own DB** at create time. See
[`dakota-send-payment`](../dakota-send-payment/SKILL.md#the-amount-field-gotcha).

### `ach_outbound_settled` rejects with `fiat_requires_account_id`

The platform handler at `internal/api/server_sandbox.go:184-193` (the
`AchOutboundSettled` case) extracts `movement_id` from the body but
never extracts `account_id`. The downstream Provider validator demands
both. Result: 400 `fiat_requires_account_id` even when you pass
`account_id` in the body (the platform drops it).

Don't fight this. Use the canonical path (`wallets.createTransaction` →
`crypto_address`) instead. `ach_outbound_settled` is not on Dakota's
public dashboard menu either.

### "Upstream authentication error"

```
502 Provider Error: Upstream authentication error (please wait a few
minutes, or contact support if urgent)
```

The Provider sandbox service is down or rotating credentials. Wait
2-5 minutes and retry. Persistent? File a support ticket — sandbox is
operationally maintained.

## Scenarios

Every `simulateInbound` accepts a `scenario` parameter (defaults to
`success_immediate`):

| Scenario | What it does | Valid for | Stateful |
|---|---|---|---|
| `success_immediate` | Settles instantly, webhooks fire in seconds | All | No |
| `success_delayed` | Settles after `delay_seconds` (default 30) | Inbound + outbound settled | No |
| `compliance_hold` | Pauses for compliance review; advance with `release`/`reject` | ACH/Wire inbound | Yes |
| `manual_review` | Pauses for manual review; advance with `approve`/`reject` | ACH/Wire inbound | Yes |
| `wrong_chain` | Crypto deposit on wrong network | `crypto_inbound` | No |
| `unsupported_token` | Unsupported token | `crypto_inbound` | No |
| `address_mismatch` | Wallet address mismatch | `crypto_inbound` | No |
| `partial_crypto` | Partial fill (requires `partial_amount`) | `crypto_inbound` | No |

For stateful scenarios, advance with:

```ts
// POST /sandbox/simulations/{id}/advance
await fetch(`${base}/sandbox/simulations/${simId}/advance`, {
  method: "POST",
  body: JSON.stringify({ action: "release" }),
});
```

## Inspecting simulations

```ts
// GET /sandbox/simulations/{id}
const sim = await dakota.sandbox.getSimulation(simulationId);
// sim.state — accepted | delivered | failed
// sim.awaiting_advance — true if stateful and paused
```

Useful when a webhook didn't fire and you want to know why.

## End-to-end test checklist

For a clean E2E that proves your integration works:

- [ ] Create customer → KYB active
- [ ] Provision wallet + on-ramp + off-ramp
- [ ] `sandbox.simulateInbound(ach_inbound, $2)` — accepts
- [ ] Create a payout (`transactions.create`, $1.50) — returns pending
- [ ] List transactions — confirm the payout is visible via unfiltered
      list + client filter
- [ ] Verify the amount renders from your cached value, not Dakota's
      (Dakota omits it on pending)

A reference implementation lives at
[neobank-nomad/scripts/e2e-neobank-flow.ts](https://github.com/dakota-xyz/neobank-nomad/blob/main/scripts/e2e-neobank-flow.ts)
that walks the entire chain and prints a results table.

## Production parity

| Behavior | Sandbox | Production |
|---|---|---|
| Webhook delivery + signing | Identical | Identical |
| Event emission + state machine | Identical | Identical |
| External API calls | In-process mocks | Real Lead Bank + Privy |
| Money movement | Simulated via `simulateInbound` | Real funds |
| KYB | Instant via `simulateOnboarding` | Real Sumsub / Persona |
| Compliance checks | Mock risk scoring | Real TRM screening |
| Float checks | Always sufficient (configurable) | Real treasury balance |

Where sandbox differs operationally from production, the docs label it
explicitly. The API surface and webhook payload shapes are identical —
that's the whole point.

## What next

- Pair every test you write here with one of the build skills —
  [`dakota-customer-onboarding`](../dakota-customer-onboarding/SKILL.md),
  [`dakota-receive-usdc`](../dakota-receive-usdc/SKILL.md), etc.
- For policy enforcement testing,
  [`dakota-compliance-policies`](../dakota-compliance-policies/SKILL.md).

## Canonical reference

- Testing overview — [docs.dakota.xyz/documentation/testing](https://docs.dakota.xyz/documentation/testing)
- Webhooks — [docs.dakota.xyz/documentation/webhooks](https://docs.dakota.xyz/documentation/webhooks)
- State lifecycles — [docs.dakota.xyz/api-reference/state-lifecycles](https://docs.dakota.xyz/api-reference/state-lifecycles)
- Without the SDK: `POST /sandbox/simulate/inbound`,
  `POST /sandbox/simulate/onboarding`, `GET /sandbox/simulations/{id}`,
  `POST /sandbox/simulations/{id}/advance` — see
  [docs.dakota.xyz/api-reference](https://docs.dakota.xyz/api-reference).
