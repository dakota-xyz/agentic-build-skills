---
name: dakota-compliance-policies
description: |
  Create Dakota wallet policies that allow or deny wallet transactions
  before they execute, amount caps, approval thresholds, address
  allow/deny lists. Use when adding guardrails to a product on Dakota
  rails, when an operator needs "block any outbound > $X" controls, or
  when audit / compliance asks for proof of in-line enforcement.
---

# Dakota: Compliance policies

Policies are the platform layer that says "no, that money isn't moving."
Rules attach to policies; policies attach to wallets. Before a wallet
transaction executes, the policy engine evaluates the rules and either
allows or denies.

This is what makes Dakota a compliance product, not just a payments
product. *"You can build the neobank agentically. You can't build the
MTL agentically."*

## How the engine decides (read this first)

1. Each rule either **applies** to the transaction or not.
2. If any applied rule denies, the transaction is **denied**.
3. Else, if any applied rule allows, it is **allowed**.
4. Else (nothing applied), it is **denied**. Default deny.

So a policy that holds only a deny rule denies **every** send once it is
attached: below the cap, no rule applies, and the default is deny.
Always pair caps and deny lists with an allow rule.

A wallet with no policy at all also denies every send
(`No policies found for wallet`).

## The chain

```ts
import { generateKeyPairSync } from "node:crypto";
import { P256MandateSigner, type PolicyRuleCreateRequest } from "@dakota-xyz/ts-sdk";

type Definition = PolicyRuleCreateRequest["definition"];

// 1. P-256 signer keypair (same as wallets, see dakota-receive-usdc).
//    Keep privateKeyDer in KMS: every later policy change is signed with it.
const { privateKey: privateKeyDer } = generateKeyPairSync("ec", {
  namedCurve: "P-256",
  publicKeyEncoding: { type: "spki", format: "der" },
  privateKeyEncoding: { type: "pkcs8", format: "der" },
});
const signer = P256MandateSigner.fromPrivateKey(privateKeyDer);
const publicKey = signer.publicKeyBase64();

// 2. Register signer + signer group (a policy requires a group; its
//    members endorse every later change to the policy)
await dakota.signers.create({
  name: "Compliance Operator",
  public_key: publicKey,
  key_type: "ES256",      // uppercase, required
});
const group = await dakota.signerGroups.create({
  name: "Compliance Operators",
  member_keys: [publicKey],
});

// 3. Policy: allow normal sends, hard-deny 10,000 USDC or more
const policy = await dakota.policies.create({
  name: "High-value transaction cap",
  description: "Deny any USDC transfer of 10,000 USDC or more",
  signer_group_id: group.id,    // required
  rules: [
    // Allow sends that carry 1 signature from the group.
    {
      rule_type: "approval_threshold",
      action: "allow",
      definition: { threshold: 1 } as unknown as Definition,
    },
    // Hard cap. min_amount is in the ASSET'S BASE UNITS.
    // USDC has 6 decimals: 10,000 USDC = 10_000_000_000.
    {
      rule_type: "amount_threshold",
      action: "deny",
      definition: {
        min_amount: 10_000_000_000,
        threshold: 0,            // 0 on a deny rule = hard deny
        asset: { id: "USDC" },   // "USDC" | "USDT" | "RD"
      } as unknown as Definition,
    },
  ],
});

// 4. Attach it: pass it to wallets.create (no endorsement needed)
const wallet = await dakota.wallets.create({
  customer_id: customer.id,
  family: "evm",
  name: "Treasury",
  signer_groups: [group.id],
  policies: [policy.id],
});
```

## The schema gotcha

