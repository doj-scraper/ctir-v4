> **Validation status:** Every file reference, line number, and code claim in this document has been verified against the actual source files in `ctir-fe-ofc-v1-main.zip` (frontend) and `ctir-backendv1-official-main.zip` (backend). All claims are accurate as of the commit date.

---

# 3-Phase Migration Analysis: CellTech Distributor Frontend

## Summary of the Current Stack

This is a Next.js 15 App Router frontend (`ctir-fe-ofc-v1`) that currently uses Clerk for authentication. The backend (separate Express/Prisma repo) uses `@clerk/express`. There is no Supabase, no Stripe client code, no Upstash, and no Prisma schema in this frontend repo — all of that lives in the backend. The frontend communicates with the backend exclusively through `lib/api.ts`.

---

## Phase 1 — Migrate Clerk → Supabase Auth

### 1.1 All Clerk References (Frontend Repo Only)

| File | Line(s) | What it Does |
|---|---|---|
| `middleware.ts` | 3–26 | Lazy-imports `clerkMiddleware` + `createRouteMatcher` from `@clerk/nextjs/server`. Guards `/dashboard(.*)` and `/admin(.*)` routes. Checks both `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` and `CLERK_SECRET_KEY` before activating. |
| `components/auth-provider.tsx` | 1, 4, 6–10 | Imports `ClerkProvider` from `@clerk/nextjs`. Wraps app children with `<ClerkProvider publishableKey={...}>`. Falls through to raw `{children}` if no key. |
| `lib/clerk-safe.ts` | 3, 5, 30–51 | Imports `useAuth as useClerkAuth`, `useUser` from `@clerk/nextjs`. Exports `useSafeClerkAuth()` and `useSafeClerkUser()` — both fail-safe wrappers that return noop objects if Clerk isn't configured. |
| `hooks/useAuth.ts` | 15–70 | Imports from `@/lib/clerk-safe`. Calls `useSafeClerkUser()` to get `clerkUser` (for email, name, id) and `useSafeClerkAuth()` to get `getToken()`. The JWT from `getToken()` is passed as a Bearer token to the backend. |
| `store/authStore.ts` | 20 | The `User` interface has a `clerkId: string` field. |
| `components/dashboard-section.tsx` | 5, 62 | Imports and calls `useSafeClerkAuth()` to get `getToken()` which is passed to `fetchOrders()`. |
| `components/checkout-section.tsx` | 6, 42 | Imports and calls `useSafeClerkAuth()` to get `getToken()` for authenticated checkout via `createCheckout()`. |
| `app/admin/health/page.tsx` | 4, 107, 116–117 | Imports and calls `useSafeClerkAuth()` to pass token to `fetchSystemHealth()`. |
| `lib/validation.ts` | 10, 56, 73 | Comments stating "authentication handled by Clerk" — note only, no code dependency. |
| `.env.example` | 2–3 | Documents `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` and `CLERK_SECRET_KEY`. |
| `package.json` | line 16 | `"@clerk/nextjs": "^7.0.7"` as a runtime dependency. |

### 1.2 Auth UI Components / Login Injection Points

There are no dedicated login/signup page files under `app/`. Authentication pages are expected at `/sign-in` and `/sign-up` routes (referenced by links in components), but no `app/sign-in/` or `app/sign-up/` directories exist yet. These must be created.

| Where Link Appears | File | Line(s) | Target Route |
|---|---|---|---|
| "Sign in" button (not logged in) | `components/dashboard-section.tsx` | 155, 234 | `/sign-in` |
| "Save this order — Create Account" CTA | `app/checkout/success/page.tsx` | 80 | `/sign-up` |
| Navigation "Sign in" text (conditionally) | `components/navigation.tsx` | 109, 191 | `/dashboard` (redirects to sign-in if not authed) |

Files that need Supabase Auth UI logic injected:

