---
name: dakota-compliance-policies
description: |
  Create Dakota compliance policies that block transactions before they
  settle — amount thresholds, address allow/deny lists, approval flows.
  Use when adding guardrails to a product on Dakota rails, when an
  operator needs "block any outbound > $X" controls, or when audit /
  compliance asks for proof of in-line enforcement.
---

# Dakota — Compliance policies

Policies are the platform layer that says "no, that money isn't moving."
Rules attach to policies; policies attach to wallets. Before a wallet
transaction executes, the policy engine evaluates the rules and either
allows or denies.

This is what makes Dakota a compliance product, not just a payments
product. *"You can build the neobank agentically. You can't build the
MTL agentically."*

## The chain (4 calls)

```ts
import { generateKeyPairSync } from "crypto";

// 1. ES256 signer keypair (same as wallets — see dakota-receive-usdc)
const { publicKey } = generateKeyPairSync("ec", {
  namedCurve: "P-256",
  publicKeyEncoding: { type: "spki", format: "der" },
  privateKeyEncoding: { type: "pkcs8", format: "der" },
});
const memberKey = Buffer.from(publicKey).toString("base64");

// 2. Register signer
await dakota.signers.create({
  name: "Compliance Operator",
  public_key: memberKey,
  key_type: "ES256",      // uppercase, required
});

// 3. Signer group (policies require one; rule mutations need endorsements
//    from members of this group in production)
const group = await dakota.signerGroups.create({
  name: "Compliance Operators",
  member_keys: [memberKey],
});

// 4. Policy with at least one rule
const policy = await dakota.policies.create({
  name: "High-value transaction review",
  description: "Block transactions ≥ $10,000 USD pending manual review",
  signer_group_id: group.id,    // required
  rules: [
    {
      rule_type: "amount_threshold",
      action: "deny",
      definition: {
        // Smallest currency unit. For USD that's CENTS — $10,000 = 1_000_000.
        min_amount: 1_000_000,
        // "Number of approvals required" per the platform domain type
        // (pkg/core/policy.go: AmountThresholdRuleDefinition.Threshold).
        // Set to 0 for a hard deny; higher values mean "this many
        // approvals can override".
        threshold: 0,
        // Asset object (NOT a `currency: "USD"` string — see the gotcha below).
        asset: { id: "USD", name: "US Dollar" },
      },
    },
  ],
});
```

## The schema gotcha

