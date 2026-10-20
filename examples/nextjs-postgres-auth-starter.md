# System understanding: nextjs-postgres-auth-starter

> **Example report.** Produced by running this repo's `SKILL.md` on a public open-source starter,
> [`vercel/nextjs-postgres-auth-starter`](https://github.com/vercel/nextjs-postgres-auth-starter) at commit
> `fde8ecf` (2024-06-23). Read-only: nothing was installed, built, run or changed. Line numbers refer to that
> commit. The findings describe a small starter template; they aren't a judgment of its authors.

**Date:** 2026-10-19

---

## 0. SUMMARY

A minimal web app that shows email + password sign-up and sign-in. A visitor registers, signs in, sees a
protected page with their email address, and signs out. There are no roles, no admin area and no other
features. All server logic runs inside Next.js (server actions plus the auth library's route handler) against
a single Postgres table.

**Top 3 risks**
1. **Duplicate accounts are possible.** The `User` table has no unique constraint on `email`, and sign-up
   checks then inserts in two separate steps (`app/db.ts:36-40`, `app/register/page.tsx:12-17`).
2. **No brute-force protection on sign-in.** No rate limiting or lockout anywhere (Not found; see §9).
3. **The protected page relies on middleware alone.** `app/protected/page.tsx` renders even when there's no
   session; only `middleware.ts` keeps signed-out visitors away.

| Area | Stack | Evidence |
|---|---|---|
| App type | Server-rendered web app, single app | `app/` directory, no workspaces in `package.json` |
| Framework | Next.js 14.0.4, App Router | `package.json:16`, resolved in `pnpm-lock.yaml:5204` |
| Language | TypeScript (strict) | `tsconfig.json:15` |
| Package manager | pnpm (lockfile v6) | `pnpm-lock.yaml:1` |
| Auth | NextAuth (Auth.js) 5.0.0-beta.4, Credentials provider | `package.json:17`, `app/auth.ts:2,15` |
| Database | Postgres via the `postgres` driver + Drizzle ORM 0.29 | `app/db.ts:1-4,10-11` |
| Password hashing | bcrypt-ts, cost 10 | `app/db.ts:5,20-21` |
| Styling | Tailwind CSS 3, Geist font | `tailwind.config.ts`, `app/layout.tsx:3` |
| Hosting | Vercel (Inferred: README deploy button, `.env.example` comments) | `README.md:9`, `.env.example:1` |

**Coverage.** Read in full: all 22 text files (585 lines). The lockfile was read only for resolved versions.
The clone is shallow, so git history is one commit. **Not checked:** runtime behaviour (the app wasn't run and
no database was connected), and the auth library's internal defaults (session strategy, cookie settings, token
contents). Those are marked **Inferred** where they matter.

---

## 1. PROJECT STRUCTURE

### Frontend
- **Framework:** Next.js App Router. Every page is in `app/`.
- **Pages:** `/` (`app/page.tsx`), `/login` (`app/login/page.tsx`), `/register` (`app/register/page.tsx`) and
  `/protected` (`app/protected/page.tsx`).
- **Shared components:** `app/form.tsx` (the email + password fields) and `app/submit-button.tsx` (a client
  component that uses `useFormStatus`, `app/submit-button.tsx:1,6`).
- **Routing setup:** file-based. Access control lives in `middleware.ts` (see §8).

### Backend
- **No separate backend.** The server-side code is:
  - server actions for sign-in (`app/login/page.tsx:17-24`), sign-up (`app/register/page.tsx:8-20`) and
    sign-out (`app/protected/page.tsx:19-22`)
  - the auth library's route handler at `/api/auth/*` (`app/api/auth/[...nextauth]/route.ts:1`, exported
    from `app/auth.ts:8`)
- **Middleware chain:** one middleware, `middleware.ts:4`, which runs the auth library's `auth` with the
  `authorized` callback from `app/auth.config.ts:12-24`. Its matcher excludes `api`, `_next/static`,
  `_next/image` and `*.png` (`middleware.ts:8`).

### Shared config
- **Imports:** non-relative, from the repo root (`'app/db'`), via `baseUrl: "."` (`tsconfig.json:7`). The
  `paths` aliases (`tsconfig.json:8-14`) point to `components/`, `pages/`, `lib/` and `styles/`, none of which
  exist (dead config).
