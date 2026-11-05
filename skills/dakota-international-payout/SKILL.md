---
name: dakota-international-payout
description: |
  Send an international wire (SWIFT) from stablecoin, in USD or in a
  foreign currency priced by an FX quote (EUR, GBP, CAD, MXN, JPY and
  more). Use when paying recipients outside US domestic rails, when an
  app needs "pay this IBAN", or when a SWIFT or FX call returns 403
  capability-unavailable or "FX access is not enabled".
---

# Dakota: International payout (SWIFT + FX)

SWIFT is Dakota's international wire rail. A payout can settle in USD,
or in one of 17 foreign currencies priced by an FX quote. The shape is
the same as [`dakota-send-payment`](../dakota-send-payment/SKILL.md):
recipient → destination → one-off → fund it. What changes is the
destination type, the rail, and (for FX) a quote before you fund.

## Before you build: check availability

SWIFT depends on where the customer and your organization are based,
and on the customer's `international_wire` capability.

```ts
// 1. Rail readiness
const { capabilities } = await dakota.customers.getCapabilities(customer.id);
const swift = capabilities.find(c => c.capability === "international_wire");
// swift?.status, and swift.requirements[] if action is needed

// 2. Foreign currencies (no SDK method yet: raw HTTP)
const base = process.env.DAKOTA_ENV === "production"
  ? "https://api.platform.dakota.xyz"
  : "https://api.platform.sandbox.dakota.xyz";
const res = await fetch(
  `${base}/customers/${customer.id}/fx-currencies?direction=outgoing`,
  { headers: { "x-api-key": process.env.DAKOTA_API_KEY! } },
);
// 200 → { rail: "swift", direction, currencies: [{ currency, minor_units,
//         fixed_enabled, indicative_enabled }] }
// 403 capability-unavailable → SWIFT is not available to this customer
// 503 eligibility-unavailable → retry with backoff
```

An empty `currencies` list means no foreign currency right now. It
does not prove USD SWIFT is ready; check the capability too.

## The chain

```ts
const sandbox = process.env.DAKOTA_ENV !== "production";

// 1. Recipient WITH an address (required before any fiat destination)
const recipient = await dakota.recipients.create(customer.id, {
  name: "Acme GmbH",
  address: { street1: "Mainzer Landstrasse 11", city: "Frankfurt", country: "DE" },
});

// 2. IBAN destination with SWIFT
const destination = await dakota.destinations.create(recipient.id, {
  destination_type: "fiat_iban",
  name: "Acme EUR account",
  iban: "DE89370400440532013000",
  bic: "COBADEFFXXX",                     // required for SWIFT
  account_holder_name: "Acme GmbH",       // 3-140 chars
  account_holder_address: { street1: "Mainzer Landstrasse 11", city: "Frankfurt", country: "DE" },
  bank_name: "Commerzbank",
  bank_address: { street1: "Kaiserplatz 1", city: "Frankfurt", country: "DE" }, // required with bic
  assets: ["USD", "EUR"],                 // foreign currencies need FX access
  capabilities: ["swift"],
});

// 3. One-off payout. For FX, leave out `amount`: the quote sizes it.
const tx = await dakota.transactions.create({
  customer_id: customer.id,
  destination_id: destination.id,
  source_asset: "RD",                     // or "USDC"
  source_network_id: sandbox ? "base-sepolia" : "base-mainnet",
  destination_asset: "EUR",               // or "USD" for a USD wire
  destination_payment_rail: "swift",
});
// tx.id, tx.crypto_address , fund this address after you quote
```

For a USD SWIFT wire, set `destination_asset: "USD"` and `amount`, then
fund `tx.send_amount` exactly as in `dakota-send-payment`. No quote is
needed.

## FX: quote, then fund

There is no SDK method for quotes yet. Call `POST /fx/quotes` with the
transaction id:

```ts
const quote = await fetch(`${base}/fx/quotes`, {
  method: "POST",
  headers: {
    "x-api-key": process.env.DAKOTA_API_KEY!,
    "x-idempotency-key": crypto.randomUUID(), // reuse it on retries
    "content-type": "application/json",
  },
  body: JSON.stringify({
    direction: "outgoing",
    mode: "indicative",          // "fixed" only where fixed_enabled
    source_asset: "RD",
    destination_asset: "EUR",
    amount: "1",
    amount_side: "source",       // "destination" to target a EUR amount
    transaction_id: tx.id,
  }),
}).then(r => r.json());
```

