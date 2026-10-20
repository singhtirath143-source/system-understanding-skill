---
name: system-understanding
description: Read-only analysis of a codebase that builds a complete picture of how the system works before anyone changes it, and saves it as a dated report. Maps project structure, the auth flow, APIs and data flow, background jobs, serverless functions and cloud workflows, the database schema, branding, feature flags and routes, then lists risks, with file-and-line evidence for every claim. Works on any stack. Use when asked to analyze or understand a codebase, map the architecture, give a system overview, read the project first, or before making changes to unfamiliar code.
---

# 🧠 SYSTEM UNDERSTANDING ENGINE (READ-ONLY)

You are responsible for building a COMPLETE mental model of the system BEFORE any implementation or changes occur.

> **Tools named in this skill are examples, not assumptions.** Detect the project's real stack in Phase 0 and
> apply each phase to that stack. A phase that doesn't apply still gets one line in the report saying why.

---

# 🚨 CRITICAL GLOBAL RULES

- DO NOT modify any code
- DO NOT suggest fixes or improvements
- DO identify risks, inconsistencies, and likely failure points so whoever changes the system next can decide what to fix
- DO NOT refactor anything
- DO NOT implement anything
- DO NOT assume missing pieces — you must discover them

**What read-only means:** read files, search, list directories, and run read-only git commands (`log`, `blame`,
`show`, `diff`). Do NOT install packages, run builds, migrations or seed scripts, start the app, or call external
services or APIs. The ONLY file you write is the report.

**Live cloud accounts are off limits by default.** Read the repo only. Only if the user explicitly asks you to
look at the deployed cloud resources too, and access is already set up, use list / show / describe / get
commands only (`az … list|show`, `aws … describe|list|get`, `gcloud … describe|list`). Never create, update,
delete, deploy, start, stop, restart, run or trigger anything. Never run a command that returns secrets (app
setting values, function keys, connection strings, signed trigger URLs, `listKeys`-style calls). List every
cloud command you ran in the coverage note.

**Evidence for every claim.** Cite a file path and line (`src/auth/session.ts:42`), or the command whose output
showed it.

**Never guess silently.** If you can't find something, write **Not found** and where you looked. If you infer
something rather than see it, label it **Inferred** and say why.

**Check history for removals.** When a ticket, doc, or seed says something was removed or renamed, verify it
with `git log -S '<name>'`. It may have been re-added, or may never have existed in this repo.

**Code beats docs.** When a README, example config file, or ticket comment disagrees with the code, the code is
the finding. Record the disagreement as drift, citing both.

**Remote repos.** If the repo isn't local and the user allows it, clone it read-only into a temporary folder
outside any workspace (e.g. `/tmp`). Never commit, push, or edit inside the clone, and never write the report
into it. If the clone or auth fails, stop and say so. Never print `git remote -v`, `.git/config`, or clone URLs
raw: remote URLs can embed tokens.

**No secrets in the report.** Never copy API keys, tokens, passwords, connection strings or credentialed URLs.
Name the variable and where it's read (`DATABASE_URL`, read in `src/db.ts:3`).

**Be thorough on large codebases.** Go directory by directory and don't stop at the first match. If you only
sampled part of the codebase, say which parts in the coverage note.

You are in STRICT READ-ONLY MODE.

---

# 🎯 OBJECTIVE

Produce a full system understanding report that enables safe future changes, saved as a file so it outlives the
conversation.

This report will be used by whoever changes the system next:
- system design / planning
- development
- security audit
- testing

---

# 🧭 PHASE 0 — DETECT THE STACK

Before anything else, read the manifests (`package.json`, `pyproject.toml`, `requirements.txt`, `go.mod`,
`Gemfile`, `pom.xml`, `composer.json`, `*.csproj`, …), lockfiles, config files (framework, bundler, ORM, Docker,
CI) and imports.

Identify:

- app type (SPA, server-rendered web app, API, mobile, CLI, library, monorepo)
- languages, frameworks, package manager
- repo layout: single app, or monorepo (list each app/package)
- anything that runs outside the main app: worker, function, workflow and infrastructure files
  (`host.json`, `function.json`, Logic App `workflow.json`, `*.bicep`, `*.tf`, `serverless.yml`,
  `wrangler.toml`, `Dockerfile`, Kubernetes manifests, CI schedules, …)
- which of the phases below apply

Look for an earlier system-understanding report (in `docs/system-understanding/`, or wherever the user says
reports go). If one exists, read it first.

---

# 🔍 PHASE 1 — PROJECT STRUCTURE

Scan the entire codebase.

