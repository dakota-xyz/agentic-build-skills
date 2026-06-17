---
name: dakota-receive-usdc
description: |
  Provision a Dakota wallet (signer → signer group → wallet) so the
  customer can receive USDC directly on-chain. Use when building any
  product where customers, counterparties, or upstream payers send
  stablecoin to the operator's treasury — global payouts, marketplace
  payments, AI-native treasury, cross-border invoicing.
---

# Dakota — Receive USDC (treasury wallet)

A Dakota wallet is a non-custodial multi-sig that holds stablecoin
on-chain. The customer gets a real EVM address; anyone in the world can
send USDC to it and balances update on-chain in seconds.

This is the modern receive rail. Pair with
[`dakota-receive-usd`](../dakota-receive-usd/SKILL.md) when you also need
ACH inbound from US-based payers.

## The chain (3 calls)

```ts
import { generateKeyPairSync } from "crypto";

// 1. Mint an ES256 (P-256) signer keypair
const { publicKey, privateKey } = generateKeyPairSync("ec", {
  namedCurve: "P-256",
  publicKeyEncoding: { type: "spki", format: "der" },
  privateKeyEncoding: { type: "pkcs8", format: "der" },
});
const memberKey = Buffer.from(publicKey).toString("base64");

// 2. Register the signer with Dakota
await dakota.signers.create({
  name: `${customer.name} Operator Signer`,
  public_key: memberKey,
  key_type: "ES256", // ◀── uppercase. "es256" is rejected.
});

// 3. Wrap it in a signer group (wallets attach to groups, not raw signers)
const group = await dakota.signerGroups.create({
  name: `${customer.name} Operator Group`,
  member_keys: [memberKey],
});

// 4. Provision the wallet
const wallet = await dakota.wallets.create({
  customer_id: customer.id,
  family: "evm",                // EVM family covers Ethereum + Sepolia
  name: `${customer.name} Treasury`,
  signer_groups: [group.id],
  policies: [],                 // empty for now; see dakota-compliance-policies
});
// wallet.id       — the wallet KSUID
// wallet.address  — the on-chain EVM address. SHARE THIS with payers.
```

## What goes wrong if you skip the signer

`wallets.create` requires `signer_groups: string[]` and `policies: string[]`
— both required, can't be empty for `signer_groups`. Without a
pre-existing signer in the system, `signerGroups.create` rejects with
`Signer Not Found`. The order is fixed: signer → group → wallet.

## What goes wrong if you use the wrong key type