| Field | Use |
|---|---|
| `funding_amount` | The token amount to send to `tx.crypto_address` (includes your developer fee) |
| `target_amount` | Quoted payout; an indicative amount can change |
| `status`, `mode` | `estimated` = not locked. Fund a fixed quote only once it is `issued` |
| `expires_at` | Fixed quotes: on-chain inclusion must land before it |
| `eligible_networks` | CAIP-2 ids (`eip155:84532` = Base Sepolia, `eip155:8453` = Base) |

Then send **exactly `funding_amount`** of the quoted asset to
`tx.crypto_address` on the matching network, in one transfer (a signed
wallet transaction, see
[`dakota-receive-usdc`](../dakota-receive-usdc/SKILL.md#wallet-transactions-sending)).
There is no separate "execute" call: the deposit funds the payout.

Track it with `transaction.one_off.updated` webhooks or
`transactions.get(tx.id)`. Reconcile on `fx_pricing.state: "final"`,
not on the quote.

## Traps

| Trap | What happens | Fix |
|---|---|---|
| Fund the target amount instead of `funding_amount` | Short or over deposit, held for review | Always send the quoted `funding_amount` |
| `destination_payment_rail: "sepa"` | SEPA is not offered as a payout rail | Use `"swift"` |
| A currency not in `fx-currencies` (e.g. THB, INR, CNY) | Rejected even with FX access | Offer only listed currencies |
| Fund a fixed quote after `expires_at` | Priced at indicative market rate instead | Check expiry; re-quote |
| `payment_reference` on SWIFT | 5-140 chars, max 4 lines of 35; letters, digits, spaces and `/ - ? : ( ) . , ' +` | Hyphens are fine here (unlike ACH) |
| Change `rail` / routing on an account | `501`; immutable after creation | Create a new account |

**Bank fees come out of the wire.** The beneficiary's bank and any
intermediary deduct their fees, so the beneficiary can receive less
than the payout. Dakota is not told about these deductions. If the
recipient must receive an exact amount, allow for them.

## Sandbox

- FX access is on for every sandbox client (it can take about a minute
  after the client is created; until then FX payouts return
  `400 FX access is not enabled`).
- Live market rates with a 0.50% spread, same 17 currencies, outgoing
  only. Payouts progress to completion on their own.
- The deposit that funds an FX payout is capped at $2 USD-equivalent,
  so each sandbox FX payout is worth about $2 in its currency.
- Fund with testnet RD from the wallet faucet
  (`POST /sandbox/wallets/{id}/faucet`, Base Sepolia). See
  [`dakota-sandbox-testing`](../dakota-sandbox-testing/SKILL.md).
- Foreign-currency on-ramps (e.g. EUR in by SWIFT) are not available in
  sandbox.

In production, FX is off by default and enabled per client on request.
Contact Dakota support before you go live with FX.

## Reusable SWIFT off-ramp

For repeated payouts to one IBAN, create an off-ramp account with
`account_type: "offramp"`, `fiat_destination_id`, `source_asset`,
`source_network_id`, `destination_asset` and `rail: "swift"`. Each
deposit to its `source_crypto_address` sends a wire. Those deposits
report as `transaction.auto.*` events. Fixed quotes are one-off only.

## Canonical reference

- Foreign exchange (FX): [docs.dakota.xyz/documentation/fx](https://docs.dakota.xyz/documentation/fx)
- SWIFT payouts: [docs.dakota.xyz/documentation/advanced-flows](https://docs.dakota.xyz/documentation/advanced-flows#swift-payouts-international-wire)
- IBAN destinations: [docs.dakota.xyz/documentation/destinations-recipients](https://docs.dakota.xyz/documentation/destinations-recipients#creating-iban-destinations)
- Sandbox FX access: [docs.dakota.xyz/documentation/testing](https://docs.dakota.xyz/documentation/testing#fx-access)
- Without the SDK: `GET /customers/{id}/capabilities`, `GET /customers/{id}/fx-currencies`,
  `POST /fx/quotes`, `GET /fx/quotes/{id}`, `POST /transactions`: see
  [docs.dakota.xyz/api-reference/introduction](https://docs.dakota.xyz/api-reference/introduction).
