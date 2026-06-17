---
name: dakota-send-payment
description: |
  Send a USDC → USD ACH payment to any US bank account. Use when building
  payouts to vendors, contractors, employees, marketplace sellers, or any
  outbound bank transfer. Covers single payouts and batch payroll.
---

# Dakota — Send payment (USDC → USD ACH)

This is the off-ramp side. The operator spends from their USDC treasury;
Dakota fires a USD ACH credit to the recipient's bank. Same pattern works
for vendor invoices, contractor payroll, marketplace seller payouts —
the only thing that changes is the recipient's bank details.

## The chain (4 calls)

```ts
// PRE-REQ: customer is KYB-active.

// 1. Recipient = the payee (person or business)
const recipient = await dakota.recipients.create(customer.id, {
  name: "Jane Worker",            // unique per customer; see gotcha below
  address: {
    street1: "456 Worker Ave",
    city: "Austin",
    region: "Texas",
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
const destinationId =
  (destination as any).destination_id ?? (destination as any).id;

// 3. Off-ramp account = the USDC → USD pipe
const offramp = await dakota.accounts.create({
  account_type: "offramp",
  fiat_destination_id: destinationId,
  source_asset: "USDC",
  source_network_id: "ethereum-mainnet",
  destination_asset: "USD",
  rail: "ach",
  capabilities: ["ach"],
});

// 4. Send a payment
const tx = await dakota.transactions.create({
  customer_id: customer.id,
  source_asset: "USDC",
  source_network_id: "ethereum-mainnet",
  destination_id: destinationId,
  destination_asset: "USD",
  destination_payment_rail: "ach",
  amount: "1500.00",
  payment_reference: "Invoice 2026-001",  // shows on the receiving bank's statement
});
// tx.id              — transaction KSUID, save this
// tx.status          — "pending" initially
// tx.crypto_address  — where USDC should be sent to FUND this transaction
```

Steps 1-3 are one-time per payee (or per address change). Step 4 is the
recurring "send a payment" call.

## Field traps

| Field | Wrong | Right |
|---|---|---|
| `name` (recipient) | Customer's own name (`customer.name`) | Anything else — uniqueness is per-customer, and Dakota already auto-created a recipient with the customer's name |
| `destination_type` (destination) | `"bank"` / `"ach"` | `"fiat_us"` for US bank, `"fiat_iban"` for IBAN |
| `account_type` (destination) | `"CHECKING"` | `"checking"` (lowercase) |
| `aba_routing_number` | `"routing_number"` | The field name has the `aba_` prefix |
| `rail` (offramp account) | `"us_bank_account"` | `"ach"` — note this is different from the on-ramp side |
| `destination_payment_rail` (transaction) | `"us_bank_account"` | `"ach"` |

The asymmetry between on-ramp `rail: "us_bank_account"` and off-ramp
`rail: "ach"` is intentional and easy to miss.

## Recipient name uniqueness

Recipient names are **unique per customer** (per `db/migrations/00002_recipients_destinations.up.sql`).
Two different customers can each have a "Jane Worker". One customer
can't have two. Auto-created treasury recipients also count.

If you're building a system that re-uses common names, suffix with
something stable (employee id, last 4 of routing):

```ts
const name = `${displayName} (${employeeId})`;
```

A timestamp suffix works too, but it makes deduplication awkward — every
re-run creates a new recipient.

## Sending the payment

`transactions.create` returns immediately with `status: 'pending'`. The
response includes a `crypto_address` field — that's where USDC needs to
be sent from to fund the transaction.

In production, the canonical flow is:
1. You call `transactions.create`.
2. You sign and post a `wallets.createTransaction` that sends `amount`
   of USDC from your treasury wallet to `tx.crypto_address`.
3. Dakota detects the USDC on-chain.
4. Dakota fires the ACH outbound to the recipient's bank.
5. ACH settles in 1-3 business days; webhooks notify you of state changes.

In sandbox the same flow works *if* you have a funded wallet. See
[`dakota-sandbox-testing`](../dakota-sandbox-testing/SKILL.md) for the
end-to-end including the wallet-tx signing path.

