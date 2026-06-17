---
name: dakota-auth-bootstrap
description: |
  End-user authentication for an app that uses Dakota. Auth is OPTIONAL
  and external to Dakota — Dakota doesn't ship one and doesn't care
  which one you use. All Dakota calls happen server-side with your
  Dakota API key, no matter what auth layer (if any) sits in front.
  Use this skill only if you're building a multi-user app on top of
  Dakota (e.g. "my customers can log in and onboard themselves") — and
  ask the user which provider they want before writing any code.
---

# Dakota — Auth bootstrap

**Auth is external to Dakota and entirely optional.**

Dakota is a server-side API. Every call from your backend authenticates
with your Dakota **API key** (`x-api-key` header). Dakota knows
nothing about your end users.

You only need an auth layer if **you're letting your own users log in
to your app** — e.g. a SaaS where each signed-in user maps to one
Dakota customer. If you're building anything else, skip this skill:

- A script, cron job, or CLI → no auth.
- A single-operator dashboard (one person, one Dakota customer) → no auth.
- A webhook receiver → no auth (the webhook signature is the gate).
- A purely server-side automation → no auth.

If you DO need it, what kind of auth and how it stores user state is
100% your choice. Dakota has no opinion.

## STEP 1 — Ask the user

Before writing any auth code, ASK. Don't pick one for them. Surface
the options and let them decide:

> *"Auth is optional and external to Dakota. Do you want to add a
> login layer for your users, or skip it? If yes, what should I use?
> Common picks: **Supabase** (Postgres + auth + RLS bundled, fastest
> start), **Clerk** (drop-in UI components), **NextAuth / Auth.js**
> (self-host, open source), **Lucia** (low-level, BYO everything),
> a **custom** session / JWT setup you already have, or **none** at
> all."*

Wait for the answer.

| Option | When it's the right pick |
|---|---|
| **None** | Server-only app, scripts, single-operator tools, webhook receivers. Default. |
| **Supabase** | Quickest start, hosted free tier, Postgres + RLS included. Example below. |
| **Clerk** | Want polished sign-in UI without building it. |
| **NextAuth / Auth.js** | Self-hosting, BYO database. |
| **Lucia** | Lower-level, full control. |
| **Custom** | You already have an auth system. |

If they pick **none**, stop here — no auth code needed. Your route
handlers just call Dakota directly with the API key.

