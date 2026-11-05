---
name: dakota-receive-usdc
description: |
  Provision a Dakota wallet (signer → signer group → policy → wallet) so
  the customer can receive USDC directly on-chain, and send from it with
  a signed intent. Use when building any product where customers,
  counterparties, or upstream payers send stablecoin to the operator's
  treasury, global payouts, marketplace payments, AI-native treasury,
  cross-border invoicing.
---

# Dakota: Receive USDC (treasury wallet)

A Dakota wallet is a non-custodial multi-sig that holds stablecoin
on-chain. The customer gets a real address; anyone in the world can
send USDC to it and balances update on-chain in seconds.

This is the modern receive rail. Pair with
[`dakota-receive-usd`](../dakota-receive-usd/SKILL.md) when you also need
ACH inbound from US-based payers.

## The chain (4 calls)

```ts
import { generateKeyPairSync } from "node:crypto";
import { P256MandateSigner, type PolicyRuleCreateRequest } from "@dakota-xyz/ts-sdk";

type Definition = PolicyRuleCreateRequest["definition"];

// 1. Mint a P-256 signer keypair. KEEP privateKeyDer (PKCS#8 DER) in
//    KMS / HSM / Vault: you need it to send from the wallet and to
//    change its policies.
const { privateKey: privateKeyDer } = generateKeyPairSync("ec", {
  namedCurve: "P-256",
  publicKeyEncoding: { type: "spki", format: "der" },
  privateKeyEncoding: { type: "pkcs8", format: "der" },
});
const signer = P256MandateSigner.fromPrivateKey(privateKeyDer);
const publicKey = signer.publicKeyBase64(); // base64 DER SPKI

// 2. Register the signer, then wrap it in a signer group
//    (wallets attach to groups, not raw signers)
await dakota.signers.create({
  name: `${customer.name} Operator Signer`,
  public_key: publicKey,
  key_type: "ES256", // ◀── uppercase. "ES256" or "WEBAUTHN" only.
});
const group = await dakota.signerGroups.create({
  name: `${customer.name} Operator Group`,
  member_keys: [publicKey],
});

// 3. A policy, so the wallet can SEND. With no policy, every outbound
//    send is denied. This rule allows a send with 1 group signature.
//    See dakota-compliance-policies for caps and allowlists.
const policy = await dakota.policies.create({
  name: `${customer.name} default policy`,
  signer_group_id: group.id,
  rules: [
    {
      rule_type: "approval_threshold",
      action: "allow",
      definition: { threshold: 1 } as unknown as Definition,
    },
  ],
});

// 4. Provision the wallet. Policies passed here attach with no endorsement.
const wallet = await dakota.wallets.create({
  customer_id: customer.id,
  family: "evm",            // "evm" or "solana"
  name: `${customer.name} Treasury`,
  signer_groups: [group.id],
  policies: [policy.id],
});
// wallet.id      , the wallet KSUID
// wallet.address , the on-chain address. SHARE THIS with payers.
```

## What goes wrong if you skip a step

- **No signer:** `signerGroups.create` rejects with `Signer Not Found`.
  The order is fixed: signer → group → wallet.
- **`signer_groups: []`:** rejected. A wallet needs at least one group.
- **`policies: []`:** accepted, but the wallet can only receive. Every
  send is denied with `No policies found for wallet`. Adding a policy
  later needs an endorsement (see
  [`dakota-compliance-policies`](../dakota-compliance-policies/SKILL.md)),
  so pass one at create time.

## What goes wrong if you use the wrong key type

The only `key_type` values are `ES256` and `WEBAUTHN`, in uppercase.
Any other value returns 400
`value is not one of the allowed values ["ES256","WEBAUTHN"]`.

**Ed25519 is not supported.** `signers.create` rejects it with 400
`invalid public_key`. Use a P-256 key (`ES256`, base64 DER SPKI) or a
passkey (`WEBAUTHN`, a base64 COSE ES256/RS256 key; see
[WebAuthn signing](https://docs.dakota.xyz/documentation/webauthn-signing)).

## The private key: keep or discard?

| Context | Keep? | Why |
|---|---|---|
| The wallet only ever receives | Discard ok | Receiving needs no signature |
| You send from the wallet, or change its policies | **Keep in HSM/KMS** | Each send and each policy change is signed with this key |

One-off payouts (`transactions.create`) need no wallet signature, but
you fund them by sending USDC from the wallet, and that send is signed.
Store the key in AWS KMS / GCP Cloud KMS / HashiCorp Vault. Do not
commit it, do not ship it to the browser.

## Sharing the address with payers

The wallet's `address` is your treasury's receive address. Display it
verbatim, with a QR option:

```ts
const wallet = await dakota.wallets.get(walletId);
console.log("Share this address:", wallet.address);
const balances = await dakota.wallets.getBalances(walletId);
```

An `evm` wallet has one address on every EVM chain Dakota supports
(Ethereum, Base, Arbitrum, Optimism, Polygon, Avalanche, and their
testnets). Pick one network for your product and tell payers which
one. Sandbox runs on **testnets** (e.g. Ethereum Sepolia, Base Sepolia).

## Sandbox testing

Sandbox wallets are real on testnets: deposits are real testnet
transfers and fire the same `wallet.deposit` webhooks as production.
Three ways to fund one:

1. **Faucet:** `POST /sandbox/wallets/{wallet_id}/faucet` sends testnet
   RD on Base Sepolia, up to $2 per call. It returns 202 with a
   `simulation_id`; poll `dakota.sandbox.getSimulation(id)`. The TS SDK
   has no faucet method yet, so call it over HTTP:

   ```ts
   await fetch(`https://api.platform.sandbox.dakota.xyz/sandbox/wallets/${wallet.id}/faucet`, {
     method: "POST",
     headers: {
       "x-api-key": process.env.DAKOTA_API_KEY!,
       "x-idempotency-key": crypto.randomUUID(),
       "content-type": "application/json",
     },
     body: JSON.stringify({ asset: "RD", amount: "2.00" }),
   });
   ```

2. **Real testnet USDC** from a public faucet (e.g.
   [Circle faucet](https://faucet.circle.com)) to `wallet.address`.
3. **ACH in:** an on-ramp whose crypto destination is this wallet, then
   `sandbox.simulateInbound({ type: "ach_inbound" })`. The USDC leg is a
   real testnet transfer. See
   [`dakota-receive-usd`](../dakota-receive-usd/SKILL.md).

See [`dakota-sandbox-testing`](../dakota-sandbox-testing/SKILL.md).

## Wallet transactions (sending)

To **send** from the wallet (e.g. to fund a one-off payout's
`crypto_address`), sign a transfer intent and post it to
`POST /wallets/{id}/transactions`. The SDK ships the canonicalizer and
the signer:

```ts
import { randomUUID } from "node:crypto";
import { P256MandateSigner, canonicalJSON, type SendTransactionIntent } from "@dakota-xyz/ts-sdk";