- `app/sign-in/page.tsx` — must be created (currently a dead link). Will need a Supabase Auth UI or custom email/password form.
- `app/sign-up/page.tsx` — must be created. Same.
- `components/auth-provider.tsx` — Replace `ClerkProvider` wrapper with a Supabase `SessionContextProvider` (from `@supabase/auth-helpers-nextjs` or `@supabase/ssr`).
- `lib/clerk-safe.ts` — entire file replaced with a `supabase-safe.ts` (or `supabase-auth.ts`) that exports equivalent hooks reading from Supabase session.
- `hooks/useAuth.ts` — Replace `useSafeClerkUser()` / `useSafeClerkAuth()` with calls to `supabase.auth.getSession()` and `supabase.auth.getUser()`. The `getToken()` call maps to Supabase's `session.access_token`.
- `store/authStore.ts` — Rename `clerkId` to `supabaseId` (or keep as generic `authId`). Update `User` interface.
- `middleware.ts` — Remove Clerk middleware entirely. Replace with Supabase SSR middleware (`createMiddlewareClient` from `@supabase/ssr`) to refresh the session cookie on every request, protecting `/dashboard` and `/admin` routes.

### 1.3 Files Needing Supabase Client/Server Utilities

The frontend uses no server-side DB access directly — all data goes through `lib/api.ts` to the Express backend. The only server-side need is session management in middleware.

| New File (to create) | Purpose |
|---|---|
| `lib/supabase/client.ts` | Browser-side Supabase client (`createBrowserClient`) for client components |
| `lib/supabase/server.ts` | Server-side Supabase client (`createServerClient`) for Server Components / Route Handlers |
| `lib/supabase/middleware.ts` | Middleware client helper (`createMiddlewareClient`) used in `middleware.ts` |

---

## Phase 2 — Stripe Managed by Supabase

### 2.1 Current Stripe Presence in the Frontend

There is no Stripe SDK code in the frontend repo. The only Stripe reference is:

| File | Line | Content |
|---|---|---|
| `lib/api.ts` | 324 | `stripeClientSecret?: string; // Present if Stripe PaymentIntent created` — part of the `CheckoutResult` interface |
| `DOCS/BACKEND_ROADMAP.md` | Stage D | Documents `POST /api/checkout/intent` returning a `clientSecret`, and `POST /api/checkout/webhook` |
| `DOCS/project docs/BACKENDREPORT.md` | 164 | Backend env schema defines `STRIPE_SECRET_KEY: z.string().optional()` |
| `DOCS/project docs/backend-status-report0325.md` | 83, 180 | Backend implements `POST /api/checkout/webhook` with Stripe signature validation; frontend needs `STRIPE_SECRET_KEY` env var |

The intended integration (per `CheckoutResult.stripeClientSecret` in `lib/api.ts:324`) is:

1. Frontend calls `POST /api/checkout` (via `createCheckout()` in `lib/api.ts:327–345`)
2. Backend creates a Stripe PaymentIntent, returns `clientSecret` in response
3. Frontend uses `@stripe/stripe-js` / Stripe Elements to complete payment in the browser

### 2.2 Where Supabase User IDs Need to Link to Stripe Customers

After Phase 1 migrates auth, the backend's `POST /api/webhooks/clerk` Clerk webhook must be replaced with a Supabase Auth webhook that creates/updates a `User` row in Postgres with the Supabase `user.id`. The frontend's `lib/api.ts:401–411` (`UserProfile.clerkId`) will need to become `supabaseId`.

Frontend touchpoints for Stripe-Supabase linkage:

| File | Line(s) | Change Needed |
|---|---|---|
| `lib/api.ts` | 327–345 (`createCheckout`) | No change needed — this calls the backend which handles Stripe. If `stripeClientSecret` comes back, the frontend must mount Stripe Elements. |
| `lib/api.ts` | 403 | `clerkId: string \| null` → `supabaseId: string \| null` |
| `components/checkout-section.tsx` | 113–140 | If `result.stripeClientSecret` is returned, integrate `@stripe/react-stripe-js` (`<Elements>` + `<PaymentElement>`) here. Currently the checkout simply creates an order without triggering Stripe payment on the frontend. |
| New file: `lib/stripe/client.ts` | — | Initialize `loadStripe(NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY)` for Stripe Elements |

**Key insight:** The frontend `checkout-section.tsx` currently submits via `createCheckout()` and redirects to `/checkout/success` without any Stripe client-side payment confirmation step. The `stripeClientSecret` field exists in `CheckoutResult` but is never consumed. Full Stripe integration requires adding a `<Elements>` wrapper and a `<PaymentElement>` step before the success redirect.

---

## Phase 3 — Supabase PostgreSQL (Prisma/Seed)