If they pick a provider, the pattern below applies (and there's a
concrete Supabase example at the end if that's what they picked).

## The universal pattern

Whatever auth provider you pick, the contract on the Dakota side is
the same:

1. **A logged-in user**, identified by some stable id (`user.id`,
   `session.userId`, etc.).
2. **A lookup table** that maps that id → the user's `dakota_customer_id`
   (and any other per-user Dakota state you cache, e.g. `kyb_status`,
   `dakota_wallet_id`).
3. **Every Dakota route handler** does (a) check the auth gate, (b) read
   the user's `dakota_customer_id` from the lookup, (c) call Dakota.

That lookup table is usually a `profiles` (or `users`, `accounts`,
whatever) table in your own DB. Schema in the simplest case:

```sql
CREATE TABLE profiles (
  id           <your-auth-user-id-type> PRIMARY KEY,
  email        text,
  dakota_customer_id      text UNIQUE,
  dakota_application_id   text UNIQUE,
  kyb_status              text,
  -- add fields as you provision more Dakota resources:
  -- dakota_wallet_id, dakota_wallet_address, etc.
  created_at   timestamptz DEFAULT now(),
  updated_at   timestamptz DEFAULT now()
);
```

How you populate the row is provider-specific:

| Provider | When the profile row gets created |
|---|---|
| Supabase | Postgres trigger on `auth.users` insert (see example below) |
| Clerk | Webhook handler for `user.created` |
| NextAuth | The `events.signIn` / database adapter callbacks |
| Custom | Wherever you create the user record |

## Route handler shape (any provider)

```ts
// app/api/some-action/route.ts (Next.js App Router example)

export async function POST(request: Request) {
  // 1. Auth gate — get the user from whatever provider you're using
  const user = await getCurrentUser(request); // your auth helper
  if (!user) {
    return Response.json({ error: "Unauthorized" }, { status: 401 });
  }

  // 2. Look up the Dakota customer id from your DB
  const profile = await db.profiles.findOne({ id: user.id });
  if (!profile?.dakota_customer_id) {
    return Response.json({ error: "Not onboarded" }, { status: 400 });
  }

  // 3. Call Dakota
  const dakota = getDakotaClient();
  const result = await dakota.someResource.someMethod({ /* ... */ });
  return Response.json({ id: result.id });
}
```

Swap `getCurrentUser` and `db.profiles.findOne` for whatever your
stack actually uses. The five-line shape is the contract — everything
else is the auth provider's API.

## One concrete example — Supabase

If the user picks Supabase, the helpers below work. If they pick
something else, adapt the route handler pattern above and skip the
rest of this section.

```bash
npm install @supabase/ssr @supabase/supabase-js
```

```bash
# .env.local
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
```

**Three SSR helpers** (Supabase needs separate clients for browser,
server, and middleware):

```ts
// lib/supabase/client.ts — browser ("use client") code
import { createBrowserClient } from "@supabase/ssr";

export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  );
}
```

```ts
// lib/supabase/server.ts — Server Components + Route Handlers
import { createServerClient } from "@supabase/ssr";
import { cookies } from "next/headers";

export async function createClient() {
  const cookieStore = await cookies();
  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return cookieStore.getAll();
        },
        setAll(cookiesToSet) {
          try {
            cookiesToSet.forEach(({ name, value, options }) =>
              cookieStore.set(name, value, options)
            );
          } catch {
            // ok if middleware refreshes
          }
        },
      },
    }
  );
}
```

```ts
// lib/supabase/middleware.ts — used from middleware.ts
import { createServerClient } from "@supabase/ssr";
import { NextResponse, type NextRequest } from "next/server";

export async function updateSession(request: NextRequest) {
  let supabaseResponse = NextResponse.next({ request });
  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() { return request.cookies.getAll(); },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value }) =>
            request.cookies.set(name, value)
          );
          supabaseResponse = NextResponse.next({ request });
          cookiesToSet.forEach(({ name, value, options }) =>
            supabaseResponse.cookies.set(name, value, options)
          );
        },
      },
    }
  );

  const { data: { user } } = await supabase.auth.getUser();

  if (
    !user &&
    !request.nextUrl.pathname.startsWith("/login") &&
    !request.nextUrl.pathname.startsWith("/signup") &&
    !request.nextUrl.pathname.startsWith("/api") &&
    request.nextUrl.pathname !== "/"
  ) {
    const url = request.nextUrl.clone();
    url.pathname = "/login";
    return NextResponse.redirect(url);
  }

  return supabaseResponse;
}
```

**Profiles table + auto-insert trigger** (Supabase SQL Editor):

```sql
CREATE TABLE IF NOT EXISTS public.profiles (
  id uuid REFERENCES auth.users ON DELETE CASCADE PRIMARY KEY,
  email text,
  dakota_customer_id text UNIQUE,
  dakota_application_id text UNIQUE,
  -- kyb_status mirrors Dakota's KybStatus enum: active|pending|partner_review|
  -- rejected|frozen|auto_declined. Stored as free-form text so future Dakota
  -- additions don't require a migration here.
  kyb_status text,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

CREATE OR REPLACE FUNCTION public.handle_new_user()
RETURNS trigger AS $$
BEGIN
  INSERT INTO public.profiles (id, email, created_at, updated_at)
  VALUES (new.id, new.email, now(), now());
  RETURN new;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

DROP TRIGGER IF EXISTS on_auth_user_created ON auth.users;
CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE PROCEDURE public.handle_new_user();

ALTER TABLE public.profiles ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Users can view own profile"
  ON public.profiles FOR SELECT USING (auth.uid() = id);
CREATE POLICY "Users can update own profile"
  ON public.profiles FOR UPDATE USING (auth.uid() = id);
```

## What next

- [`dakota-project-scaffold`](../dakota-project-scaffold/SKILL.md) —
  where the auth helpers slot into a Next.js App Router project.
- [`dakota-customer-onboarding`](../dakota-customer-onboarding/SKILL.md) —
  what to do after a user signs up: create their Dakota customer and
  drive KYB.

## Canonical reference

- Supabase Next.js docs — [supabase.com/docs/guides/auth/server-side/nextjs](https://supabase.com/docs/guides/auth/server-side/nextjs)
- Clerk Next.js docs — [clerk.com/docs/quickstarts/nextjs](https://clerk.com/docs/quickstarts/nextjs)
- NextAuth / Auth.js — [authjs.dev](https://authjs.dev)
- Lucia — [lucia-auth.com](https://lucia-auth.com)

Dakota has no opinion on the auth layer — this skill is a pattern, not
a Dakota requirement.
