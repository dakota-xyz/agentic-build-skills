---
name: dakota-customer-onboarding
description: |
  Create a Dakota customer and drive KYB to active. Use when scaffolding the
  "open account" step of any product on Dakota rails, when KYB is stuck in
  pending, or when downstream calls fail with
  "Customer is not KYB-approved by Dakota" because KYB never flipped.
---

# Dakota: Customer onboarding (KYB)

Every Dakota integration starts here. Accounts (on-ramp, off-ramp,
swap), one-off transactions and wallet sends are refused until the
customer has `kyb_status === 'active'`. You can create recipients and
destinations before that.

## The chain (3 calls in sandbox)

```ts
// 1. Create the customer
const customer = await dakota.customers.create({
  name: "Acme Remote Inc",        // shown to operators, min 3 chars
  customer_type: "business",      // or "individual"
  external_id: "<your-system-id>" // optional, your stable id (not unique-enforced)
});
// customer.id                    , the Dakota customer KSUID
// customer.application_id        , the KYB application KSUID  ◀── this matters
// customer.application_url       , hosted onboarding link (for production)
// customer.application_expires_at, token expiry, NANOSECONDS since epoch (90 days)
// customer.kyb_links             , per-provider KYB links
// NOTE: the create response does NOT include kyb_status.
//       Read it with customers.get().

// 2. Approve KYB in sandbox
await dakota.sandbox.simulateOnboarding({
  type: "kyb_approve",
  applicant_id: customer.application_id, // ◀── NOT customer.id
  simulation_id: `sim_kyb_${customer.id}`, // required
});

// 3. Read the status. Provisioning runs inline, so it is often active
//    at once. If not, it arrives via customer.kyb_status.* webhooks.
let final = await dakota.customers.get(customer.id);
for (let i = 0; i < 20 && final.kyb_status !== "active"; i++) {
  await new Promise(r => setTimeout(r, 500));
  final = await dakota.customers.get(customer.id);
}
```

## The #1 trap

`sandbox.simulateOnboarding` expects an `applicant_id`. **That is the
`application_id`** from the create response, **NOT the customer.id**.
This is the most common bug in agentic Dakota integrations. Both are
KSUID strings, the field names look similar, and the error does not
point at the mix-up:

```ts
// ❌ Wrong, 404 "No onboarding application found with id <id>"
await dakota.sandbox.simulateOnboarding({
  type: "kyb_approve",
  applicant_id: customer.id,
  simulation_id: `sim_kyb_${customer.id}`,
});

// ✅ Right
await dakota.sandbox.simulateOnboarding({
  type: "kyb_approve",
  applicant_id: customer.application_id,   // ◀── from create() response
  simulation_id: `sim_kyb_${customer.id}`,
});
```

Only the API client that owns the customer can simulate its
onboarding. Another client's application also returns 404.

`kyb_approve` and `applicant_activate` run the same code path. Call
one of them, not both.

## After KYB flips active

**Dakota auto-creates one recipient** named after the customer, if the
customer has no recipient yet. This exists so the customer can be a
payee in their own right (treasury sweeps, on-ramp crypto destination).
In sandbox it exists right after `kyb_approve`. In production it can
take up to about 10 minutes.

```ts
const recipients = await dakota.recipients.list(customer.id).toArray();
// recipients.length === 1
// recipients[0].name === customer.name
```

If your code later does `recipients.create(customer.id, { name: customer.name })`
you'll get **409 Recipient Conflict: "A recipient with this name
already exists"**. Either reuse the auto-created recipient (for
treasury / on-ramp crypto destination, see
[`dakota-receive-usdc`](../dakota-receive-usdc/SKILL.md) and
[`dakota-receive-usd`](../dakota-receive-usd/SKILL.md)) or pick a
distinct name for new recipients.

Recipient names are unique **per customer**, not globally. The match
ignores case and leading/trailing spaces.

## What `kyb_status` actually means

The `kyb_status` enum is
`active | pending | partner_review | rejected | frozen | auto_declined`:

| Value | Means |
|---|---|
| `pending` | Dakota has no decision for the customer yet |
| `partner_review` | Dakota has a status, but the customer is not yet active, rejected or frozen |
| `active` | Approved. The customer can create accounts and move money |
| `rejected` | Rejected, including automatic declines |
| `frozen` | Frozen, cannot move money. A freeze during onboarding shows as `pending` until compliance decides |
| `auto_declined` | In the enum, but never returned. An automatic decline reads `rejected` |

The sibling `application_status` field tracks the application
lifecycle: `pending | submitted | under_review | request_for_information
| admin_revision | approved | declined | compliance_review | closed`
(plus the deprecated `completed`). For downstream gating use
`kyb_status`.

## Sandbox vs production