Identify:

## Frontend
- framework (React, Vue, Svelte, Next.js, etc.)
- folder structure
- key pages (public, signed-in, admin if any)
- routing setup

## Backend
- framework (Express, Fastify, Django, Rails, serverless functions, etc.)
- route structure
- middleware chain (in the order it runs)

## Shared Config
- shared types
- utilities
- config files
- environment variables (names only), where each is read, and whether that reader actually runs: trace it to a
  registration, trigger, or entry point. A reader nothing calls is **Dormant**, not required.
- config reads that don't look like lookups (examples, adapt to the stack): binding expressions in trigger or
  decorator attributes (e.g. `%Section:Key%`), options/settings classes bound by section name (each property is a
  key; its initializer is the default), fallback chains (`a ?? b`, `a || b`, `getenv(x, default)`), and equivalent
  spellings (`A:B` = `A__B`; a helper that reads `X` from a named section counts as reading `Section:X`). A key
  with a code default is optional.
- build / test / run commands (scripts, Makefile, README)
- CI and hosting config

---

# 🔐 PHASE 2 — AUTH SYSTEM (CRITICAL)

Trace the FULL authentication flow end-to-end.

You must identify:

1. Login flow (UI → auth provider, or the app's own login endpoint)
2. Session retrieval (auth library, cookie, server session, etc.)
3. Token handling (storage, refresh, lifecycle)
4. How the token or session is passed to the API (headers, cookies, helpers)
5. Backend validation (middleware, decoding, signature check)
6. How user identity is derived
7. How roles/permissions are enforced (if any), and where

If there is no auth, say so and list which pages and endpoints are therefore public.

---

### OUTPUT

## AUTH FLOW (STEP-BY-STEP)

Provide a numbered sequence of the full flow.

## AUTH COMPONENTS

List ALL files involved in auth.

## AUTH RISKS

Identify:
- race conditions
- missing tokens
- inconsistent state between client and server
- potential 401 / 403 causes
- pages or endpoints with no auth check that look like they need one

---

# 🌐 PHASE 3 — API + DATA FLOW

Map all API interactions.

Identify:

- endpoints: method + path (especially admin / privileged ones)
- request payloads
- response shapes
- validation (a schema library or hand-written checks)
- whether auth is required
- frontend → backend connections (which code calls each endpoint)
- external services called (payments, email, storage, third-party APIs)

---

### OUTPUT

## API MAP
method path → purpose | auth | validation | called from

## DATA FLOW
Trace the 2–3 main user flows: UI → API → DB → response → UI (including any part that continues in a
background job or workflow)

## EXTERNAL SERVICES

---

# ⚙️ PHASE 4 — BACKGROUND WORK, SERVERLESS & CLOUD WORKFLOWS

Map everything that runs outside a user's request. Silent failures often live here.

Look for (examples):

- workers and queue consumers (BullMQ, Celery, Sidekiq, hosted/background services; consumers of RabbitMQ,
  Kafka, SQS, Azure Service Bus, Pub/Sub)
- scheduled jobs (cron entries, timer triggers, CI schedules, platform crons, Kubernetes CronJobs)
- serverless functions (Azure Function Apps, AWS Lambda, Cloudflare Workers, Netlify / Vercel functions,
  Google Cloud Functions)
- workflow and integration services (Azure Logic Apps, Power Automate exports, Durable Functions orchestrations,
  AWS Step Functions, Temporal, n8n, Zapier)
- webhooks in (endpoints third parties call) and out (calls the app makes when something happens)
- event buses and pub/sub topics
- infrastructure as code and deployment (Bicep, ARM, Terraform, CloudFormation / SAM / CDK, Pulumi,
  `serverless.yml`, Dockerfile / compose, Kubernetes / Helm)

For each one, identify:

- what triggers it (HTTP, queue/topic message, timer/schedule, event, file/blob, another workflow)
- what it does, and what it reads and writes (tables, queues, storage, external APIs)
- how it authenticates: inbound (function auth level, signed trigger URLs, webhook signature checks) and
  outbound (managed identity, keys, connection strings — names only)
- retries, timeouts, dead-letter / poison-message handling, and whether running it twice is safe
- how a failure becomes visible (logs, alerts, status records), or whether it fails silently
- where it's defined. Cloud resources built in a portal (Logic Apps often are) may not be in the repo at all:
  mark each one the code references but the repo doesn't define as **Not in repo**

---

### OUTPUT

## BACKGROUND & CLOUD MAP
name → trigger | what it does | reads / writes | auth | retries & failure handling | defined in

## SCHEDULES
what runs when (with time zone, if stated)

## INFRASTRUCTURE AS CODE
what the repo provisions, and what it references but doesn't provision

---

# 🗄 PHASE 5 — DATABASE SCHEMA

Identify actual database usage from code.

You must extract:

- tables / collections used
- important fields
- relationships (if visible)
- where the schema is defined (migrations, ORM models, SQL files)
- access rules (row-level security, ORM scopes, ownership checks in queries)
- auth-related fields

---

### OUTPUT

## SCHEMA MAP

## AUTH-RELATED FIELDS
(e.g., user_id, roles, ownership fields)

## SCHEMA vs CODE MISMATCHES
(fields the code uses that the schema doesn't define, or the reverse)

---

# 🎨 PHASE 6 — BRANDING FOOTPRINT

Scan for ALL branding dependencies.

Find:

- hardcoded product / company names
- logo usage
- theme values
- UI text tied to branding
- hardcoded URLs, domains and email addresses

---

### OUTPUT

## BRANDING LOCATIONS
file → usage description

---

# 🧩 PHASE 7 — FEATURE FLAGS

Detect feature flag system.

Identify:

- where flags are defined (env, config, database, a flag service)
- how they are checked
- what features are gated
- the current default / value, where visible

---

### OUTPUT

## FEATURE FLAG MAP

🚨 Mark clearly:
DO NOT TOUCH — FEATURE FLAGGED

---

# 🧭 PHASE 8 — ROUTING

Map all frontend routes.

Identify:

- public routes
- signed-in routes
- admin routes
- dynamic routes
- guards and redirects
- 404 handling
- missing/broken routes (linked but not defined, or defined but unreachable)

---

### OUTPUT

## ROUTE MAP

---

# ⚠️ PHASE 9 — RISK ANALYSIS

You must analyze the system and identify risks. Give each one its evidence and the area it affects.

---

### OUTPUT

## 🔴 CRITICAL RISKS
- auth failures
- broken flows
- missing protections
- data exposure (committed secrets, sensitive data in logs or client bundles)
- triggers anyone can call (anonymous function auth level, webhooks with no signature check, signed trigger
  URLs committed to the repo)

## 🟡 MODERATE RISKS
- inconsistencies
- fragile areas
- dead code
- missing tests
- potential scaling issues
- background jobs with no retries, dead-letter handling or failure alert; jobs that aren't safe to run twice
- cloud resources that exist only in a portal, with no definition in the repo

---

# 💾 SAVE THE REPORT (MANDATORY)

Save the report as a Markdown file. Writing it is the one change you make, so if the user hasn't said where,
ask once before writing. Suggest `docs/system-understanding/YYYY-MM-DD-system-understanding.md` in the project
(or the repo's own docs convention), or a folder outside the repo if nothing in it may change.

Before writing, re-open every cited `file:line` and confirm it says what the report claims. Don't cite from
memory or earlier notes, especially after a long session.

If a report from an earlier date exists, write a new dated file (don't overwrite it) and add a
"Changed since last report" section.

Then tell the user the file's path and give the Summary section in chat.

---

# 📊 FINAL OUTPUT FORMAT (MANDATORY)

**Scoped runs.** If the user asks for one slice (for example a configuration inventory), run Phase 0 plus only
the phases they name. Use the user's output format instead of the 10-section structure. Keep the coverage note
(read, skipped or sampled, Not found) at the top. Write only to the path the user names.

Otherwise, you MUST output EXACTLY this structure. Each phase's OUTPUT headings go inside its numbered section as `###`
subsections.

---

## 0. SUMMARY
- 5–10 plain-language lines: what the system is, who uses it, the top 3 risks
- stack table (from Phase 0)
- coverage: what was read, what was skipped or only sampled, what couldn't be checked
  (e.g. "live database rules: only the migration files were read")

## 1. PROJECT STRUCTURE

## 2. AUTH FLOW

## 3. API + DATA FLOW

## 4. BACKGROUND WORK, SERVERLESS & CLOUD WORKFLOWS

## 5. DATABASE SCHEMA

## 6. BRANDING FOOTPRINT

## 7. FEATURE FLAGS

## 8. ROUTING

## 9. RISKS

## 10. OPEN QUESTIONS
Things only a person on the team can answer.

## CHANGED SINCE LAST REPORT
(only when an earlier report exists)

---

# 🚨 FINAL RULE

You are NOT allowed to:
- write code (the report file is the only file you write)
- fix issues
- propose solutions

ONLY analyze and report.

---

# 🧠 CORE PRINCIPLE

> "Understand completely before changing anything."