The API accepts two `key_type` values: `ES256` and `WEBAUTHN`. They MUST
be uppercase. The error message ("not one of the allowed values
[es256, webauthn]") helpfully lowercases them, which leads agents to send
lowercase, which gets rejected.

**Ed25519 is not supported.** You'll see `Signer Not Found` because the
key fingerprint doesn't match anything Dakota recognises.

## The private key — keep or discard?

| Context | Keep? | Why |
|---|---|---|
| Sandbox / demo flow that only uses **one-off transactions** | Discard ok | One-off transactions don't require wallet signing |
| Production, or any flow that calls `wallets.createTransaction` | **Keep in HSM/KMS** | You sign transfer intents with this key |

If you're doing the canonical Dakota production flow — wallet sends USDC
to a one-off transaction's `crypto_address` to settle the off-ramp — you
need the private key for [`wallets.createTransaction`](#wallet-transactions-production-flow).
Store it in AWS KMS / GCP Cloud KMS / HashiCorp Vault. Do not commit it,
do not ship it to the browser.

## Sharing the address with payers

The wallet's `address` is your treasury's receive address. Display it
verbatim, with a QR option:

```ts
const wallet = await dakota.wallets.get(walletId);
console.log("Share this address:", wallet.address);
// 0x7089a6f92D4f5f5Fbb1eD96B770C1e43F6a92bFf
```

The address is the same across the EVM family (Ethereum mainnet,
Sepolia testnet, Base, Polygon, etc.) — but pick a single network for
your product and tell payers which one. The most common production
choice is **Ethereum mainnet** with **USDC** (Circle's contract).

## Sandbox testing

Funding a wallet with simulated USDC requires `USE_MOCK_PRIVY=true` on
your sandbox tenant, which is **off by default**. With it off,
`sandbox.simulateInbound({ type: 'crypto_inbound', wallet_id })` returns
502 from the Provider service.

Three paths to fund a sandbox wallet:

1. **Ask Dakota Support** to flip `USE_MOCK_PRIVY=true` on your tenant.
   Then `crypto_inbound` simulations work.
2. **Send real Sepolia USDC** from a faucet-funded EOA
   ([Circle faucet](https://faucet.circle.com)) to the wallet address.
   Dakota's webhooks watch the chain.
3. **Don't fund the wallet** — wire your on-ramp's `crypto_destination_id`
   to the wallet address, then `sandbox.simulateInbound({ type: 'ach_inbound' })`
   on the on-ramp simulates the ACH event (this also won't actually mint
   USDC into the wallet without Privy mock, but it's the cleanest
   architecture for the demo).

See [`dakota-sandbox-testing`](../dakota-sandbox-testing/SKILL.md) for the
gory details.

## Wallet transactions (production flow)

To **send** from the wallet (e.g. to settle a one-off off-ramp transaction
in production), you sign a transfer intent and post it to
`POST /wallets/{id}/transactions`:

```ts
import { canonicalize } from "canonicalize"; // RFC 8785 JCS
import { createSign, createPrivateKey, randomUUID } from "crypto";

const intent = {
  wallet_id: wallet.id,
  caip2: "eip155:1",            // ethereum-mainnet (use 11155111 for sepolia)
  operation: {
    kind: "transfer",
    from: wallet.address,
    to: targetAddress,
    amount: "100.00",
    asset_id: "USDC",
  },
  idempotency_key: randomUUID(),
};

// Canonicalize per RFC 8785 → SHA-256 → ECDSA P-256 ASN.1 DER → base64
const canonical = canonicalize(intent);
const sigDer = createSign("SHA256").update(canonical).sign({
  key: createPrivateKey({ key: privateKeyDer, format: "der", type: "pkcs8" }),
  dsaEncoding: "der",
});
const signature = sigDer.toString("base64");

await dakota.wallets.createTransaction(wallet.id, {
  signatures: [signature],
  intent,
});
```

The canonical JSON form mirrors the
[policy-engine](https://github.com/dakota-xyz/policy-engine)
`CanonicalizeIntent` implementation. The signature is over the SHA-256
hash of the RFC-8785-canonicalized JSON of the intent (alphabetical key
order, normalized numbers, no whitespace).

## Reference implementation

A lazy provisioning function — idempotent, caches the wallet on your side
so you only call Dakota once:

```ts
async function ensureTreasuryWallet(customer: Customer, profileDb: Profile) {
  if (profileDb.dakota_wallet_id && profileDb.dakota_wallet_address) {
    return {
      walletId: profileDb.dakota_wallet_id,
      address: profileDb.dakota_wallet_address,
    };
  }

  const dakota = getDakotaClient();
  const { publicKey } = generateKeyPairSync("ec", {
    namedCurve: "P-256",
    publicKeyEncoding: { type: "spki", format: "der" },
    privateKeyEncoding: { type: "pkcs8", format: "der" },
  });
  const memberKey = Buffer.from(publicKey).toString("base64");

  await dakota.signers.create({
    name: `${customer.name} Signer`,
    public_key: memberKey,
    key_type: "ES256",
  });
  const group = await dakota.signerGroups.create({
    name: `${customer.name} Group ${Date.now() % 100000}`,
    member_keys: [memberKey],
  });
  const wallet = await dakota.wallets.create({
    customer_id: customer.id,
    family: "evm",
    name: `${customer.name} Treasury`,
    signer_groups: [group.id],
    policies: [],
  });

  await profileDb.update({
    dakota_wallet_id: wallet.id,
    dakota_wallet_address: wallet.address,
  });

  return { walletId: wallet.id, address: wallet.address };
}
```

## What next

- Pair with [`dakota-receive-usd`](../dakota-receive-usd/SKILL.md) to also
  accept ACH inbound. The on-ramp's `crypto_destination_id` can point at
  this wallet's address so US ACH → USDC lands directly in your treasury.
- [`dakota-send-payment`](../dakota-send-payment/SKILL.md) to spend
  from the wallet.
- [`dakota-compliance-policies`](../dakota-compliance-policies/SKILL.md) to
  attach amount-threshold or address-list rules to the wallet.

## Canonical reference

- Wallets overview — [docs.dakota.xyz/documentation/wallets](https://docs.dakota.xyz/documentation/wallets)
- Wallet signing (intent canonicalization + ECDSA) — [docs.dakota.xyz/documentation/wallet-signing](https://docs.dakota.xyz/documentation/wallet-signing)
- Signing guide (end-to-end) — [docs.dakota.xyz/documentation/signing-guide](https://docs.dakota.xyz/documentation/signing-guide)
- Without the SDK: `POST /signers`, `POST /signer-groups`, `POST /wallets`,
  `GET /wallets/{id}/balances`, `POST /wallets/{id}/transactions` — see
  [docs.dakota.xyz/api-reference](https://docs.dakota.xyz/api-reference).
