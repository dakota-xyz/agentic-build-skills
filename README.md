# Dakota: Agentic Build Skills

Optional skills to give your coding agent (Claude Code, Cursor) a
head start on Dakota concepts where the field names or call order
are easy to miss.

The Dakota SDKs and the docs already ship their own agent-facing
material: `AGENTS.md` and `CLAUDE.md` in the TypeScript and Go SDK repos. This pack is
just extra shortcuts for common build patterns.

These skills are SDK-agnostic. They describe the concepts and call
order, not the transport. Use them with the official Dakota SDKs
(TypeScript, Go), raw HTTP, or whatever client you prefer. Same idea
applies to the framework and auth choices below: swap in whatever
your stack already uses.

## The skills

| Skill | Concept |
|---|---|
| [`dakota-auth-bootstrap`](skills/dakota-auth-bootstrap/SKILL.md) | **Optional.** End-user auth (Supabase, Clerk, NextAuth, custom, or none) if you're building a multi-user app on top of Dakota |
| [`dakota-project-scaffold`](skills/dakota-project-scaffold/SKILL.md) | Where Dakota lives in a Next.js App Router project |
| [`dakota-sdk-bootstrap`](skills/dakota-sdk-bootstrap/SKILL.md) | Install + configure the TS SDK |
| [`dakota-customer-onboarding`](skills/dakota-customer-onboarding/SKILL.md) | Create a customer + drive KYB to active |
| [`dakota-receive-usdc`](skills/dakota-receive-usdc/SKILL.md) | Provision a wallet to receive stablecoin on-chain |
| [`dakota-receive-usd`](skills/dakota-receive-usd/SKILL.md) | Provision an ACH on-ramp to receive USD |
| [`dakota-send-payment`](skills/dakota-send-payment/SKILL.md) | Send a USDC → USD ACH payout |
| [`dakota-international-payout`](skills/dakota-international-payout/SKILL.md) | SWIFT payouts in USD or a foreign currency (FX quotes) |
| [`dakota-webhooks`](skills/dakota-webhooks/SKILL.md) | Register a webhook target, verify signatures, map events to flows |
| [`dakota-sandbox-testing`](skills/dakota-sandbox-testing/SKILL.md) | Simulations, testnet-only and $2 rules, faucet |
| [`dakota-compliance-policies`](skills/dakota-compliance-policies/SKILL.md) | Wallet policies: caps, approvals, address lists |

The first two are framework / auth choices, not Dakota concepts,
swap them if you're using a different stack. Each skill ends with
links back to the relevant page on
[docs.dakota.xyz](https://docs.dakota.xyz) where the canonical
reference lives.

## Before you build: get a Dakota API key

Every Dakota call (from these skills, the SDKs, or raw HTTP)
authenticates with an API key. For **The Agentic Build** you want a
**sandbox** key. That's where the challenge runs and the only place
you can simulate money movement without it being real.

1. Sign up for The Agentic Build at
   [dakota.xyz/agentic-build](https://dakota.xyz/agentic-build):
   this is how you get into the challenge and get access to the
   sandbox dashboard where your key lives.
2. Once you're in, sign in to the sandbox dashboard and open
   **API Keys** → **Create New API Key**.
3. Copy the key. It is a 44-character URL-safe base64 string, shown **only once**.
4. Put it in your project's `.env.local`:

   ```bash
   DAKOTA_API_KEY=
   DAKOTA_ENV=sandbox
   ```

5. Pass it as the `x-api-key` header (the SDKs do this for you).

Full setup instructions (other required headers, idempotency, etc.):
[docs.dakota.xyz/documentation/authentication/api-keys-headers](https://docs.dakota.xyz/documentation/authentication/api-keys-headers).

> Production keys live in a separate dashboard at
> [platform.dakota.xyz](https://platform.dakota.xyz). You don't need
> one for the challenge. You need one only when you go live.

Auth between **your end users** and your app (Supabase, Clerk, etc.) is
**optional and external to Dakota**: see
[`dakota-auth-bootstrap`](skills/dakota-auth-bootstrap/SKILL.md) only
if you're building a multi-user app on top.

## Install

### Claude Code

```bash
cp -r skills/dakota-customer-onboarding ~/your-project/.claude/skills/
```

Or all of them:

```bash
mkdir -p ~/your-project/.claude/skills
cp -r skills/dakota-* ~/your-project/.claude/skills/
```

### Cursor

```bash
for dir in skills/dakota-*; do
  name=$(basename "$dir")
  cp "$dir/SKILL.md" ~/your-project/.cursor/rules/"$name".md
done
```

## Where else to look

- **API and endpoint reference**: [docs.dakota.xyz](https://docs.dakota.xyz)
- **SDKs**: [docs.dakota.xyz/documentation/sdks](https://docs.dakota.xyz/documentation/sdks). Each SDK repo also ships its own `AGENTS.md` / `CLAUDE.md` with the full method reference.
- **Feeding docs to your agent**: [docs.dakota.xyz/documentation/using-llms](https://docs.dakota.xyz/documentation/using-llms) and [docs.dakota.xyz/llms.txt](https://docs.dakota.xyz/llms.txt)
- **Agentic payments (beta)**: payment agents, mandates, x402: [docs.dakota.xyz/documentation/agentic-payments](https://docs.dakota.xyz/documentation/agentic-payments)
- **The challenge**: [dakota.xyz/agentic-build](https://dakota.xyz/agentic-build)

## License

MIT. See [LICENSE](LICENSE). Fork and modify freely.
