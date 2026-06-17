---
name: dakota-customer-onboarding
description: |
  Create a Dakota customer and drive KYB to active. Use when scaffolding the
  "open account" step of any product on Dakota rails, when KYB is stuck in
  pending/under_review, or when downstream calls fail with
  "fiat_requires_account_id" or "customer is not active" because KYB never
  flipped.
---

# Dakota — Customer onboarding (KYB)

Every Dakota integration starts here. Nothing — no on-ramp, off-ramp,
wallet, recipient, destination, transaction — works until you have a
customer with `kyb_status === 'active'`.

## The chain (3 calls, ~1s in sandbox)

```ts
// 1. Create the customer
const customer = await dakota.customers.create({
  name: "Acme Remote Inc",        // shown to operators
  customer_type: "business",      // or "individual"
  external_id: "<your-system-id>" // optional — your stable id
});
// customer.id            — the Dakota customer KSUID
// customer.application_id — the KYB application KSUID  ◀── this matters
// customer.application_url — hosted onboarding link (for production)

// 2. Approve KYB in sandbox
await dakota.sandbox.simulateOnboarding({
  type: "kyb_approve",
  applicant_id: customer.application_id, // ◀── NOT customer.id
  simulation_id: `sim_kyb_${customer.id}`,
});

// 3. Poll until active. Sandbox usually flips on the first poll.
let final = customer;
for (let i = 0; i < 8; i++) {
  await new Promise(r => setTimeout(r, 300));
  final = await dakota.customers.get(customer.id);
  if (final.kyb_status === "active") break;
}
```

## The #1 trap

`sandbox.simulateOnboarding` expects an `applicant_id`. **That is the
`application_id`** from the create response, **NOT the customer.id**.
This is the most common bug in agentic Dakota integrations — the SDK type
is permissive, the field names look similar, and the error message
("application not found") doesn't point at it.

```ts
// ❌ Wrong — fails with "application not found"
await dakota.sandbox.simulateOnboarding({
  type: "kyb_approve",
  applicant_id: customer.id,
});

// ✅ Right
await dakota.sandbox.simulateOnboarding({
  type: "kyb_approve",
  applicant_id: customer.application_id,
});
```

## After KYB flips active

**Dakota auto-creates one recipient** named after the customer. This
exists so the customer can be a payee in their own right (treasury
sweeps, internal transfers).

```ts
const recipients = await dakota.recipients.list(customer.id).toArray();
// recipients.length === 1
// recipients[0].name === customer.name
```

If your code later does `recipients.create(customer.id, { name: customer.name })`
you'll get a **409 Recipient Conflict**. Either reuse the auto-created
recipient (for treasury / on-ramp crypto destination — see
[`dakota-receive-usdc`](../dakota-receive-usdc/SKILL.md) and
[`dakota-receive-usd`](../dakota-receive-usd/SKILL.md)) or pick a
distinct name for new recipients.

Recipient names are unique **per customer**, not globally.

## What `kyb_status` actually means

The valid `kyb_status` enum is
`active | pending | partner_review | rejected | frozen | auto_declined`
(verified against `openapi.public.yaml` `KybStatus` definition):

| Value | Means |
|---|---|
| `pending` | Application created, customer hasn't submitted yet |
| `partner_review` | Submitted, awaiting verification with the KYB provider |
| `active` | Cleared — you can create accounts, wallets, destinations, transactions |
| `rejected` | Failed verification |
| `auto_declined` | Auto-rejected by risk screening |
| `frozen` | Previously-active customer suspended |

`under_review` is NOT a valid `kyb_status` — it only appears in the
sibling `application_status` field (`pending | submitted | under_review |
approved | declined`), which tracks the application lifecycle. For
downstream gating use `kyb_status`.

## Sandbox vs production

| | Sandbox | Production |
|---|---|---|
| Time to active | ~1 second | Minutes to days (Sumsub / Persona) |
| How | `sandbox.simulateOnboarding({ type: 'kyb_approve' })` | Hosted flow at `application_url`; webhooks fire on state changes |
| Idempotency | `simulation_id` | The applicant flow is itself the idempotency boundary |
| `applicant_activate` | Auto-folded into `kyb_approve` for most cases | Real activation step the verification provider triggers |

For production, hand the operator the `application_url` from the create
response. They finish KYB through Dakota's hosted UI. Listen for
`customer.kyb_status.updated` webhooks (see [`dakota-sandbox-testing`](../dakota-sandbox-testing/SKILL.md)).

## Reference implementation

End-to-end onboarding flow that's idempotent — safe to call N times,
backfills missing pieces, never duplicates work:

```ts
async function ensureCustomer(opts: {
  displayName: string;
  externalId: string;
  existingCustomerId?: string;
}) {
  const dakota = getDakotaClient();

  // 1. Resolve or create
  let customer = opts.existingCustomerId
    ? await dakota.customers.get(opts.existingCustomerId)
    : await dakota.customers.create({
        name: opts.displayName,
        customer_type: "business",
        external_id: opts.externalId,
      });

  // 2. Drive to active (sandbox only)
  if (customer.kyb_status !== "active") {
    if (!customer.application_id) {
      throw new Error("No application on customer; production flow required");
    }
    await dakota.sandbox.simulateOnboarding({
      type: "kyb_approve",
      applicant_id: customer.application_id,
      simulation_id: `sim_kyb_${customer.id}`,
    });
    for (let i = 0; i < 8; i++) {
      await new Promise(r => setTimeout(r, 300));
      customer = await dakota.customers.get(customer.id);
      if (customer.kyb_status === "active") break;
    }
  }

  if (customer.kyb_status !== "active") {
    throw new Error(`KYB did not activate (status=${customer.kyb_status})`);
  }

  return customer;
}
```

## Common errors

| Error | Cause | Fix |
|---|---|---|
| `400 application not found` | Passed `customer.id` as `applicant_id` | Pass `customer.application_id` instead |
| `409 Recipient Conflict` (later) | Tried to create a recipient with the customer's name | Reuse the auto-created recipient or pick a distinct name |
| `400 customer is not active` (downstream) | Tried to create an account/wallet/destination on a non-active customer | Poll until `kyb_status === 'active'` first |

## What next

You now have a customer. Pick a receive rail:

- [`dakota-receive-usdc`](../dakota-receive-usdc/SKILL.md) — provision a Dakota wallet so customers can pay you in stablecoin on-chain.
- [`dakota-receive-usd`](../dakota-receive-usd/SKILL.md) — provision an ACH on-ramp so customers can pay you in USD that auto-converts to USDC.

Then [`dakota-send-payment`](../dakota-send-payment/SKILL.md) to pay anyone
out to their bank.

## Canonical reference

- Customer onboarding overview — [docs.dakota.xyz/documentation/customer-onboarding](https://docs.dakota.xyz/documentation/customer-onboarding)
- KYB / KYC state lifecycle — [docs.dakota.xyz/api-reference/state-lifecycles](https://docs.dakota.xyz/api-reference/state-lifecycles)
- Terminology — [docs.dakota.xyz/documentation/terminology](https://docs.dakota.xyz/documentation/terminology)
- Without the SDK: `POST /customers`, `POST /sandbox/simulate/onboarding`,
  `GET /customers/{id}` — see the API reference at
  [docs.dakota.xyz/api-reference](https://docs.dakota.xyz/api-reference).