### 3.1 Prisma Schema and Seed Location

These files do not exist in this frontend repo. Per `DOCS/ARCHITECTURE.md:36` and `DOCS/NEXT_STEPS.md:48`:

- `prisma/schema.prisma` → lives in the backend repo (`celltech-backend/prisma/schema.prisma`)
- `prisma/seed.ts` → `celltech-backend/prisma/seed.ts`

The frontend has no direct database connection. It communicates only via `lib/api.ts` → Express backend → Prisma → Neon PostgreSQL.

### 3.2 Database Connection Points in the Frontend

| File | Nature of DB Access |
|---|---|
| `lib/api.ts` (all functions) | Indirect only — all calls go through `NEXT_PUBLIC_API_URL` (the Express backend). No `DATABASE_URL` or Prisma client is present in this repo. |
| `middleware.ts` | After Supabase migration, will use Supabase SSR which talks to Supabase (Postgres) for session refresh — but that's Auth, not direct DB. |

The only scenario where Postgres touches the frontend directly is if you add Next.js API Route Handlers (e.g., `app/api/...`) that use Prisma or Supabase's `supabase-js` to query Postgres. Currently none exist.

---

## General Requirements

### Environment Variables — Full Inventory

**Currently in Frontend (`.env.example`)**

```
NEXT_PUBLIC_API_URL=https://ctir-backendv1-official.vercel.app
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_...
CLERK_SECRET_KEY=sk_...
```

**After All 3 Phases — Complete Frontend `.env` Required**

| Variable | Source | Used In | Required? |
|---|---|---|---|
| `NEXT_PUBLIC_API_URL` | Backend Vercel URL | `lib/api.ts:16` | ✅ Required |
| `NEXT_PUBLIC_SUPABASE_URL` | Supabase project URL | `lib/supabase/client.ts`, `lib/supabase/server.ts`, `middleware.ts` | ✅ Required (replaces Clerk) |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Supabase anon key | Same as above | ✅ Required (replaces Clerk) |
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase service role | Server-only: `lib/supabase/server.ts` for admin ops | ⚠️ Only if server-side admin queries needed |
| `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY` | Stripe dashboard | `lib/stripe/client.ts` for Stripe Elements | ✅ Required for payment UI (Phase 2) |
| `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` | Clerk | Removed in Phase 1 | ❌ Remove |
| `CLERK_SECRET_KEY` | Clerk | Removed in Phase 1 | ❌ Remove |

**Backend `.env` (for reference)**

```
DATABASE_URL          # Neon PostgreSQL connection string
DIRECT_URL            # Neon direct connection (for migrations)
JWT_SECRET            # 32+ chars (can be removed if pure Clerk/Supabase JWT)
REDIS_URL             # Upstash Redis URL
STRIPE_SECRET_KEY     # Stripe secret key
STRIPE_WEBHOOK_SECRET # For webhook signature verification
CLERK_SECRET_KEY      # Backend Clerk verification (→ SUPABASE_JWT_SECRET in Phase 1)
```

### Middleware (`middleware.ts`) — Analysis and Upstash Integration

**Current `middleware.ts` logic (`middleware.ts:1–33`):**

1. Checks if `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` and `CLERK_SECRET_KEY` are set (line 3–5)
2. If not set, passes through all requests with `NextResponse.next()` (line 8–10)
3. Dynamically imports `clerkMiddleware` and `createRouteMatcher` (line 12)
4. Protects `/dashboard(.*)` and `/admin(.*)` routes with `auth.protect()` (lines 14–23)

**After Phase 1 (Supabase), `middleware.ts` must:**

1. Create a Supabase SSR client using `createMiddlewareClient()`
2. Call `supabase.auth.getSession()` to refresh the session cookie
3. Redirect unauthenticated users away from `/dashboard` and `/admin`
4. Forward the supabase response cookies

**Upstash Redis Rate Limiting — where it should be integrated:**

Upstash (`REDIS_URL` / `@upstash/ratelimit`) belongs in the backend middleware (`celltech-backend/src/middleware/rateLimit.ts`), per:

- `DOCS/BACKEND_ROADMAP.md` Stage A-6: "Redis singleton — Upstash/ioredis, used for rate-limit store + session cache"
- `DOCS/BACKEND_ROADMAP.md` Stage C-11: "Rate limiting — Redis-backed — Auth endpoints: 10 req/15 min per IP. Cart: 60 req/min per user."

