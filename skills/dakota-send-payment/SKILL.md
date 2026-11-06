---
name: dakota-send-payment
description: |
  Send a USDC → USD ACH payment to any US bank account. Use when building
  payouts to vendors, contractors, employees, marketplace sellers, or any
  outbound bank transfer. Covers single payouts and batch payroll.
---

# Dakota: Send payment (USDC → USD ACH)

This is the off-ramp side. The operator spends from their USDC treasury;
Dakota fires a USD ACH credit to the recipient's bank. Same pattern works
for vendor invoices, contractor payroll, marketplace seller payouts,
the only thing that changes is the recipient's bank details.

## The chain (3 calls)

```ts
// PRE-REQ: customer is KYB-active.
const sandbox = process.env.DAKOTA_ENV !== "production";

// 1. Recipient = the payee (person or business)
const recipient = await dakota.recipients.create(customer.id, {
  name: "Jane Worker",            // unique per customer; see gotcha below
  address: {
    street1: "456 Worker Ave",
    city: "Austin",
    region: "TX",                 // ISO 3166-2 subdivision code, not "Texas"
    postal_code: "73301",
    country: "US",
  },
});

// 2. Fiat US destination = their bank account
const destination = await dakota.destinations.create(recipient.id, {
  destination_type: "fiat_us",
  name: "Jane Chase Checking",
  bank_name: "JPMorgan Chase",
  account_holder_name: "Jane Worker",
  account_number: "000123456789",
  aba_routing_number: "021000021",
  account_type: "checking",          // or "savings"
});
const destinationId = destination.id; // create returns { id }

// 3. Send a payment (a one-off transaction; no off-ramp account needed)
const tx = await dakota.transactions.create({
  customer_id: customer.id,
  source_asset: "USDC",
  source_network_id: sandbox ? "ethereum-sepolia" : "ethereum-mainnet",
  destination_id: destinationId,
  destination_asset: "USD",
  destination_payment_rail: "ach",
  amount: sandbox ? "1.50" : "1500.00",   // sandbox cap: $2 per request
  payment_reference: "Invoice 2026 001",  // ACH: 1-18 letters, digits, spaces
});
// tx.id             , transaction KSUID, save this
// tx.status         , "pending" initially
// tx.crypto_address , send USDC here to FUND this transaction
// tx.send_amount    , the exact USDC amount to send there
```

Steps 1-2 are one-time per payee (or per bank change). Step 3 is the
recurring "send a payment" call.

## Field traps

| Field | Wrong | Right |
|---|---|---|
| `name` (recipient) | Customer's own name (`customer.name`) | Anything else, uniqueness is per-customer, and Dakota already auto-created a recipient with the customer's name |
| `region` (recipient address) | `"Texas"` | `"TX"` (ISO 3166-2 subdivision) |
| `destination_type` (destination) | `"bank"` / `"ach"` | `"fiat_us"` for US bank, `"fiat_iban"` for IBAN / SWIFT |
| `account_type` (destination) | `"CHECKING"` | `"checking"` (lowercase) |
| `aba_routing_number` | `"routing_number"` | The field name has the `aba_` prefix |
| `destination_payment_rail` (transaction) | `"us_bank_account"` / `"wire"` | `"ach"`, `"fedwire"`, `"fednow"` or `"swift"`. Optional: omit it and Dakota picks the rail |
| `payment_reference` (ACH) | `"Invoice 2026-001"` | Letters, digits and spaces only, max 18 chars. A hyphen returns 400 |
| `source_network_id` | `"ethereum-mainnet"` in sandbox | Sandbox accepts testnets only (`ethereum-sepolia`, `base-sepolia`, ...). Production accepts mainnets only |
| `amount` | `"1500.00"` in sandbox | Max `"2.00"` per request in sandbox |

## Recipient name uniqueness

Recipient names are **unique per customer**. Dakota ignores case and
leading or trailing spaces, so "Jane Worker" and "jane worker " collide
(409 Recipient Conflict). Two different customers can each have a
"Jane Worker". The recipient that KYB approval creates with the
customer's name also counts.

The API also takes an optional `external_id` on a recipient (your own
key, not unique) and filters on it:
`GET /customers/{id}/recipients?external_id=...`. The TS SDK 3.x types
do not include it yet, so use raw HTTP for it. If display names repeat,
suffix them with something stable:

```ts
const name = `${displayName} (${employeeId})`;
```

## Funding the payment

`transactions.create` returns immediately with `status: 'pending'` and
a `crypto_address`. Dakota pays out the USD after the USDC arrives
there.

1. You call `transactions.create`.
2. You send **exactly `tx.send_amount`** of USDC to `tx.crypto_address`,
   in **one** transfer. The address takes one deposit only. In
   production this is a signed `wallets.createTransaction` from your
   treasury wallet (see [`dakota-receive-usdc`](../dakota-receive-usdc/SKILL.md)).
3. Dakota detects the USDC on-chain.
4. Dakota fires the ACH outbound to the recipient's bank.
5. ACH settles in 1-3 business days. `transaction.one_off.updated`
   webhooks report each status (see [`dakota-webhooks`](../dakota-webhooks/SKILL.md)).

Status values: `pending | processing | completed | failed | cancelled |
reversed | pending_return | returned | pending_reversal`.

