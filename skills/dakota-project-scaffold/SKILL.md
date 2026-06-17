---
name: dakota-project-scaffold
description: |
  Where Dakota lives in a Next.js App Router project — directory layout,
  the singleton SDK client, route handlers vs server components, env
  files, and how the pieces talk to Supabase auth. Use when standing up
  a new Dakota-on-Next.js project, when an agent is unsure where to put
  the next API route, or when an existing project is hitting Dakota
  from client code (which leaks the API key).
---

# Dakota — Project scaffold (Next.js App Router)

The whole reason this skill exists: **Dakota API calls must run on the
server.** The API key cannot ship to the browser. Anything that calls
Dakota goes in a Server Component, a route handler, or a server action.

This skill is the App Router layout that worked for the neobank demo —
the directory structure, where the singleton lives, what goes where.

## Directory layout

```
your-app/
├── app/
│   ├── (auth)/                  # Auth-only routes (login, signup)
│   │   ├── login/page.tsx
│   │   └── signup/page.tsx
│   ├── (dashboard)/             # Authenticated routes
│   │   ├── dashboard/page.tsx
│   │   ├── treasury/page.tsx
│   │   ├── compliance/page.tsx
│   │   └── settings/page.tsx
│   ├── api/
│   │   ├── onboarding/route.ts        # POST: create customer + KYB
│   │   ├── account/                   # On-ramp / treasury account
│   │   ├── workers/                   # Payees / off-ramp recipients
│   │   ├── transactions/route.ts      # GET: list, POST: create
│   │   ├── treasury/route.ts          # GET: balances + history
│   │   ├── compliance/policies/       # Policies CRUD
│   │   └── webhooks/dakota/route.ts   # Dakota → your app
│   ├── layout.tsx
│   └── page.tsx                       # Landing
├── lib/
│   ├── dakota.ts                      # SDK client singleton (server-only)
│   └── supabase/
│       ├── client.ts                  # Browser
│       ├── server.ts                  # Server components + routes
│       └── middleware.ts              # Middleware helper
├── middleware.ts                       # Auth gate
├── supabase/
│   └── migrations/
│       └── *.sql                      # Idempotent schema
├── .env.local
└── package.json
```

Route groups `(auth)` and `(dashboard)` don't affect URLs — they let
you split layouts (e.g. dashboard sidebar vs login splash).

## `lib/dakota.ts` (server-only)

```ts
import { DakotaClient, Environment } from "@dakota-xyz/ts-sdk";

let _client: DakotaClient | null = null;

export function getDakotaClient(): DakotaClient {
  if (!_client) {
    if (!process.env.DAKOTA_API_KEY) {
      throw new Error("DAKOTA_API_KEY is required");
    }
    _client = new DakotaClient({
      apiKey: process.env.DAKOTA_API_KEY,
      environment:
        process.env.DAKOTA_ENV === "production"
          ? Environment.Production
          : Environment.Sandbox,
    });
  }
  return _client;
}
```

Import this from any `app/api/*/route.ts` or any Server Component
(default — components without `"use client"`).

## Route handler pattern

Every Dakota route handler does the same five things in order:

```ts
// app/api/some-action/route.ts
import { createClient } from "@/lib/supabase/server";
import { getDakotaClient } from "@/lib/dakota";

export async function POST(request: Request) {
  // 1. Auth gate
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) {
    return Response.json({ error: "Unauthorized" }, { status: 401 });
  }

  // 2. Look up the user's Dakota customer id from your profile table
  const { data: profile } = await supabase
    .from("profiles")
    .select("dakota_customer_id, kyb_status")
    .eq("id", user.id)
    .single();
  if (!profile?.dakota_customer_id || profile.kyb_status !== "active") {
    return Response.json({ error: "Not onboarded" }, { status: 400 });
  }

  // 3. Parse + validate inputs
  const body = await request.json();
  if (!body.amount) {
    return Response.json({ error: "amount required" }, { status: 400 });
  }

  // 4. Call Dakota
  const dakota = getDakotaClient();
  try {
    const result = await dakota.someResource.someMethod({ /* ... */ });
    // 5. Return the slim shape your UI needs (not the whole Dakota response)
    return Response.json({ id: result.id, status: result.status });
  } catch (err) {
    return Response.json(
      { error: err instanceof Error ? err.message : "Failed" },
      { status: 500 }
    );
  }
}
```

This shape works for `POST` (mutate Dakota), `GET` (read from Dakota
via SDK), and any other verb. The auth gate + profile lookup come
first; the SDK call is the meat.

## Server component pattern

If you want to fetch Dakota data inside a server component (e.g. an
account dashboard), skip the route handler and call directly:

```ts
// app/(dashboard)/dashboard/page.tsx
import { createClient } from "@/lib/supabase/server";
import { getDakotaClient } from "@/lib/dakota";

export default async function DashboardPage() {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) return null; // middleware already redirected

  const { data: profile } = await supabase
    .from("profiles")
    .select("dakota_customer_id")
    .eq("id", user.id)
    .single();

  const dakota = getDakotaClient();
  const customer = profile?.dakota_customer_id
    ? await dakota.customers.get(profile.dakota_customer_id)
    : null;

  return <DashboardClient customer={customer} />;
}
```

Faster than a route handler — the data flows directly into the
component, no extra fetch round-trip. Use route handlers when the
client needs to trigger a mutation.

## `.env.local`

```bash
# Dakota
DAKOTA_API_KEY=
DAKOTA_ENV=sandbox

# Supabase (anon key — safe in NEXT_PUBLIC_*)
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=

# Optional — wallet treasury
DAKOTA_WEBHOOK_PUBLIC_KEY=
```

**Never** prefix `DAKOTA_API_KEY` with `NEXT_PUBLIC_` — that ships it
to the browser. The Supabase anon key is safe in `NEXT_PUBLIC_*`
because RLS enforces what it can do.

## Where things go (rule of thumb)

| What you're building | Where it goes |
|---|---|
| "When the user clicks X, call Dakota" | `app/api/*/route.ts` |
| "When the page loads, show Dakota data" | Server Component in `app/.../page.tsx` |
| "Reusable Dakota SDK client" | `lib/dakota.ts` |
| "Helper to read the current user" | `lib/supabase/server.ts` |
| "Schema for a new column" | `supabase/migrations/<timestamp>_<name>.sql` |
| "Dakota webhook receiver" | `app/api/webhooks/dakota/route.ts` |
| "Auth-gated UI section" | `app/(dashboard)/.../page.tsx` |
| "Public UI section" | `app/page.tsx` or `app/(auth)/...` |

## What next

- [`dakota-customer-onboarding`](../dakota-customer-onboarding/SKILL.md) —
  the first Dakota route you'll write.
- [`dakota-receive-usdc`](../dakota-receive-usdc/SKILL.md) /
  [`dakota-receive-usd`](../dakota-receive-usd/SKILL.md) — receive
  rails to wire up next.

## Canonical reference

- Next.js App Router docs — [nextjs.org/docs/app](https://nextjs.org/docs/app)
- Next.js route handlers — [nextjs.org/docs/app/building-your-application/routing/route-handlers](https://nextjs.org/docs/app/building-your-application/routing/route-handlers)
- Next.js server components — [nextjs.org/docs/app/building-your-application/rendering/server-components](https://nextjs.org/docs/app/building-your-application/rendering/server-components)

Dakota itself has no opinion on the framework — this skill is the
pattern that worked for the demo, not a Dakota requirement.