The OpenAPI spec types `definition` as `Record<string, never>` (a
quirk of `openapi-typescript`'s handling of free-form objects). The
real API has strict server-side requirements that the schema doesn't
surface. For `amount_threshold` rules, the platform validator at
`internal/api/server_policies.go` casts to:

```go
type AmountThresholdRuleDefinition struct {
    MinAmount int64 `json:"min_amount"` // Minimum amount (smallest currency unit)
    Threshold int32 `json:"threshold"`  // Number of approvals required
    Asset     Asset `json:"asset"`      // { id, name }
}
```

Three traps:

- `min_amount` — number (int64), in the **smallest currency unit**.
  Cents for USD. `min_amount: 10000` means $100.00, not $10,000.
- `threshold` — number (int32), **approvals required**. NOT the same
  as `min_amount`. Setting `threshold: 10000` is nonsensical (10,000
  approvers).
- `asset` — object `{ id, name }`. Not a `currency: "USD"` string.
  The OpenAPI example in the docs (`{ amount: "10000", currency: "USD" }`)
  is **stale and rejected** by the validator with
  `Invalid min_amount in amount threshold rule at index 0`.

What works in sandbox (verified empirically):

```ts
definition: {
  min_amount: 1_000_000,                     // $10,000 in cents
  threshold: 0,                              // 0 approvals = hard deny
  asset: { id: "USD", name: "US Dollar" },   // object, not string
}
```

## Why a signer group is required

Policies are the operator's compliance posture. Mutations to a policy
(`addRule`, `updateRule`, `removeRule`, `delete`) are sensitive — they
literally change what money can move. Dakota requires every policy to
be tied to a signer group so those mutations can be endorsed by
multi-party signatures in production.

In sandbox you can mutate without endorsement, but the group still has
to exist. Create one signer + one group at app boot and reuse them for
every policy.

## Rule types

| `rule_type` | Definition shape | Use case |
|---|---|---|
| `amount_threshold` | `{ min_amount: <int64-cents>, threshold: <int32-approvals>, asset: { id, name } }` | Block payouts above a dollar amount (set `threshold: 0` for hard deny) |
| `approval_threshold` | `{ threshold: <int32-approvals>, description?: <string> }` | Require N approvers (independent of amount) |
| `address_list` | `{ addresses: ["0x...", ...] }` | Allow/deny a list of addresses |

Allow vs deny semantics come from the rule's outer `action` field, not
from anything inside `definition`. For `address_list`, the platform
validator at `server_policies.go` only reads `addresses` — there is no
`list_type` field inside `definition`:

- `{ rule_type: "address_list", action: "allow", definition: { addresses: [...] } }` — outbound to anything else is denied.
- `{ rule_type: "address_list", action: "deny",  definition: { addresses: [...] } }` — outbound to anything else is allowed.

## Attaching policies to wallets

A policy only fires on wallet transactions when attached. Read paths
are unauthenticated:

```ts
// Read — bare call
const wallets = await dakota.policies.getWallets(policy.id);
```

Mutations (`attachToWallet`, `detachFromWallet`) require an
endorsement — see the section above.

One-off transactions (`transactions.create`) are not gated by policies
attached to wallets — they're not from a wallet. Build your guardrails
on the **wallet-transaction path** if you need in-flight enforcement,
or implement application-layer checks before calling
`transactions.create` for one-offs.

## Listing existing policies

```ts
const policies = await dakota.policies.list().toArray();
for (const p of policies) {
  console.log(p.id, p.name, "rules:", p.rules?.length);
  for (const r of p.rules ?? []) {
    console.log("  ", r.rule_type, r.action, r.definition);
  }
}
```

## Lazy signer group provisioning

The cleanest production pattern: one app-wide signer group, reused for
every policy. Lazy-creation on first policy is the
agent-friendly shape:

```ts
const DEFAULT_GROUP_NAME = "Compliance Operators";

async function ensureComplianceSignerGroup() {
  const dakota = getDakotaClient();
  const existing = (await dakota.signerGroups.list().toArray())
    .find(g => g.name === DEFAULT_GROUP_NAME);
  if (existing) return existing.id;

  const { publicKey } = generateKeyPairSync("ec", {
    namedCurve: "P-256",
    publicKeyEncoding: { type: "spki", format: "der" },
    privateKeyEncoding: { type: "pkcs8", format: "der" },
  });
  const memberKey = Buffer.from(publicKey).toString("base64");

  await dakota.signers.create({
    name: "Compliance Operator",
    public_key: memberKey,
    key_type: "ES256",
  });
  const group = await dakota.signerGroups.create({
    name: DEFAULT_GROUP_NAME,
    member_keys: [memberKey],
  });
  return group.id;
}

async function createUsdAmountThresholdPolicy(opts: {
  name: string;
  description?: string;
  /** USD threshold in DOLLARS (we convert to cents below). */
  amountUsd: number;
}) {
  const dakota = getDakotaClient();
  const signerGroupId = await ensureComplianceSignerGroup();
  return dakota.policies.create({
    name: opts.name,
    description: opts.description,
    signer_group_id: signerGroupId,
    rules: [
      {
        rule_type: "amount_threshold",
        action: "deny",
        definition: {
          min_amount: Math.round(opts.amountUsd * 100), // dollars → cents
          threshold: 0,                                 // 0 approvers = hard deny
          asset: { id: "USD", name: "US Dollar" },
        },
      },
    ],
  } as any); // cast to bypass the Record<string, never> OpenAPI quirk
}
```

The `as any` (or `as unknown as PolicyCreateRequest`) is the cleanest
way to deal with the SDK's broken `definition` typing.

## Mutations are endorsed

Once a policy exists, these methods require an `endorsement` envelope
in `RequestOptions` (a `{ signatures, intent }` shape signed by a
member of the policy's `signer_group`):

- `policies.delete(id, { endorsement })`
- `policies.deleteRule(policyId, ruleId, { endorsement })`
- `policies.attachToWallet(policyId, walletId, { endorsement })`
- `policies.detachFromWallet(policyId, walletId, { endorsement })`

`policies.create()`, `policies.list()`, `policies.get()`,
`policies.addRule()`, and `policies.updateRule()` accept bare data
without endorsement.

For a sandbox demo the simplest move is to **not expose delete /
attach in the UI** — show that policies exist, and skip the lifecycle
operations. The full signing flow (canonicalize intent → SHA-256 →
ECDSA P-256 → base64) is the same as wallet transactions; see the
[wallet-signing](https://docs.dakota.xyz/documentation/wallet-signing)
and [signing-guide](https://docs.dakota.xyz/documentation/signing-guide)
docs.

## What next

- [`dakota-send-payment`](../dakota-send-payment/SKILL.md) for the path
  policies enforce on.
- [`dakota-receive-usdc`](../dakota-receive-usdc/SKILL.md) to provision
  the wallet a policy attaches to.

## Canonical reference

- Policies overview — [docs.dakota.xyz/documentation/policies](https://docs.dakota.xyz/documentation/policies)
- Wallet signing (intent canonicalization) — [docs.dakota.xyz/documentation/wallet-signing](https://docs.dakota.xyz/documentation/wallet-signing)
- Without the SDK: `POST /signers`, `POST /signer-groups`, `POST /policies`,
  `POST /policies/{id}/rules`, `POST /policies/{id}/wallets/{walletId}` —
  see [docs.dakota.xyz/api-reference](https://docs.dakota.xyz/api-reference).
