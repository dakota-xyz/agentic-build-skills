---
name: dakota-sandbox-testing
description: |
  All Dakota sandbox simulation paths in one place, inbound deposits,
  outbound returns, KYB approval, the testnet-only and $2 rules, the
  wallet faucet, and how to actually drive a payout to completed. Use
  when integration tests stall, when a tx is stuck pending, or when a
  sandbox call returns 400 for a value that works in production.
---

# Dakota: Sandbox testing

The sandbox runs **real crypto custody on testnets** and **mocks the
banking layer**. Wallets sign and broadcast real testnet transactions;
the policy engine runs unmodified. Fiat rails (ACH, Fedwire) are
mocked. Webhooks use the same pipeline, envelope and signing as
production.

| Flow | What is mocked | What is real |
|---|---|---|
| On-ramp | The USD deposit (`simulateInbound`) | The stablecoin transfer to your destination (testnet) |
| Off-ramp / one-off payout | The USD outbound to the bank | The on-chain deposit you send (testnet) |
| Swap | Nothing | Fully on-chain on testnets |
| KYB / KYC | Review (`simulateOnboarding`) |, |

Base URL: `https://api.platform.sandbox.dakota.xyz`. Dashboard:
`https://platform.sandbox.dakota.xyz`. The simulation endpoints work
only there; on the production base URL they return 403.

## The two hard rules

### 1. Testnets only

Object-create endpoints (`POST /customers/{id}/recipients`,
`POST /recipients/{id}/destinations`, `POST /accounts`,
`POST /transactions`) reject mainnet networks in sandbox and testnets
in production:

```
400 network "ethereum-mainnet" is not allowed in sandbox; use a testnet
```

| Environment | Accepted |
|---|---|
| Sandbox | `ethereum-sepolia`, `base-sepolia`, `polygon-amoy`, `arbitrum-sepolia`, `optimism-sepolia`, `solana-devnet`, `evm` |
| Production | `ethereum-mainnet`, `base-mainnet`, `polygon-mainnet`, `arbitrum-mainnet`, `optimism-mainnet`, `solana-mainnet`, `evm` |

Pick the network from an env var so one code path works in both.

### 2. $2 per request

```
400 Invalid Request: amount 5000 exceeds sandbox cap of 2; reduce the amount and retry
```

The cap applies to `POST /transactions`, every `simulateInbound`, and
the wallet faucet (422 there). USD, USDC and RD count 1:1. USDT is not
supported in sandbox yet. Use `"1.00"` / `"2.00"` in fixtures. You can
stack several requests to show higher volume.

## The simulation endpoints

### Drive KYB to active

```ts
await dakota.sandbox.simulateOnboarding({
  type: "kyb_approve",
  applicant_id: customer.application_id, // NOT customer.id
  simulation_id: `sim_kyb_${customer.id}`, // required
});
```

| `type` | Effect |
|---|---|
| `kyb_approve` | Fully approves the customer (business or individual). Creates the recipient, fires `customer.kyb_status.created` |
| `applicant_activate` | Same as `kyb_approve`. Call one, not both |
| `kyb_reject` / `applicant_suspend` | Application → `declined`. No webhook |
| `kyb_info_request` | Application → `request_for_information`. No webhook; poll `customers.get` |
| `kyc_approve` / `kyc_reject` / `kyc_info_request` | Application status only. To approve an individual, use `kyb_approve` |
| `poa_reject` | Rejects an individual's proof of address |

Only the API client that owns the customer can simulate. Your role
also needs the sandbox-simulations permission (else 403).

### Receive USD via ACH or wire

```ts
await dakota.sandbox.simulateInbound({
  simulation_id: `sim_dep_${Date.now()}`,
  type: "ach_inbound",       // or "fedwire_inbound", "fednow_inbound", "swift_inbound"
  account_id: onramp.id,     // the on-ramp account
  amount: "2.00",            // cap: $2
  currency: "USD",
});
```

`wire_inbound` is a legacy alias. `simulation_id` makes the call
replay-safe: the same id with the same params returns the first
response; conflicting params return 409.

### Fund a wallet (faucet)