- **Environment variables (names only):**

  | Variable | Where it's read |
  |---|---|
  | `POSTGRES_URL` | `app/db.ts:10` |
  | `AUTH_SECRET` | Never read in this code; read by the auth library by convention (Inferred). Listed at `.env.example:7`. |
  | `POSTGRES_PRISMA_URL`, `POSTGRES_URL_NON_POOLING` | Listed at `.env.example:3-4`, never read anywhere (grep) |

- **Commands:** `dev`, `build`, `start` and `lint` (`package.json:4-7`). There is no `test` script.
- **CI / hosting config:** Not found. Looked for `.github/`, `vercel.json` and a `Dockerfile`.
- **Secrets:** none committed. `.env*` files are git-ignored (`.gitignore:29-33,37`), and `.env.example`
  holds empty values only.

---

## 2. AUTH FLOW

1. **Sign-up.** The form posts email + password to a server action (`app/register/page.tsx:8-11,31`). It looks
   the email up (`:12`). If none exists, it stores a bcrypt hash (`app/db.ts:18-24`) and redirects to `/login`
   (`:17-18`).
2. **Sign-in.** The form's server action calls `signIn('credentials', { email, password, redirectTo:
   '/protected' })` (`app/login/page.tsx:17-24`).
3. **Credential check.** `authorize` loads the user by email (`app/auth.ts:17`), returns `null` if there is no
   row (`:18`) and compares the bcrypt hash (`:19`). On a match it returns the **whole database row**,
   password hash included (`:20`). On a mismatch it falls through and returns `undefined` (`:20-21`).
4. **Session.** No database adapter or `session` option is configured (Not found in `app/auth.config.ts` or
   `app/auth.ts`). So the library default applies, which is a signed JWT in a cookie (Inferred). Expiry and
   refresh are also library defaults (no `maxAge` set).
5. **Passing the session to the server.** The browser sends the cookie automatically. There are no custom API
   calls or auth headers.
6. **Server-side check.** `middleware.ts:4` runs `authorized` (`app/auth.config.ts:12-24`):
   - `/protected…` requires a user; otherwise the visitor is sent to the sign-in page, `/login`
     (`app/auth.config.ts:5,16-18`)
   - a signed-in user on any other matched path is redirected to `/protected` (`:19-21`)
7. **Identity.** `auth()` on the protected page reads `session.user.email` (`app/protected/page.tsx:4,9`).
8. **Roles and permissions.** None. There is no role field (`app/db.ts:36-40`) and no role check anywhere.
9. **Sign-out.** A server action calls `signOut()` (`app/protected/page.tsx:16-27`).

### AUTH COMPONENTS
`middleware.ts` · `app/auth.config.ts` · `app/auth.ts` · `app/api/auth/[...nextauth]/route.ts` ·
`app/db.ts` (`getUser`, `createUser`) · `app/login/page.tsx` · `app/register/page.tsx` ·
`app/protected/page.tsx` · `app/form.tsx`

### AUTH RISKS
- **Race condition at sign-up.** The lookup and the insert are separate steps, and the database has no unique
  constraint, so two simultaneous sign-ups can create two rows with the same email. Sign-in then uses whichever
  row comes first (`app/auth.ts:19-20`).
- **No server-side input checks.** `formData.get(...) as string` (`app/login/page.tsx:21-22`,
  `app/register/page.tsx:10-11`) trusts the browser's `required` / `type="email"` (`app/form.tsx:23-26,41`).
  A request sent without the form can pass `null`, or an email over 64 characters. The column is
  `VARCHAR(64)` (`app/db.ts:38`), so Postgres would reject the insert with an unhandled error (Inferred).
- **A null password crashes sign-in.** The `password` column is nullable (`app/db.ts:39`) and is non-null
  asserted at `app/auth.ts:19`. A row without a hash would make `compare` fail with an error instead of a
  clean "wrong password" (Inferred).
- **The password hash is handed to the auth library** (`app/auth.ts:20`). Whether it ends up in the session
  token depends on the library's default JWT callback, which normally copies only name, email, picture and id
  (Inferred, not verified against the library source). See §10.
- **No visible error on failed sign-in.** The login page reads no error parameter (`app/login/page.tsx`) and
  never tells the user why sign-in failed.
- **Unprotected API routes.** The middleware matcher excludes `api` (`middleware.ts:8`). Today that only skips
  the auth library's own routes, but any API route added later is unprotected unless it checks the session
  itself.

---

## 3. API + DATA FLOW

### API MAP
| Method + path | Purpose | Auth | Validation | Called from |
|---|---|---|---|---|
| `GET/POST /api/auth/*` | The auth library's handlers (session, sign-in callback, sign-out, CSRF; Inferred: defined by the library, not in this repo) | Library-managed | Library-managed | The library's client and server helpers |
| Server action: sign-in | Check credentials, start a session | None (public) | None server-side | `app/login/page.tsx:17-24` |
| Server action: sign-up | Create a user | None (public) | None server-side; existence check only | `app/register/page.tsx:8-20` |
| Server action: sign-out | End the session | Session cookie | — | `app/protected/page.tsx:19-22` |

### DATA FLOW
- **Sign-up:** form (`app/form.tsx`) → server action (`app/register/page.tsx:8`) → `getUser` →
  `ensureTableExists` + `SELECT` (`app/db.ts:13-15,26-32`) → `createUser` → `INSERT` (`app/db.ts:18-23`) →
  redirect to `/login`. If the email exists, the action returns `'User already exists'`
  (`app/register/page.tsx:15`). Nothing renders that value, so the user sees no message (the line carries a
  `TODO`).
- **Sign-in:** form → `signIn` (`app/login/page.tsx:19`) → `authorize` → `getUser` + bcrypt `compare`
  (`app/auth.ts:16-21`) → session cookie → redirect to `/protected`.
- **Protected page:** request → middleware `authorized` (`app/auth.config.ts:12-24`) → `auth()` → renders
  the email (`app/protected/page.tsx:4-9`).

### EXTERNAL SERVICES
- **Postgres** at `POSTGRES_URL`, with `?sslmode=require` appended (`app/db.ts:10`).
- None other. No email, payments, storage or analytics. The page links to nextjs.org, next-auth.js.org and
  vercel.com, but those are navigation links, not API calls (`app/page.tsx:28,37,46,65`).

---

## 4. BACKGROUND WORK, SERVERLESS & CLOUD WORKFLOWS

**None found.** There are no workers, queue consumers, scheduled jobs, serverless function definitions,
workflow or integration services, webhooks, event buses or infrastructure-as-code. I looked for `vercel.json`
crons, `.github/` schedules, `host.json` / `function.json`, Logic App or Step Functions definitions, `*.bicep`,
`*.tf`, `serverless.yml`, `wrangler.toml`, a `Dockerfile` and Kubernetes manifests, and checked the
dependencies (`package.json:9-29`) for queue or job libraries. The only work outside a page request is the auth
library's route handler (§3).

**Inferred:** on Vercel, each server action and route handler is deployed as a serverless function by the
platform. Nothing in the repo configures that.

---

## 5. DATABASE SCHEMA

### SCHEMA MAP
| Table | Column | Type | Notes |
|---|---|---|---|
| `"User"` | `id` | `SERIAL PRIMARY KEY` | `app/db.ts:37,44` |
| | `email` | `VARCHAR(64)` | Nullable; **not unique** (`app/db.ts:38,45`) |
| | `password` | `VARCHAR(64)` | bcrypt hash (60 characters, so it fits); nullable (`app/db.ts:39,46`) |

- **Schema source:** there are no migrations and no Drizzle config (Not found: looked for `drizzle.config.*`,
  `drizzle/` and `migrations/`). The table is created **at runtime** by `ensureTableExists`
  (`app/db.ts:26-41`), which runs before every `getUser` and `createUser` call (`:14,19`).
- **Access rules:** none in the database. The app connects with the one role in `POSTGRES_URL`. There's no
  row-level security, and ownership doesn't apply (users only ever read their own row through sign-in).

### AUTH-RELATED FIELDS
`email` (the login identifier) and `password` (the bcrypt hash). There is no role, status or verified-email
field.

### SCHEMA vs CODE MISMATCHES
- The code treats `email` as unique (sign-up checks it; sign-in uses `user[0]`), but the database doesn't
  enforce it (`app/db.ts:38`).
- The code treats `password` as always set (`app/auth.ts:19` uses `!`), but the column is nullable
  (`app/db.ts:39`).
- `.env.example` and the README's deploy link mention Prisma (`.env.example:3`, `README.md:9`), but the app
  uses Drizzle (`app/db.ts:1`).

---

## 6. BRANDING FOOTPRINT

| File | Usage |
|---|---|
| `app/page.tsx:7-20` | Inline Vercel logo SVG (`aria-label="Vercel logo"`) |
| `app/page.tsx:23`, `app/layout.tsx:5-7` | Product name and description: "Next.js + Postgres Auth Starter" |
| `app/layout.tsx:17` | `metadataBase` hardcoded to `https://nextjs-postgres-auth.vercel.app` |
| `app/page.tsx:28,37,46,65` | Hardcoded links to nextjs.org, next-auth.js.org, vercel.com/postgres and a Vercel template page |
| `app/form.tsx:24` | Placeholder email `user@acme.com` |
| `README.md:3,9` | Neon / Vercel Postgres; the deploy link describes a Prisma version |
| `tailwind.config.ts:5` | Default theme (empty `theme`); Geist font (`app/layout.tsx:3,27`) |

`tailwind.config.ts:4` scans `./content/**/*.mdx` and `./public/**/*.svg`, but neither folder exists.

---

## 7. FEATURE FLAGS

**None found.** There's no flag config, flag service or flag-like environment variable. I searched for
"flag", "feature" and "experiment", and the only `process.env` read is `POSTGRES_URL` (`app/db.ts:10`).

---

## 8. ROUTING

### ROUTE MAP
| Route | Type | Signed-out visitor | Signed-in user |
|---|---|---|---|
| `/` | Public | Sees the landing page | Redirected to `/protected` (`app/auth.config.ts:19-21`) |
| `/login` | Public | Sees the sign-in form | Redirected to `/protected` |
| `/register` | Public | Sees the sign-up form | Redirected to `/protected` |
| `/protected` | Signed-in only | Sent to `/login` (`app/auth.config.ts:5,16-18`) | Sees their email + sign-out |
| `/api/auth/*` | Dynamic catch-all (`[...nextauth]`) | Not matched by the middleware (`middleware.ts:8`) | — |

- **Admin routes:** none.
- **404s:** there is no `not-found` page, so Next's default 404 applies (Inferred). A signed-in user on an
  unknown path is redirected to `/protected` before any 404, because the callback redirects every matched path
  except `/protected` (`app/auth.config.ts:19-21`).
- **Broken links:** none. Every internal link (`/protected`, `/login`, `/register`) points to an existing page.
- **Likely side effect (Inferred, not run):** `/favicon.ico` is not excluded by the matcher (`middleware.ts:8`
  excludes only `.png`). So a signed-in user's favicon request would get the redirect to `/protected` instead
  of the icon.

---

## 9. RISKS

### 🔴 CRITICAL RISKS
- **Duplicate accounts per email.** There's no unique constraint, and sign-up checks then inserts in separate
  steps. Sign-in picks the first row. Area: auth, data. Evidence: `app/db.ts:38`,
  `app/register/page.tsx:12-17`, `app/auth.ts:19-20`.
- **No brute-force protection.** Nothing limits sign-in or sign-up attempts. Area: auth. Evidence: Not found
  (searched for "rate" and "limit"; no middleware or library for it in `package.json`).
- **The protected page trusts middleware alone.** `app/protected/page.tsx:4-9` renders without checking that
  the session exists. If the matcher changes or a path slips past it, the page renders for anyone. Area: auth,
  routing.

### 🟡 MODERATE RISKS
- **No server-side input validation**, so bad input surfaces as unhandled database errors (§2). Area: API.
- **Sign-up and sign-in failures show nothing to the user** (`app/register/page.tsx:15`,
  `app/login/page.tsx`). Area: UX, broken flow.
- **Runtime DDL on every auth call.** `ensureTableExists` adds a query to every sign-in and sign-up, creates a
  table at runtime, and two first-ever calls at once could both try `CREATE TABLE` (`app/db.ts:14,19,26-41`).
  Area: database, scaling.
- **A fragile connection string.** `?sslmode=require` is appended blindly (`app/db.ts:10`). If `POSTGRES_URL`
  already has a query string, the result is malformed (Inferred). Area: config.
- **Pinned pre-release auth library**, `next-auth@5.0.0-beta.4` (`package.json:17`). Area: dependencies.
- **Dead and stale config:** `tsconfig.json` path aliases to missing folders, Tailwind globs for missing folders,
  two unused env vars, and Prisma wording in the README and `.env.example`. Area: maintainability.
- **No tests and no CI** (no `test` script, no `.github/`). Area: quality.

---

## 10. OPEN QUESTIONS

1. Is this deployed anywhere with real users, or is it used only as a template?
2. `.env.example:6` says `AUTH_SECRET` is "only required for localhost". Is it set in every deployed
   environment?
3. Should `email` be unique at the database level, and should it be case-insensitive?
4. What session lifetime is intended? Nothing overrides the library default.
5. Does the database hold anything else? `ensureTableExists` creates `"User"` in the `public` schema at
   runtime.
6. Has anyone confirmed the password hash never reaches the session token (§2)?