## The amount field gotcha

When `tx.status === 'pending'` and there's no `send_amount` yet (no USDC
has been quoted), **the API response omits the `amount` field**:

```json
{
  "id": "3Ea8YBuiTNIxK74RqrlbloJLzpM",
  "status": "pending",
  "source_asset": "USDC",
  "destination_asset": "USD",
  "destination_routing_number": "021000021",
  "destination_account_number_last_four": "6789"
  // NO amount. NO send_amount.
}
```

This is from the platform mapper (`internal/api/mappers/one_off_transactions.go`):
`response.Amount` is only set when `transaction.SendAmount != nil`.

**Cache the amount you submitted** when you call `transactions.create`,
keyed by `tx.id`. Otherwise your UI shows `$0` for pending payouts.

```ts
// At create time:
await db.paymentAmounts.upsert({
  dakota_transaction_id: tx.id,
  amount,
  currency: "USD",
});

// At list time:
const txs = await dakota.transactions.list().toArray();
const myAmounts = await db.paymentAmounts.findByTxIds(txs.map(t => t.id));
const merged = txs.map(t => ({
  ...t,
  amount: (t as any).amount ?? myAmounts[t.id] ?? "0",
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
SDK's retry policy is configured (`maxAttempts: 5+`, exponential
backoff). Dakota rate-limits per API key — read the response headers.

## Listing past transactions

```ts
const txs = await dakota.transactions
  .list({ customer_id: customer.id })  // ← can be unreliable in sandbox
  .toArray();
```

**Sandbox quirk:** `transactions.list({ customer_id })` returns empty
for some valid customer ids whose transactions are visible if you call
`list()` without the filter. Work around with an unfiltered list +
client-side filter:

```ts
const all = await dakota.transactions.list().toArray();
const mine = all.filter(t => (t as any).customer_id === customer.id);
```

This is a known sandbox indexing issue tracked in
[`dakota-sandbox-testing`](../dakota-sandbox-testing/SKILL.md). Production
isn't affected.

## Reference implementation

Single payout:

```ts
async function payRecipient(opts: {
  customerId: string;
  recipientId: string;
  amount: string;
  reference: string;
}) {
  const dakota = getDakotaClient();
  const destinations = await dakota.destinations.list(opts.recipientId).toArray();
  const bank = destinations.find(d => d.destination_type === "fiat_us");
  if (!bank) throw new Error("Recipient has no fiat_us destination");
  const destinationId = (bank as any).destination_id ?? (bank as any).id;

  const tx = await dakota.transactions.create({
    customer_id: opts.customerId,
    source_asset: "USDC",
    source_network_id: "ethereum-mainnet",
    destination_id: destinationId,
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

  return tx;
}
```

## What next

- [`dakota-sandbox-testing`](../dakota-sandbox-testing/SKILL.md) — how to
  watch a sandbox payout move from pending → completed.
- [`dakota-compliance-policies`](../dakota-compliance-policies/SKILL.md) —
  block payments above a threshold before they're submitted.

## Canonical reference

- Common flows (off-ramp included) — [docs.dakota.xyz/documentation/common-flows](https://docs.dakota.xyz/documentation/common-flows)
- Advanced flows (multi-rail, complex routing) — [docs.dakota.xyz/documentation/advanced-flows](https://docs.dakota.xyz/documentation/advanced-flows)
- Destinations + recipients — [docs.dakota.xyz/documentation/destinations-recipients](https://docs.dakota.xyz/documentation/destinations-recipients)
- Transaction state lifecycle — [docs.dakota.xyz/api-reference/state-lifecycles](https://docs.dakota.xyz/api-reference/state-lifecycles)
- Without the SDK: `POST /customers/{id}/recipients`, `POST /recipients/{id}/destinations`,
  `POST /accounts`, `POST /transactions`, `GET /transactions` — see
  [docs.dakota.xyz/api-reference](https://docs.dakota.xyz/api-reference).
