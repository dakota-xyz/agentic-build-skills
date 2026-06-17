---
name: dakota-receive-usd
description: |
  Provision an ACH on-ramp account so the customer can receive USD that
  Dakota auto-converts to USDC. Use when building any product where
  US-based payers send USD (invoices, payroll funding, fiat top-ups,
  customer payments). Pairs with dakota-receive-usdc for global payers.
---

# Dakota — Receive USD (ACH on-ramp)

An on-ramp account gives the customer a real ACH routing + account
number at Lead Bank (in sandbox) or whichever sponsor bank is on file
(in production). When USD lands via ACH or wire, Dakota mints the
equivalent USDC into a crypto destination you specify — typically the
customer's treasury wallet.

## The chain (3 calls)

```ts
// PRE-REQ: customer is KYB-active (see dakota-customer-onboarding)
// PRE-REQ: you have a wallet address to receive the converted USDC.
//          Either the customer's treasury wallet (recommended — see
//          dakota-receive-usdc) or any address you control.

// 1. Use the auto-created treasury recipient (KYB creates one named after
//    the customer). DON'T create a new recipient here — you'll 409.
const recipients = await dakota.recipients.list(customer.id).toArray();
const treasury = recipients[0];

// 2. Crypto destination = where the minted USDC lands
const cryptoDestination = await dakota.destinations.create(treasury.id, {
  destination_type: "crypto",
  name: "Treasury USDC",
  crypto_address: wallet.address,    // your treasury wallet's address
  network_id: "ethereum-mainnet",    // or "ethereum-sepolia" in sandbox
});

// 3. On-ramp account = the USD-side bank account, linked to the crypto destination.
//    NOTE: customer_id is NOT a field in AccountCreateRequest — the platform
//    derives it server-side from the destination → recipient → customer chain.
const account = await dakota.accounts.create({
  account_type: "onramp",
  crypto_destination_id: cryptoDestination.id,   // ◀── required
  source_asset: "USD",
  destination_asset: "USDC",
  destination_network_id: "ethereum-mainnet",
  rail: "us_bank_account",            // ◀── covers ACH + wire (see below)
  capabilities: ["ach"],              // payment capabilities to enable
});

// account.id              — Dakota account KSUID
// account.bank_account.aba_routing_number  ← share this with payers
// account.bank_account.account_number       ← share this with payers
// account.bank_account.bank_name            ← e.g. "Lead Bank" in sandbox
```

## The on-ramp **requires** a crypto_destination_id

This is non-obvious. On-ramp = USD coming in, USDC going out — so Dakota
has to know **where the USDC lands**. There is no on-ramp account
without a crypto destination. The platform enforces it
(`server_accounts.go:1143`: *"Cannot create an onramp account for a
non-crypto destination"*).

That's why the chain has THREE calls, not one. You can't shortcut it.

## Why point the crypto destination at the wallet (not a random address)

Two reasons:

1. **Custody.** USDC minted at a random address you don't control is
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
| `rail` | `"ach"` | `"us_bank_account"` — covers both ACH and Fedwire |
| `account_type` | `"USD"` / `"checking"` | `"onramp"` |
| `source_asset` | `"USDC"` | `"USD"` — source is what payers send |
| `destination_asset` | `"USD"` | `"USDC"` — destination is what you receive |
| `destination_network_id` | `"ethereum"` | `"ethereum-mainnet"` (or `"ethereum-sepolia"`) |
| `capabilities` | `["us_bank_account"]` | `["ach"]` |

`rail` is the on-ramp's settlement family. `capabilities` are the
inbound payment types enabled. Setting `rail: "us_bank_account"` +
`capabilities: ["ach"]` is the standard configuration.

## What the response gives you (and what to share)

Sandbox response shape (production looks identical, real bank):

```json
{
  "id": "3EaO...",
  "account_type": "onramp",
  "status": "active",
  "bank_account": {
    "bank_name": "Lead Bank",
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

They're real ACH details — a US payer can send a real ACH to them from
their own bank's bill-pay or via Plaid.

## Sandbox testing

```ts
// Simulate an inbound ACH credit
await dakota.sandbox.simulateInbound({
  simulation_id: `sim_dep_${Date.now()}`,
  type: "ach_inbound",
  account_id: onramp.id,
  amount: "2.00",      // sandbox cap is $2
  currency: "USD",
});
```

The simulation accepts. In a tenant with `USE_MOCK_PRIVY=true`, USDC
materialises at the configured crypto destination. With it off (the
default), the simulation is accepted but the USDC mint doesn't
complete — useful for showing the API surface, not for end-to-end
balance updates.

See [`dakota-sandbox-testing`](../dakota-sandbox-testing/SKILL.md).

## Sandbox vs production

| | Sandbox | Production |
|---|---|---|
| Bank | Lead Bank (test) | Lead Bank or other Dakota sponsor bank |
| Routing / account | Sandbox-specific, not real | Real Lead Bank routing + virtual account |
| ACH inbound | `sandbox.simulateInbound({type: 'ach_inbound'})` | Real ACH from payer's bank |
| Amount cap | $2 / simulation | None |
| Time to settle | Instant on `success_immediate` scenario | 1-3 business days for ACH |
| USDC mint | Requires `USE_MOCK_PRIVY=true` on tenant | Always real |

## Reference implementation

Idempotent ensure-onramp that:
- Reuses an existing on-ramp if one already exists for this customer
- Routes the crypto destination at the customer's wallet (use after `ensureTreasuryWallet`)

```ts
async function ensureOnramp(customer: Customer, wallet: { address: string }) {
  const dakota = getDakotaClient();

  // Look for an existing onramp for this customer. The list endpoint
  // accepts customer_id as a query filter; the account response itself
  // does NOT include customer_id, so client-side filtering won't work.
  const existing = await dakota.accounts
    .list({ account_type: "onramp", customer_id: customer.id })
    .toArray();
  if (existing.length > 0) return existing[0];

  // The treasury recipient is auto-created at KYB-active; just grab it
  const [treasury] = await dakota.recipients.list(customer.id).toArray();
  if (!treasury) throw new Error("Customer has no auto-created recipient");

  const cryptoDestination = await dakota.destinations.create(treasury.id, {
    destination_type: "crypto",
    name: "Treasury USDC",
    crypto_address: wallet.address,
    network_id: "ethereum-mainnet",
  });

  return dakota.accounts.create({
    account_type: "onramp",
    crypto_destination_id:
      (cryptoDestination as any).destination_id ?? (cryptoDestination as any).id,
    source_asset: "USD",
    destination_asset: "USDC",
    destination_network_id: "ethereum-mainnet",
    rail: "us_bank_account",
    capabilities: ["ach"],
  });
}
```

## What next

- [`dakota-send-payment`](../dakota-send-payment/SKILL.md) — pay anyone out
  of the USDC that lands here.
- [`dakota-sandbox-testing`](../dakota-sandbox-testing/SKILL.md) — drive
  the inbound side without real money.

## Canonical reference

- Destinations + recipients — [docs.dakota.xyz/documentation/destinations-recipients](https://docs.dakota.xyz/documentation/destinations-recipients)
- Common flows (on-ramp included) — [docs.dakota.xyz/documentation/common-flows](https://docs.dakota.xyz/documentation/common-flows)
- Without the SDK: `GET /customers/{id}/recipients`, `POST /recipients/{id}/destinations`,
  `POST /accounts` — see [docs.dakota.xyz/api-reference](https://docs.dakota.xyz/api-reference).