const signer = P256MandateSigner.fromPrivateKey(privateKeyDer); // from your KMS
const intent: SendTransactionIntent = {
  wallet_id: wallet.id,
  caip2: "eip155:11155111",   // Ethereum Sepolia. eip155:1 = Ethereum mainnet
  operation: {
    kind: "transfer",
    from: wallet.address,
    to: targetAddress,
    amount: "100",            // shortest form: "100", not "100.00" (see below)
    asset_id: "USDC",
  },
  idempotency_key: randomUUID(),
};

// RFC 8785 JCS → SHA-256 → ECDSA P-256 (DER) → base64
const signature = signer.sign(new TextEncoder().encode(canonicalJSON(intent)));

await dakota.wallets.createTransaction(wallet.id, {
  signatures: [signature],
  intent,
});
```

**Write `amount` in its shortest decimal form** (`"100"`, `"1.5"`).
The server parses the amount as a decimal and checks the signature
against the re-serialized intent, so trailing zeros (`"100.00"`) make
your signature cover different bytes.

Progress arrives as `wallet.transaction.updated` webhooks
(`status_code`: `waiting_for_signature → broadcasted →
awaiting_confirmation → completed | failed | reversed`).

## Reference implementation

A lazy provisioning function, idempotent, caches the wallet on your side
so you only call Dakota once:

```ts
async function ensureTreasuryWallet(
  customer: { id: string; name: string },
  profileDb: Profile,
) {
  if (profileDb.dakota_wallet_id && profileDb.dakota_wallet_address) {
    return {
      walletId: profileDb.dakota_wallet_id,
      address: profileDb.dakota_wallet_address,
    };
  }

  const dakota = getDakotaClient();
  const { privateKey: privateKeyDer } = generateKeyPairSync("ec", {
    namedCurve: "P-256",
    publicKeyEncoding: { type: "spki", format: "der" },
    privateKeyEncoding: { type: "pkcs8", format: "der" },
  });
  // Store the key BEFORE you register it, so a crash cannot orphan the wallet.
  await kms.storeSignerKey(customer.id, privateKeyDer);
  const publicKey = P256MandateSigner.fromPrivateKey(privateKeyDer).publicKeyBase64();

  await dakota.signers.create({
    name: `${customer.name} Signer`,
    public_key: publicKey,
    key_type: "ES256",
  });
  const group = await dakota.signerGroups.create({
    name: `${customer.name} Group ${Date.now() % 100000}`,
    member_keys: [publicKey],
  });
  const policy = await dakota.policies.create({
    name: `${customer.name} default policy`,
    signer_group_id: group.id,
    rules: [
      { rule_type: "approval_threshold", action: "allow",
        definition: { threshold: 1 } as unknown as Definition },
    ],
  });
  const wallet = await dakota.wallets.create({
    customer_id: customer.id,
    family: "evm",
    name: `${customer.name} Treasury`,
    signer_groups: [group.id],
    policies: [policy.id],
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
  accept ACH inbound. The on-ramp's crypto destination can point at
  this wallet's address so US ACH → USDC lands directly in your treasury.
- [`dakota-send-payment`](../dakota-send-payment/SKILL.md) to spend
  from the wallet.
- [`dakota-compliance-policies`](../dakota-compliance-policies/SKILL.md) to
  add amount caps or address allowlists.
- [`dakota-webhooks`](../dakota-webhooks/SKILL.md) for `wallet.deposit`.

## Canonical reference

- Wallets overview: [docs.dakota.xyz/documentation/wallets](https://docs.dakota.xyz/documentation/wallets)
- Wallet signing (intent canonicalization + ECDSA): [docs.dakota.xyz/documentation/wallet-signing](https://docs.dakota.xyz/documentation/wallet-signing)
- Signing guide (end-to-end): [docs.dakota.xyz/documentation/signing-guide](https://docs.dakota.xyz/documentation/signing-guide)
- Passkey (WebAuthn) signers: [docs.dakota.xyz/documentation/webauthn-signing](https://docs.dakota.xyz/documentation/webauthn-signing)
- Without the SDK: `POST /signers`, `POST /signer-groups`, `POST /policies`, `POST /wallets`,
  `GET /wallets/{id}/balances`, `POST /wallets/{id}/transactions`,
  `POST /sandbox/wallets/{id}/faucet`: see
  [docs.dakota.xyz/api-reference/introduction](https://docs.dakota.xyz/api-reference/introduction).