In sandbox the same flow works on testnets: the on-chain leg is a real
testnet transfer and the ACH leg is mocked. See
[`dakota-sandbox-testing`](../dakota-sandbox-testing/SKILL.md).

## Amounts in the response

`amount` and `send_amount` in the response are the **deposit to send,
in the source asset** (USDC here). They are present when you sent
`amount` and the pair is at par (for example USDC → USD). They are
absent when you omit `amount`, pay out a non-USD currency (FX), or send
a USD amount below one cent.

The response does not echo the payout amount you asked for. Webhooks
carry it as `destination_amount`. If your UI lists payouts, store the
requested amount at create time, keyed by `tx.id`:

```ts
await db.paymentAmounts.upsert({
  dakota_transaction_id: tx.id,
  amount,
  currency: "USD",
});

const txs = await dakota.transactions.list({ customer_id: customer.id }).toArray();
const myAmounts = await db.paymentAmounts.findByTxIds(txs.map(t => t.id));
const merged = txs.map(t => ({
  ...t,
  amount: myAmounts[t.id] ?? t.amount ?? "0",
}));
```

## Batch payroll

For multiple payouts in one go, run `transactions.create` per row. Two
patterns:

```ts
// Sequential (deterministic, easier to debug)
for (const item of payments) {
  try {
    await sendOne(item);
  } catch (e) {
    results.push({ id: item.id, ok: false, error: e.message });
  }
}

// Parallel (faster, hammers the rate limit)
const results = await Promise.allSettled(payments.map(sendOne));
```

Pick sequential for batches < 50. Parallel is fine above that **if** the
SDK's retry policy is configured (`retryPolicy: { maxAttempts: 5 }`;
the default is 3). Dakota rate-limits per API key. Read
`X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` and
`Retry-After`.

## Listing past transactions

```ts
const txs = await dakota.transactions
  .list({ customer_id: customer.id })
  .toArray();
```

The TS SDK (3.x) asks for one-off transactions by default. With raw
HTTP, send `GET /transactions?transaction_type=one_off&customer_id=...`.
`customer_id` alone selects the auto-account (on-ramp / off-ramp)
family, so one-off payouts do not appear.

## Optional: a reusable off-ramp account

A one-off needs no account. If you want a standing USDC → USD pipe to
one bank destination (every deposit to its address pays out), create an
off-ramp account with `account_type: "offramp"`,
`fiat_destination_id`, `source_asset`, `source_network_id`,
`destination_asset` and `rail` (`ach`, `fedwire`, `fednow` or `swift`).
Its deposits report as `transaction.auto.*`, not one-off events. See
[common flows](https://docs.dakota.xyz/documentation/common-flows).

Both one-offs and accounts accept a developer fee: `developer_fee_bps`
or `developer_fee_fixed`, never both. See
[advanced flows](https://docs.dakota.xyz/documentation/advanced-flows#developer-fees).

## International payouts

For SWIFT and FX payouts (a `fiat_iban` destination,
`destination_payment_rail: "swift"`, quotes), see
[`dakota-international-payout`](../dakota-international-payout/SKILL.md).

## Reference implementation

Single payout:

```ts
async function payRecipient(opts: {
  customerId: string;
  recipientId: string;
  amount: string;
  reference: string; // ACH: 1-18 letters, digits, spaces
}) {
  const dakota = getDakotaClient();
  const sandbox = process.env.DAKOTA_ENV !== "production";
  const destinations = await dakota.destinations.list(opts.recipientId).toArray();
  const bank = destinations.find(d => d.destination_type === "fiat_us");
  if (!bank) throw new Error("Recipient has no fiat_us destination");

  const tx = await dakota.transactions.create({
    customer_id: opts.customerId,
    source_asset: "USDC",
    source_network_id: sandbox ? "ethereum-sepolia" : "ethereum-mainnet",
    destination_id: bank.destination_id, // list rows use destination_id
    destination_asset: "USD",
    destination_payment_rail: "ach",
    amount: opts.amount,
    payment_reference: opts.reference,
  });

  await db.paymentAmounts.upsert({
    dakota_transaction_id: tx.id,
    amount: opts.amount,
    currency: "USD",
  });

  return tx; // fund it: send tx.send_amount USDC to tx.crypto_address
}
```

## What next

- [`dakota-sandbox-testing`](../dakota-sandbox-testing/SKILL.md), how to
  watch a sandbox payout move from pending → completed.
- [`dakota-compliance-policies`](../dakota-compliance-policies/SKILL.md):
  guard the wallet transfer that funds a payout.

## Canonical reference

- Common flows (off-ramp included): [docs.dakota.xyz/documentation/common-flows](https://docs.dakota.xyz/documentation/common-flows)
- Advanced flows (SWIFT, developer fees): [docs.dakota.xyz/documentation/advanced-flows](https://docs.dakota.xyz/documentation/advanced-flows)
- Destinations + recipients: [docs.dakota.xyz/documentation/destinations-recipients](https://docs.dakota.xyz/documentation/destinations-recipients)
- Transaction state lifecycle: [docs.dakota.xyz/api-reference/state-lifecycles](https://docs.dakota.xyz/api-reference/state-lifecycles)
- Without the SDK: `POST /customers/{id}/recipients`, `POST /recipients/{id}/destinations`,
  `POST /transactions`, `GET /transactions`: see
  [docs.dakota.xyz/api-reference/introduction](https://docs.dakota.xyz/api-reference/introduction).
