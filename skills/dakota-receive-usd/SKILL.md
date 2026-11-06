---
name: dakota-receive-usd
description: |
  Provision an ACH / Fedwire on-ramp account so the customer can receive
  USD that Dakota auto-converts to USDC. Use when building any product
  where US-based payers send USD (invoices, payroll funding, fiat
  top-ups, customer payments). Pairs with dakota-receive-usdc for global
  payers.
---

# Dakota: Receive USD (ACH / Fedwire on-ramp)

An on-ramp account gives the customer US bank deposit details (routing
+ account number). When USD lands via ACH or wire, Dakota converts it
and sends USDC to a crypto destination you specify, typically the
customer's treasury wallet.

## The chain (3 calls)

```ts
// PRE-REQ: customer is KYB-active (see dakota-customer-onboarding)
// PRE-REQ: you have a wallet address to receive the converted USDC.
//          Either the customer's treasury wallet (recommended, see
//          dakota-receive-usdc) or any address you control.

// Sandbox accepts testnets only (ethereum-sepolia, base-sepolia,
// polygon-amoy, arbitrum-sepolia, optimism-sepolia, solana-devnet) or
// "evm". Production accepts mainnets only. The wrong one returns
// 400 `network "ethereum-mainnet" is not allowed in sandbox; use a testnet`.
const NETWORK =
  process.env.DAKOTA_ENV === "production" ? "ethereum-mainnet" : "ethereum-sepolia";

// 1. Use the auto-created treasury recipient (KYB creates one named after
//    the customer). DON'T create a new recipient here, you'll 409.
const recipients = await dakota.recipients.list(customer.id).toArray();
const treasury = recipients[0];

// 2. Crypto destination = where the converted USDC lands
const cryptoDestination = await dakota.destinations.create(treasury.id, {
  destination_type: "crypto",
  name: "Treasury USDC",
  crypto_address: wallet.address,    // your treasury wallet's address
  network_id: NETWORK,
});
// The create response is { id }.

// 3. On-ramp account = the USD-side bank account, linked to the crypto destination.
//    NOTE: customer_id is NOT a field in AccountCreateRequest, the platform
//    derives it server-side from the destination → recipient → customer chain.
const account = await dakota.accounts.create({
  account_type: "onramp",
  crypto_destination_id: cryptoDestination.id,   // ◀── required
  source_asset: "USD",
  destination_asset: "USDC",
  destination_network_id: NETWORK,               // must match the destination's network
  capabilities: ["us_bank_account"],             // ACH + Fedwire. ["ach"] for ACH only.
});

// account.id                              , Dakota account KSUID
// account.bank_account.aba_routing_number  ← share this with payers
// account.bank_account.account_number      ← share this with payers
// account.bank_account.bank_name           ← share this with payers
```

## The on-ramp **requires** a crypto_destination_id

This is non-obvious. On-ramp = USD coming in, USDC going out, so Dakota
has to know **where the USDC lands**. There is no on-ramp account
without a crypto destination. A fiat destination returns
400 *"Cannot create an onramp account for a non-crypto destination"*.
A `destination_network_id` that differs from the destination's
`network_id` returns 400 *"Destination network does not match
destination"*.

That's why the chain has THREE calls, not one. You can't shortcut it.

## Why point the crypto destination at the wallet (not a random address)

Two reasons:

1. **Custody.** USDC sent to a random address you don't control is
   lost. Always use an address whose private key you (or your KMS) hold.
2. **Treasury cohesion.** Pointing at the operator's treasury wallet
   means ACH inbound + on-chain USDC inbound both increment the same
   balance, which is what the dashboard / accounting layer expects.

The
[`dakota-receive-usdc`](../dakota-receive-usdc/SKILL.md) skill provisions
the wallet. Run that first, then pass `wallet.address` into the crypto
destination here.

## Field names that bite

| Field | Common wrong value | Right value |
|---|---|---|
| `capabilities` | omitted | `["us_bank_account"]` (ACH + Fedwire), or `["ach"]`, `["fedwire"]` |
| `rail` | required | Optional on an on-ramp. The inbound rails come from `capabilities`. If you set it, it must be in `capabilities` |
| `account_type` | `"USD"` / `"checking"` | `"onramp"` |
| `source_asset` | `"USDC"` | `"USD"`: source is what payers send |
| `destination_asset` | `"USD"` | `"USDC"`: destination is what you receive |
| `destination_network_id` | `"ethereum"` | `"ethereum-sepolia"` in sandbox, `"ethereum-mainnet"` in production |

`capabilities: ["us_bank_account"]` expands to ACH + Fedwire. A `rail`
of `"ach"`, `"fedwire"`, `"fednow"` or `"us_bank_account"` is stored
as `us_bank_account`.

## What the response gives you (and what to share)

Response shape (abridged):