The SDK types `definition` as `Record<string, never>` (a quirk of
`openapi-typescript`'s handling of free-form objects). Cast **only the
definition**: `{ ... } as unknown as PolicyRuleCreateRequest["definition"]`.
A bare literal fails to compile, and `as any` on the whole request hides
real mistakes.

The server parses each definition strictly and rejects unknown keys.
For `amount_threshold` it accepts exactly `asset`, `min_amount` and
`threshold`:

- `min_amount`: integer ≥ 0, in the asset's **smallest unit**. The
  engine multiplies the transfer amount by 10^decimals before it
  compares. USDC: 6 decimals, so `1_000_000` = **1 USDC**.
- `threshold`: integer. On a `deny` rule, `0` means hard deny. To
  require N approvals at or above an amount, use `action: "allow"` with
  `threshold: N`.
- `asset`: object `{ id, name? }` with `id` `"USDC"`, `"USDT"` or
  `"RD"`. A rule only matches transfers of its own asset, so
  `{ id: "USD" }` never fires on a USDC transfer.

Old `{ amount, currency }` shapes are rejected with
`unknown key "amount" in definition; an amount_threshold rule accepts
asset, min_amount, threshold`.

## Rule types

| `rule_type` | Definition shape | Use case |
|---|---|---|
| `amount_threshold` | `{ min_amount: <int, base units>, threshold: <int ≥ 0>, asset: { id: "USDC" \| "USDT" \| "RD" } }` | Cap or require approvals at or above an amount |
| `approval_threshold` | `{ threshold: <int ≥ 1>, description?: <string> }` | Require N signatures from the group (any amount) |
| `address_list` | `{ addresses: ["0x...", ...] }` (not empty) | Allow or deny a list of addresses |

Allow vs deny semantics come from the rule's outer `action` field, not
from anything inside `definition`. There is no `list_type` field. An
`address_list` rule applies when the transfer's `from`, `to` or token
contract is in the list:

- `action: "allow"`: sends to listed addresses are allowed. Others are
  not covered by this rule, so default deny applies unless another rule
  allows them.
- `action: "deny"`: sends to listed addresses are denied. Others are
  not covered by this rule; another rule (e.g. an `approval_threshold`
  allow) must allow them.

## Mutations are endorsed (sandbox too)

Only these calls need no endorsement: `policies.create`, `list`, `get`,
`getWallets`, and passing `policies: [id]` to `wallets.create`. Every
other change needs an `endorsement` envelope in `RequestOptions` (a
`{ signatures, intent }` shape signed by a member of the policy's
signer group), **in sandbox and in production**:

- `policies.addRule(policyId, rule, { endorsement })`
- `policies.updateRule(policyId, ruleId, rule, { endorsement })`
- `policies.deleteRule(policyId, ruleId, { endorsement })`
- `policies.delete(id, { endorsement })`
- `policies.attachToWallet(policyId, walletId, { endorsement })`
- `policies.detachFromWallet(policyId, walletId, { endorsement })`

Bare data gets 400 `invalid intent format`. The SDK builds the bytes to
sign: `attachPolicyPayload`, `detachPolicyPayload`, `deletePolicyPayload`,
`addPolicyRulePayload`, `updatePolicyRulePayload`,
`removePolicyRulePayload`. The intent's `idempotency_key` must equal
the request's `idempotencyKey`.

```ts
import { randomUUID } from "node:crypto";
import { attachPolicyPayload, addPolicyRulePayload } from "@dakota-xyz/ts-sdk";

// Attach an existing policy to an existing wallet
const k1 = randomUUID();
await dakota.policies.attachToWallet(policy.id, wallet.id, {
  idempotencyKey: k1,
  endorsement: {
    signatures: [signer.sign(attachPolicyPayload(wallet.id, policy.id, k1))],
    intent: {
      type: "attach_policy_to_wallet",
      wallet_id: wallet.id,
      policy_id: policy.id,
      idempotency_key: k1,
    },
  },
});

// Add an address allowlist rule
const def = { addresses: [payeeAddress] };
const k2 = randomUUID();
await dakota.policies.addRule(
  policy.id,
  { rule_type: "address_list", action: "allow", definition: def as unknown as Definition },
  {
    idempotencyKey: k2,
    endorsement: {
      signatures: [signer.sign(addPolicyRulePayload(policy.id, "allow", "address_list", def, k2))],
      intent: {
        type: "add_policy_rule",
        policy_id: policy.id,
        rule_type: "address_list",
        action: "allow",
        definition: def as unknown as Definition,
        idempotency_key: k2,
      },
    },
  },
);
```

For a sandbox demo the simplest move is to create the policy with all
its rules up front and pass it to `wallets.create`. Then nothing needs
an endorsement.

## What policies do not cover

One-off transactions (`transactions.create`) are not gated by wallet
policies, they're not from a wallet. The wallet send that **funds** a
one-off (USDC to its `crypto_address`) is gated. Build your guardrails
on the wallet-transaction path, or add application-layer checks before
you call `transactions.create`.

## Reading policies

Read calls need only your API key:

```ts
const policies = await dakota.policies.list().toArray();
for (const p of policies) {
  console.log(p.id, p.name, "rules:", p.rules?.length);
  for (const r of p.rules ?? []) {
    console.log("  ", r.rule_type, r.action, r.definition);
  }
}
const attached = await dakota.policies.getWallets(policy.id);
const onWallet = await dakota.wallets.getPolicies(wallet.id);
```

## Lazy signer group provisioning

One app-wide signer group, reused for every policy. Keep its private
key in KMS, because you need it to change any policy later:

```ts
const DEFAULT_GROUP_NAME = "Compliance Operators";

async function ensureComplianceSignerGroup(): Promise<string> {
  const dakota = getDakotaClient();
  const existing = (await dakota.signerGroups.list().toArray())
    .find(g => g.name === DEFAULT_GROUP_NAME);
  if (existing) return existing.id;

  const { privateKey: privateKeyDer } = generateKeyPairSync("ec", {
    namedCurve: "P-256",
    publicKeyEncoding: { type: "spki", format: "der" },
    privateKeyEncoding: { type: "pkcs8", format: "der" },
  });
  await kms.storeSignerKey(DEFAULT_GROUP_NAME, privateKeyDer);
  const publicKey = P256MandateSigner.fromPrivateKey(privateKeyDer).publicKeyBase64();

  await dakota.signers.create({
    name: "Compliance Operator",
    public_key: publicKey,
    key_type: "ES256",
  });
  const group = await dakota.signerGroups.create({
    name: DEFAULT_GROUP_NAME,
    member_keys: [publicKey],
  });
  return group.id;
}

async function createUsdcCapPolicy(opts: {
  name: string;
  description?: string;
  /** Cap in whole USDC. */
  capUsdc: number;
}) {
  const dakota = getDakotaClient();
  const signerGroupId = await ensureComplianceSignerGroup();
  return dakota.policies.create({
    name: opts.name,
    description: opts.description,
    signer_group_id: signerGroupId,
    rules: [
      { rule_type: "approval_threshold", action: "allow",
        definition: { threshold: 1 } as unknown as Definition },
      { rule_type: "amount_threshold", action: "deny",
        definition: {
          min_amount: Math.round(opts.capUsdc * 1_000_000), // USDC → base units (6 dp)
          threshold: 0,
          asset: { id: "USDC" },
        } as unknown as Definition },
    ],
  });
}
```

## What next

- [`dakota-receive-usdc`](../dakota-receive-usdc/SKILL.md) to provision
  the wallet a policy attaches to, and to sign wallet sends.
- [`dakota-send-payment`](../dakota-send-payment/SKILL.md) for the
  payouts that a guarded wallet funds.

## Canonical reference

- Policies overview: [docs.dakota.xyz/documentation/policies](https://docs.dakota.xyz/documentation/policies)
- Wallet signing (intent canonicalization): [docs.dakota.xyz/documentation/wallet-signing](https://docs.dakota.xyz/documentation/wallet-signing)
- Signing guide: [docs.dakota.xyz/documentation/signing-guide](https://docs.dakota.xyz/documentation/signing-guide)
- Without the SDK: `POST /signers`, `POST /signer-groups`, `POST /policies`,
  `POST /policies/{id}/rules`, `PUT /policies/{id}/wallets/{wallet_id}`,
  see [docs.dakota.xyz/api-reference/introduction](https://docs.dakota.xyz/api-reference/introduction).
