
# Yarah + Kinde Integration Guide

Kinde **does not support custom JWT signing keys**, so you sign a separate JWT server-side using `jsonwebtoken`. The flow: get the Kinde user from the server session → sign a JWT with Yarah's secret → pass it to Yarah as `accessToken` (deprecated alias: `edgeFunctionToken`).

## Key packages

- `@kinde-oss/kinde-auth-nextjs` — Kinde SDK for Next.js
- `@yarahdev/sdk` — Yarah client
- `jsonwebtoken` + `@types/jsonwebtoken` — for server-side JWT signing

## Recommended Workflow

```text
1. Create Kinde application        → Kinde Dashboard (manual)
2. Create/link Yarah project    → npx -y @yarahdev/cli create or link
3. Install deps + configure env    → npm install, .env.local
4. Create Kinde auth route         → app/api/auth/[kindeAuth]/route.js
5. Create Yarah client utility  → lib/yarah.ts (server-side JWT signing)
6. Set up Yarah database        → requesting_user_id() + table + RLS
7. Build features                  → CRUD pages using Yarah client
```

## Dashboard setup (manual, cannot be automated)

### Kinde Application
- Create in Kinde Dashboard > Add application
- Type: **Back-end web**, SDK: **Next.js**
- Set **Allowed callback URL** to `http://localhost:3000/api/auth/kinde_callback`
- Set **Allowed logout redirect URL** to `http://localhost:3000`
- Enable desired auth methods (Email, Google, etc.) under Authentication
- Note down **Domain**, **Client ID**, **Client Secret** from App Keys

### Yarah Project
- Create via `npx -y @yarahdev/cli create` or link via `npx -y @yarahdev/cli link --project-id <id>`
- Get the JWT secret via CLI: `npx -y @yarahdev/cli secrets get JWT_SECRET`
- Note down **URL** and **Anon Key** from Yarah, then export the CLI value as `YARAH_JWT_SECRET`

## Kinde auth route

- Create `app/api/auth/[kindeAuth]/route.js` that exports `handleAuth()` from `@kinde-oss/kinde-auth-nextjs/server`

```javascript
// app/api/auth/[kindeAuth]/route.js
import { handleAuth } from "@kinde-oss/kinde-auth-nextjs/server";

export const GET = handleAuth();
```

## Yarah client

- Create a server-side utility at `lib/yarah.ts` — cannot be used in client components
- Use `getKindeServerSession()` to get `getUser`
- Sign a JWT with `jsonwebtoken` using `process.env.YARAH_JWT_SECRET`
- Required claims: `sub` (from `user.id`), `role: "authenticated"`, `aud: "yarah-api"`, `email`
- Set `expiresIn: '1h'`
- Pass the signed token as `accessToken` to `createClient`

```typescript
// lib/yarah.ts
import { createClient } from '@yarahdev/sdk';
import { getKindeServerSession } from '@kinde-oss/kinde-auth-nextjs/server';
import jwt from 'jsonwebtoken';

export async function createYarahClient() {
  const { getUser } = getKindeServerSession();
  const user = await getUser();

  let accessToken: string | undefined;
  if (user) {
    accessToken = jwt.sign(
      {
        sub: user.id,
        role: 'authenticated',
        aud: 'yarah-api',
        email: user.email,
      },
      process.env.YARAH_JWT_SECRET!,
      { expiresIn: '1h' }
    );
  }

  return createClient({
    baseUrl: process.env.NEXT_PUBLIC_YARAH_URL!,
    accessToken,
  });
}
```

## Database setup

- Kinde user IDs are strings (e.g. `kp_1234abcd`), not UUIDs — use `TEXT` columns for `user_id`
- Create a `requesting_user_id()` SQL function that extracts the `sub` claim from `auth.jwt()` as text
- Set `user_id` column default to `requesting_user_id()` so it auto-populates on insert
- Enable RLS and create policies that compare `user_id = requesting_user_id()`

```sql
create or replace function public.requesting_user_id()
returns text
language sql stable
as $$
  select nullif(auth.jwt() ->> 'sub', '')::text
$$;
```

## Environment variables

| Variable | Source |
|----------|--------|
| `KINDE_CLIENT_ID` | Kinde Dashboard > App Keys |
| `KINDE_CLIENT_SECRET` | Kinde Dashboard > App Keys |
| `KINDE_ISSUER_URL` | `https://YOUR_DOMAIN.kinde.com` |
| `KINDE_SITE_URL` | `http://localhost:3000` |
| `KINDE_POST_LOGOUT_REDIRECT_URL` | `http://localhost:3000` |
| `KINDE_POST_LOGIN_REDIRECT_URL` | `http://localhost:3000` |
| `NEXT_PUBLIC_YARAH_URL` | Yarah Dashboard |
| `NEXT_PUBLIC_YARAH_ANON_KEY` | Yarah Dashboard |
| `YARAH_JWT_SECRET` | Yarah CLI (`npx -y @yarahdev/cli secrets get JWT_SECRET`) |

## Common Mistakes

| Mistake | Solution |
|---------|----------|
| ❌ Using Kinde's JWT directly with Yarah | ✅ Kinde doesn't sign with your secret — sign a separate JWT server-side |
| ❌ Using Yarah client in a client component | ✅ `getKindeServerSession` is server-only — keep the utility server-side |
| ❌ Using `auth.uid()` for RLS policies | ✅ Use `requesting_user_id()` — Kinde IDs are strings, not UUIDs |