```json
{
  "id": "3EaO...",
  "account_type": "onramp",
  "bank_account": {
    "bank_name": "<from response>",
    "aba_routing_number": "101019644",
    "account_number": "2239762231",
    "account_holder_name": "Acme Remote Inc",
    "account_type": "checking"
  }
}
```

These three values are what your operator hands to a payer:

- `bank_account.bank_name`
- `bank_account.aba_routing_number`
- `bank_account.account_number`

Always read the bank name from the response. Do not hardcode it.

In production these are real deposit details. In sandbox the banking
layer is mocked, so a real ACH to them goes nowhere. Use
`sandbox.simulateInbound` instead.

## Sandbox testing

```ts
// Simulate an inbound ACH credit
await dakota.sandbox.simulateInbound({
  simulation_id: `sim_dep_${Date.now()}`,
  type: "ach_inbound",          // or "wire_inbound"
  account_id: account.id,
  amount: "2.00",               // sandbox cap is $2 per request
  currency: "USD",
});
```

The USD deposit is mocked. The follow-on USDC transfer to your crypto
destination is a **real testnet transaction**, and webhooks fire as in
production. Over the cap returns
400 `amount X exceeds sandbox cap of 2; reduce the amount and retry`.

See [`dakota-sandbox-testing`](../dakota-sandbox-testing/SKILL.md).

## Sandbox vs production

| | Sandbox | Production |
|---|---|---|
| Routing / account | Not reachable by real banks | Real deposit details |
| ACH inbound | `sandbox.simulateInbound({type: 'ach_inbound'})` | Real ACH from payer's bank |
| Networks | Testnets only | Mainnets only |
| Amount cap | $2 / request | None |
| Time to settle | Instant on `success_immediate` scenario | 1-3 business days for ACH |
| USDC delivery | Real testnet transfer | Real mainnet transfer |

## Reference implementation

Idempotent ensure-onramp that:
- Reuses an existing on-ramp if one already exists for this customer
- Routes the crypto destination at the customer's wallet (use after `ensureTreasuryWallet`)

```ts
async function ensureOnramp(customer: { id: string }, wallet: { address: string }) {
  const dakota = getDakotaClient();
  const network =
    process.env.DAKOTA_ENV === "production" ? "ethereum-mainnet" : "ethereum-sepolia";

  // Look for an existing onramp for this customer. The list endpoint
  // accepts customer_id as a query filter; the account response itself
  // does NOT include customer_id, so client-side filtering won't work.
  const existing = await dakota.accounts
    .list({ account_type: "onramp", customer_id: customer.id })
    .toArray();
  if (existing.length > 0) return existing[0];

  // The treasury recipient is auto-created at KYB-active; just grab it
  const [treasury] = await dakota.recipients.list(customer.id).toArray();
  if (!treasury) {
    throw new Error("No auto-created recipient yet (production can take ~10 min)");
  }

  const cryptoDestination = await dakota.destinations.create(treasury.id, {
    destination_type: "crypto",
    name: "Treasury USDC",
    crypto_address: wallet.address,
    network_id: network,
  });

  return dakota.accounts.create({
    account_type: "onramp",
    crypto_destination_id: cryptoDestination.id,
    source_asset: "USD",
    destination_asset: "USDC",
    destination_network_id: network,
    capabilities: ["us_bank_account"],
  });
}
```

## Optional: earn a fee on deposits

`accounts.create` accepts a developer fee, your cut of each deposit:
`developer_fee_bps` (0-10000) or `developer_fee_fixed` (a decimal
string), never both. Sending both returns 400. See
[advanced flows](https://docs.dakota.xyz/documentation/advanced-flows#developer-fees).
SWIFT on-ramps are on the same page. Foreign-currency on-ramps (for
example EUR in) are not available in sandbox.

## What next

- [`dakota-send-payment`](../dakota-send-payment/SKILL.md), pay anyone out
  of the USDC that lands here.
- [`dakota-webhooks`](../dakota-webhooks/SKILL.md), get notified when a
  deposit arrives.
- [`dakota-sandbox-testing`](../dakota-sandbox-testing/SKILL.md), drive
  the inbound side without real money.

## Canonical reference

- Destinations + recipients: [docs.dakota.xyz/documentation/destinations-recipients](https://docs.dakota.xyz/documentation/destinations-recipients)
- Common flows (on-ramp included): [docs.dakota.xyz/documentation/common-flows](https://docs.dakota.xyz/documentation/common-flows)
- Sandbox restrictions (networks, amount cap): [docs.dakota.xyz/documentation/testing](https://docs.dakota.xyz/documentation/testing)
- Without the SDK: `GET /customers/{id}/recipients`, `POST /recipients/{id}/destinations`,
  `POST /accounts`: see [docs.dakota.xyz/api-reference/introduction](https://docs.dakota.xyz/api-reference/introduction).