`POST /sandbox/wallets/{wallet_id}/faucet` sends testnet **RD** on
Base Sepolia to an EVM wallet, up to $2 per call. It returns 202 with a
`simulation_id`. Poll `dakota.sandbox.getSimulation(id)` until
`state_position` is `completed`. The TS SDK has no method for it yet;
see the `fetch` sample in
[`dakota-receive-usdc`](../dakota-receive-usdc/SKILL.md#sandbox-testing).

| Limit | Response |
|---|---|
| More than $2 in one call | `422 sandbox-amount-cap-exceeded` |
| Wallet's daily allowance used | `429 sandbox-faucet-wallet-daily-limit` |
| Your account's daily allowance used | `429 sandbox-faucet-client-daily-limit` |
| Shared faucet balance low | `429 sandbox-faucet-float-floor` (contact Dakota) |

For testnet **USDC**, send it from a public faucet (e.g.
[Circle faucet](https://faucet.circle.com)), or route an on-ramp at the
wallet and simulate `ach_inbound`.

The docs say to fund wallets with real testnet transfers, not
simulated crypto deposits. The API still has `type: "crypto_inbound"`
(with `wallet_address`, not `wallet_id`), but where the sandbox does not
support it you get
`502 provider sandbox service returned an error`. Use real testnet
funds instead.

### Drive a payout to completed

The canonical path for a one-off payout:

1. `transactions.create` returns `crypto_address` and `send_amount`.
2. Send exactly `send_amount` of testnet USDC to `crypto_address`, in
   one transfer (a signed wallet transaction, or any testnet wallet).
3. Dakota detects the deposit, the mocked ACH fires, and the tx moves
   to `completed`. Watch `transaction.one_off.updated`.

### Outbound outcomes (returned / failed / rejected / settled)

```ts
await dakota.sandbox.simulateInbound({
  simulation_id: `sim_ret_${Date.now()}`,
  type: "ach_outbound_returned",
  account_id: offramp.id,          // required for every fiat type
  one_off_transaction_id: tx.id,   // movement_id is a legacy alias
  amount: "1.50",                  // cap: $2
  currency: "USD",
});
```

Types: `ach_outbound_settled | _failed | _rejected | _returned`,
`ach_reversal`, and the same set for `fedwire_*` and `swift_*`
(`wire_*` are legacy aliases). The spec defines `account_id` as the
off-ramp account that funded the one-off. The one-off response does
not return an account id, so for plain one-offs prefer the canonical
funding path above.

## Scenarios

`simulateInbound` takes a `scenario` (default `success_immediate`).
List them with `dakota.sandbox.listScenarios()`.

| Scenario | What it does | Valid for | Stateful |
|---|---|---|---|
| `success_immediate` | Settles instantly | All except `fednow_inbound` | No |
| `success_delayed` | Settles after `delay_seconds` (1-86400) | Inbound + outbound settled | No |
| `compliance_hold` | Pauses for compliance review; advance `release` / `reject` | `ach_inbound`, `fedwire_inbound`, `swift_inbound` | Yes |
| `manual_review` | Pauses for manual review; advance `approve` / `reject` | `ach_inbound`, `fedwire_inbound`, `swift_inbound` | Yes |
| `unconfirmed` | Crypto deposit seen, not confirmed; advance `confirm` / `expire` | `crypto_inbound` | Yes |
| `wrong_chain` / `unsupported_token` / `address_mismatch` | Crypto deposit edge cases | `crypto_inbound` | No |
| `partial_crypto` | Partial fill (requires `partial_amount`) | `crypto_inbound` | No |

Advance a stateful simulation:

```ts
await dakota.sandbox.advanceSimulation(simId, { action: "release" });
```

## Inspecting simulations

```ts
const sim = await dakota.sandbox.getSimulation(simulationId);
// sim.state           , "accepted" | "rejected" (was the request taken)
// sim.state_position  , progress, e.g. "completed" | "failed"
// sim.awaiting_advance, true if stateful and paused
// sim.callbacks[]     , webhook delivery history
```

Useful when a webhook didn't fire and you want to know why.

## Error-injection headers

Sandbox reads a few request headers (not in the OpenAPI spec) to force
error branches on any call:

| Header | Effect |
|---|---|
| `X-Sandbox-Scenario` | Preset failure, e.g. `insufficient_funds`, `compliance_block`, `invalid_account`, `network_timeout`, `kyb_rejected`, `invalid_swift` (list: `GET /sandbox/scenarios`) |
| `X-Sandbox-Error-Status` + `-Message` + `-Step` | Inject a custom error at a pipeline step |
| `X-Sandbox-Instant-Completion: true` | Async flows complete without an advance call |
| `X-Sandbox-Skip-Auto-Approval: true` | Keep KYB pending |

Address screening is mocked by pattern: an address that contains
`hack` or `scam`, or starts with `0xBAD0000`, screens high risk;
`review` or `manual` screens medium. See the testing docs for the full
table.

## Other things that bite

### Recipient name uniqueness per customer

```
409 Recipient Conflict: A recipient with this name already exists
```

KYB auto-creates one recipient named after the customer. Names compare
case-insensitively, trimmed. Reuse it for treasury / on-ramp work, and
pick distinct names for actual payees.

### Customer cleanup

`dakota.customers.delete(id)` (`DELETE /customers/{customer_id}`)
soft-deletes a customer. It returns 409 once the customer has accounts
or is a sub-client. There is no sandbox-wide reset. Rotate
`external_id` per test run and filter your UI to customers your app
created.

### Listing one-off transactions

`dakota.transactions.list({ customer_id })` returns that customer's
one-offs (SDK 3.x sends `transaction_type=one_off`). With raw HTTP,
`customer_id` alone selects the auto-account family; add
`transaction_type=one_off`.

### Webhooks arrive out of order

Sandbox transitions are sub-second, so two events for one resource can
swap. Treat the `status` in each event as the truth. See
[`dakota-webhooks`](../dakota-webhooks/SKILL.md).

## End-to-end test checklist

- [ ] Create customer → `kyb_approve` → `customers.get` shows `active`
- [ ] Provision wallet (with a policy) + on-ramp on `ethereum-sepolia`
- [ ] `simulateInbound(ach_inbound, "2.00")` → USDC lands in the wallet
      (testnet transfer, `transaction.auto.*` + `wallet.deposit` webhooks)
- [ ] Create a payout (`transactions.create`, `"1.50"`) → `pending`
- [ ] Sign a wallet send of `send_amount` to `crypto_address` →
      payout reaches `completed`
- [ ] `transactions.list({ customer_id })` shows the payout

## Production parity

| Behavior | Sandbox | Production |
|---|---|---|
| Webhook delivery + signing | Identical (sandbox key) | Identical (production key) |
| Event emission + state machine | Identical | Identical |
| Crypto custody + policy engine | Real, on testnets | Real, on mainnets |
| Banking rails | Mocked | Real |
| KYB | `simulateOnboarding` | Real compliance review |
| Address screening | Mock risk patterns | Real screening |
| Amounts | $2 per request | No cap |

## What next

- Pair every test you write here with one of the build skills,
  [`dakota-customer-onboarding`](../dakota-customer-onboarding/SKILL.md),
  [`dakota-receive-usdc`](../dakota-receive-usdc/SKILL.md), etc.
- For policy enforcement testing,
  [`dakota-compliance-policies`](../dakota-compliance-policies/SKILL.md).

## Canonical reference

- Testing overview: [docs.dakota.xyz/documentation/testing](https://docs.dakota.xyz/documentation/testing)
- Webhooks: [docs.dakota.xyz/documentation/webhooks](https://docs.dakota.xyz/documentation/webhooks)
- State lifecycles: [docs.dakota.xyz/api-reference/state-lifecycles](https://docs.dakota.xyz/api-reference/state-lifecycles)
- Without the SDK: `POST /sandbox/simulate/inbound`,
  `POST /sandbox/simulate/onboarding`, `POST /sandbox/wallets/{id}/faucet`,
  `GET /sandbox/simulations/{id}`, `POST /sandbox/simulations/{id}/advance`,
  `GET /sandbox/scenarios`: see
  [docs.dakota.xyz/api-reference/introduction](https://docs.dakota.xyz/api-reference/introduction).