However, if you want edge-level rate limiting in the Next.js frontend middleware (e.g., to protect `/api` routes or the `/dashboard` route from abuse), you can add `@upstash/ratelimit` + `@upstash/redis` to `middleware.ts`. The required env vars would then also be needed on the frontend:

| Variable | Purpose |
|---|---|
| `UPSTASH_REDIS_REST_URL` | Upstash REST endpoint for `@upstash/redis` |
| `UPSTASH_REDIS_REST_TOKEN` | Auth token for Upstash REST |

Suggested Upstash integration point in `middleware.ts` — after session refresh, before route protection:

```typescript
// In middleware.ts — after Supabase session refresh
const ratelimit = new Ratelimit({ redis, limiter: Ratelimit.slidingWindow(10, "15m") });
const { success } = await ratelimit.limit(req.ip ?? 'unknown');
if (!success) return new NextResponse('Too Many Requests', { status: 429 });
```

### UI.md Design Alignment

Per `DOCS/UI.md`:

- All auth UI (`/sign-in`, `/sign-up`) must use the dark theme with `ct-bg` (`#070A12`) backgrounds, `ct-accent` (`#00E5C0`) for CTAs, and `ct-text` / `ct-text-secondary` for text
- Form inputs must use the `.input-dark` utility class (defined in `app/globals.css`)
- Buttons: `.btn-primary` (accent fill) and `.btn-secondary` (outlined)
- No light-theme components; no hardcoded hex values in JSX — use `ct-*` Tailwind tokens
- Font: `font-display` (Sora) for headings, `font-body` (Inter) for inputs/labels, `font-mono` (IBM Plex Mono) for SKUs/session IDs
- Cards: `dashboard-card` CSS class pattern from `globals.css`
- Animations: Framer Motion `AnimatePresence` / `motion.div` as used throughout (e.g., `components/hero-section.tsx`, `components/checkout-section.tsx`)

The existing `components/forms/FormInput.tsx`, `FormSelect.tsx`, `PasswordStrength.tsx`, and `AddressForm.tsx` are already styled correctly and can be reused directly in the new sign-in/sign-up pages.

---

## Full File Impact Table

| File | Phase | Action |
|---|---|---|
| `middleware.ts` | 1 + Upstash | Replace Clerk middleware with Supabase SSR + optional Upstash rate limit |
| `components/auth-provider.tsx` | 1 | Replace `ClerkProvider` with Supabase `SessionContextProvider` |
| `lib/clerk-safe.ts` | 1 | Delete; replace with `lib/supabase-auth.ts` |
| `hooks/useAuth.ts` | 1 | Replace Clerk hooks with Supabase `session.access_token` |
| `store/authStore.ts` | 1 | Rename `clerkId` → `supabaseId` in `User` interface |
| `lib/api.ts` (line 403) | 1 | `clerkId: string \| null` → `supabaseId: string \| null` |
| `app/sign-in/page.tsx` | 1 | Create — Supabase Auth UI or custom form |
| `app/sign-up/page.tsx` | 1 | Create — Supabase Auth UI or custom form |
| `lib/supabase/client.ts` | 1 | Create — browser Supabase client |
| `lib/supabase/server.ts` | 1 | Create — server Supabase client |
| `components/dashboard-section.tsx` | 1 | Replace `useSafeClerkAuth` with Supabase session hook |
| `components/checkout-section.tsx` | 1+2 | Replace `useSafeClerkAuth`; add Stripe Elements if `stripeClientSecret` present |
| `app/admin/health/page.tsx` | 1 | Replace `useSafeClerkAuth` with Supabase session hook |
| `lib/stripe/client.ts` | 2 | Create — `loadStripe(NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY)` |
| `lib/api.ts` (`createCheckout`) | 2 | Consume `stripeClientSecret` from response; mount Stripe Elements |
| `.env.example` | 1+2 | Replace Clerk keys with Supabase + Stripe publishable key |
| `package.json` | 1+2 | Remove `@clerk/nextjs`; add `@supabase/ssr`, `@supabase/supabase-js`, `@stripe/stripe-js`, `@stripe/react-stripe-js`, optionally `@upstash/redis`, `@upstash/ratelimit` |