| | Sandbox | Production |
|---|---|---|
| Time to active | Seconds | Minutes to days (real compliance review) |
| How | `sandbox.simulateOnboarding({ type: 'kyb_approve' })` | Hosted flow at `application_url`; webhooks fire on state changes |
| Replay safety | Idempotency key (the SDK sends one). `simulation_id` is required and echoed back | The applicant flow is itself the idempotency boundary |

For production, hand the operator the `application_url` from the create
response. They finish KYB through Dakota's hosted UI. Listen for
`customer.kyb_status.created` (first status) and
`customer.kyb_status.updated` (later changes). See
[`dakota-webhooks`](../dakota-webhooks/SKILL.md).

To check which rails a customer can use and what still blocks each,
call `dakota.customers.getCapabilities(customer.id)`.

## Reference implementation

End-to-end onboarding flow that's idempotent, safe to call N times,
backfills missing pieces, never duplicates work:

```ts
async function ensureCustomer(opts: {
  displayName: string;
  externalId: string;
  existingCustomerId?: string;
  existingApplicationId?: string;
}) {
  const dakota = getDakotaClient();

  // 1. Resolve or create. create() carries application_id;
  //    get() carries kyb_status.
  let customerId = opts.existingCustomerId;
  let applicationId = opts.existingApplicationId;
  if (!customerId) {
    const created = await dakota.customers.create({
      name: opts.displayName,
      customer_type: "business",
      external_id: opts.externalId,
    });
    customerId = created.id;
    applicationId = created.application_id;
    // Persist customerId + applicationId in your DB here.
  }

  let customer = await dakota.customers.get(customerId);

  // 2. Drive to active (sandbox only)
  if (customer.kyb_status !== "active") {
    if (!applicationId) {
      throw new Error("No application id stored; production flow required");
    }
    await dakota.sandbox.simulateOnboarding({
      type: "kyb_approve",
      applicant_id: applicationId,
      simulation_id: `sim_kyb_${customerId}`,
    });
    for (let i = 0; i < 20 && customer.kyb_status !== "active"; i++) {
      await new Promise(r => setTimeout(r, 500));
      customer = await dakota.customers.get(customerId);
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
| `404 No onboarding application found with id <id>` | Passed `customer.id` as `applicant_id`, or the customer belongs to another API client | Pass `customer.application_id` instead |
| `400 applicant_id must be a valid KSUID` | Passed something that is not a KSUID | Pass `customer.application_id` |
| `409 Recipient Conflict` (later) | Tried to create a recipient with the customer's name | Reuse the auto-created recipient or pick a distinct name |
| `400 Customer is not KYB-approved by Dakota` (downstream) | Tried to create an account or transaction on a non-active customer | Wait until `kyb_status === 'active'` first |

## Other ways to onboard

- **Individuals (KYC):** `customer_type: "individual"`. Proof of Address
  rules differ, see
  [Proof of Address for individual customers](https://docs.dakota.xyz/documentation/customer-onboarding#proof-of-address-for-individual-customers).
  In sandbox, approve with `kyb_approve` too (`kyc_approve` sets only
  the application status).
- **Import customers already verified elsewhere:** Sumsub and Persona
  token import / sharing. See the docs pages listed below.

## What next

You now have a customer. Pick a receive rail:

- [`dakota-receive-usdc`](../dakota-receive-usdc/SKILL.md), provision a Dakota wallet so customers can pay you in stablecoin on-chain.
- [`dakota-receive-usd`](../dakota-receive-usd/SKILL.md), provision an ACH on-ramp so customers can pay you in USD that auto-converts to USDC.

Then [`dakota-send-payment`](../dakota-send-payment/SKILL.md) to pay anyone
out to their bank.

## Canonical reference

- Customer onboarding overview: [docs.dakota.xyz/documentation/customer-onboarding](https://docs.dakota.xyz/documentation/customer-onboarding)
- KYB / KYC state lifecycle: [docs.dakota.xyz/api-reference/state-lifecycles](https://docs.dakota.xyz/api-reference/state-lifecycles)
- Terminology: [docs.dakota.xyz/documentation/terminology](https://docs.dakota.xyz/documentation/terminology)
- Token import: [Sumsub](https://docs.dakota.xyz/documentation/sumsub-token-import), [Persona](https://docs.dakota.xyz/documentation/persona-token-import), [completing an imported application](https://docs.dakota.xyz/documentation/completing-an-imported-application)
- Without the SDK: `POST /customers`, `POST /sandbox/simulate/onboarding`,
  `GET /customers/{id}`, `GET /customers/{id}/capabilities`: see the API
  reference at
  [docs.dakota.xyz/api-reference/introduction](https://docs.dakota.xyz/api-reference/introduction).
