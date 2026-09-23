# Humanitarian Aid Platform — Stages 1, 2, and crypto donations

Next.js + Supabase donation platform for an international humanitarian charity.

- **Stage 1 (`supabase/`)**: database, security model, storage, tests, CI.
- **Stage 2 (`web/`)**: the Next.js app: public site, auth, donor dashboard, admin gate.
- **Crypto donations (out of order, on request)**: Coinbase Commerce and Binance Pay are wired up end to end — donor form, charge creation, signed webhook, receipt. This normally would have waited for Stage 3 (card payments), built first for broader coverage; it's the only payment rail that currently works.
- **Still not built:** card/PayPal/Apple Pay/Google Pay/bank transfer/mobile money (Stage 3/5), admin management screens (Stage 4), image service and media library UI (Stage 6). `/donate` says plainly which methods aren't switched on yet.

> **Verification status, stated honestly.** No Postgres, npm registry or browser was available where this was written.
> - **Run and passing:** a syntax check of all 77 TypeScript/TSX files, a check that every internal import resolves, 12 executed tests of money formatting/redirect-guard/the homepage ledger, and **11 executed tests of the real Coinbase and Binance signature-verification code** — using a generated HMAC secret and a generated RSA keypair, including tamper-detection cases (flipped status, altered timestamp, wrong key). Run it yourself: `cd web && npm run test:crypto-signatures`.
> - **Written but never executed:** all SQL and its 33 pgTAP tests; the full Next.js app (never installed, type-checked against Next/Supabase types, built, or opened in a browser); no real API call has ever been made to Coinbase Commerce or Binance Pay — the HTTP request shapes are written from their published API docs but unverified against the live services.
>
> Push the repo and let **Database CI** and **Web CI** run on GitHub. Expect a few small fixes on first run, especially TypeScript type errors and SQL details.

## Before you enable Binance Pay: read this

Binance Pay merchant services are **not available to donors or merchants in every country** —
notably, Binance has excluded US persons from many of its products, and Binance itself has been
subject to significant regulatory action in multiple jurisdictions (including a 2023 US DOJ/Treasury
settlement). For a charity taking donations internationally, that means:
- Some donors simply won't be able to pay this way, and the donor form says so.
- You should get your own legal/compliance advice on accepting Binance Pay for a nonprofit
  before turning it on in production, separately from the general "before production launch"
  checklist at the bottom of this file.
- Coinbase Commerce has broader, more consistent availability and a simpler compliance footprint
  for a merchant (you never take custody of keys either way), which is why it's listed first
  and enabled by default in `site_settings.crypto_payments`.

Both are OFF by nothing more than a missing API key: leave `COINBASE_COMMERCE_API_KEY` /
`BINANCE_PAY_API_KEY` unset in your environment and that option's requests will fail closed
(`ProviderConfigError`), not silently succeed.

## What Stage 1 gives you

| Area | What is in place |
|---|---|
| Schema | All 22 tables from the spec, plus `media_assets` (image doc) and `campaign_categories` (admin-configurable); UUID keys, FKs, indexes, constraints, money in integer minor units |
| Auth model | `profiles` auto-created on signup, 5 roles, suspension, role changes only through audited functions |
| RLS | Enabled on every table; least-privilege grants (column-level where needed) |
| Payment safety | Only `service_role` can mark donations succeeded/refunded; campaign totals are recomputed from confirmed donations; duplicate webhooks are idempotent; verified amounts are immutable |
| Audit | Append-only `admin_audit_logs`; triggers on campaigns, donations, media, settings, reports, beneficiaries; explicit entries for role and suspension changes |
| Storage | 4 buckets (2 public, 2 private) with role-based policies; receipts readable only by their owner |
| Honesty by design | Impact figures come from `site_settings.verified_impact` (null until an admin enters verified data); demo content is flagged `is_demo` and contains no fake donations or figures |
| Tests + CI | 33 pgTAP assertions covering the rules above, run on every push |

## Repository layout

```
supabase/
  config.toml                         local dev config (email confirmation on)
  migrations/
    ..0001_schema.sql                 tables, enums, indexes
    ..0002_functions_triggers.sql     role checks, audit, workflow, payment state machine
    ..0003_rls_and_grants.sql         RLS policies + grants
    ..0004_storage.sql                buckets + storage policies
    ..0005_reference_data.sql         roles, categories, currencies, countries, settings
  seed.sql                            [DEMO] campaigns (dev only)
  tests/security.test.sql             pgTAP tests
web/                                  Next.js 15 app (App Router, TypeScript, Tailwind)
  app/                                pages, server actions, auth callback
  components/                         header, footer, forms, campaign card, image component
  lib/                                Supabase clients, data access, formatting, validation
  middleware.ts                       session refresh + protects /dashboard and /admin
.github/workflows/database-ci.yml     applies migrations + runs tests on GitHub
.github/workflows/web-ci.yml          installs, type-checks and builds the web app
.env.example                          every variable, with server-only ones marked
```

## Run the web app locally

```bash
supabase start                       # prints API URL and anon key
cd web
cp ../.env.example .env.local        # set NEXT_PUBLIC_SUPABASE_URL and NEXT_PUBLIC_SUPABASE_ANON_KEY
npm install
npm run dev                          # http://localhost:3000
```

Local sign-up emails are caught by the Supabase Inbucket mail viewer (URL shown by `supabase status`).
Email templates must use the default `{{ .ConfirmationURL }}` link so the PKCE code reaches `/auth/callback`.

What you can try: browse `[DEMO]` campaigns, filter and search, sign up and verify, edit your profile and
notification preferences, submit the contact and safeguarding forms, and (after granting yourself a role
with the SQL below) open `/admin`.

## Put it on GitHub

```bash
unzip humanitarian-platform.zip && cd humanitarian-platform
git remote add origin git@github.com:<you>/<repo>.git     # create an empty private repo first
git push -u origin main
```

Then open the **Actions** tab. GitHub runs the database CI; that is the "it runs on GitHub"
part. GitHub hosts code and CI, not the finished site. The website itself will be deployed to
a host such as Vercel, with Supabase as the backend (Stage 2 onward).

## Run locally

Requires Docker and the [Supabase CLI](https://supabase.com/docs/guides/cli).

```bash
supabase start        # boots Postgres, Auth, Storage; applies migrations + seed
supabase test db      # runs supabase/tests
supabase db reset     # rebuild from scratch
```

## Apply to a hosted Supabase project

1. Create a project at supabase.com (choose a region close to your donors; note the DB password).
2. `supabase link --project-ref <ref>` then `supabase db push` (**do not** run `seed.sql` on production).
3. Auth → Providers: keep **Email** on with **Confirm email** enabled. Add Google/phone later if wanted.
4. Auth → URL configuration: set the Site URL and redirect URLs to your deployed domain.

## Create the first super administrator

Sign up through the app (or Auth → Users → Add user), then in the SQL editor:

```sql
insert into public.user_roles (user_id, role)
select id, 'super_admin' from auth.users where email = 'you@example.org'
on conflict do nothing;
```

Use a dedicated admin account with MFA. This is the only role assignment done by raw SQL;
every later change goes through `admin_grant_role` / `admin_revoke_role` and is audit-logged.

## Roles

| Role | Can do |
|---|---|
| `user` | Read/update own profile, read own donations, receipts, recurring donations, notifications |
| `moderator` | Read user list, suspend regular users, read contact and safeguarding messages |
| `finance_admin` | Read all donations, payments, webhook events, financial reports; private documents |
| `campaign_admin` | Manage campaigns, updates, media, impact reports, beneficiaries; approve other admins' campaigns |
| `super_admin` | Everything, plus roles, settings, audit log, newsletter list |

Suspending a user must also ban them in Supabase Auth from the calling Edge Function
(`auth.admin.updateUserById(id, { ban_duration })`); the database flag alone removes staff powers
but does not end an existing session.

## Design decisions you should know about

1. **Money is `bigint` minor units.** `2500` USD = $25.00. Never floats.
2. **Campaign totals count a donation only when its settled currency equals the campaign currency.**
   Otherwise `mark_donation_succeeded` raises `currency_mismatch` and the webhook handler records it
   for manual review, so foreign-currency amounts are never silently added. The Stage 3 checkout will
   therefore charge in the campaign's currency (or use provider-reported settlement). Decide which you want.
3. **Full refunds only.** Partial refunds need a refund ledger; not included yet.
4. **Contact and newsletter forms** go through `submit_contact_message()` / `subscribe_newsletter()`
   so anonymous users never get direct table access and can't discover which emails are subscribed.
   Rate limiting belongs in the Edge Function/WAF in front of them.
5. **Guest checkout is possible.** `donations.user_id` is nullable. The spec requires login before the
   platform; I'd recommend allowing guest donations with an optional account, since forced sign-up
   typically costs donations. It is a UI/Edge Function choice, not a schema change.
6. **Wallet tables exist but are disabled** (`wallet_enabled=false`). They are a donation-credit ledger,
   not a crypto wallet, and should stay off unless the legal and payment infrastructure exists.
7. **No secrets in the database.** Provider credentials belong in Edge Function secrets.
8. **Public buckets are public by URL.** Keep drafts and sensitive files in `private-documents`.
9. **`audit_row_change` on `donations` stores donor emails in the log.** Fine for finance audit;
   review against your privacy policy and retention rules.

## How crypto donations work

```
web/lib/crypto/
  types.ts     CryptoProvider interface (createCharge, verifyAndParseWebhook) — the
               "payment-provider abstraction layer" the original spec asked for
  coinbase.ts  Coinbase Commerce adapter: HMAC-SHA256 webhook verification
  binance.ts   Binance Pay adapter: HMAC-SHA512 request signing (outgoing),
               RSA-SHA256 webhook verification (incoming)
  index.ts     registry — add a third provider by implementing the interface and
               listing it here; nothing else in the app changes
web/app/api/crypto/
  charge/route.ts              creates a pending `donations` row + a hosted charge/order
  webhook/[provider]/route.ts  verifies signature, idempotent via payment_events UNIQUE
                                (provider, event_id), calls mark_donation_succeeded/_failed
web/lib/supabase/service.ts    the ONE service-role client in the app; used only by the two
                                routes above, which is why they can call the service_role-only
                                RPCs from supabase/migrations/*_functions_triggers.sql
web/components/CryptoDonateForm.tsx   donor-facing form on /donate
```

Money flow: form posts amount + provider to `/api/crypto/charge` → a `pending` donation row is
written (service role, bypassing RLS same as the rest of the payment pipeline) → the provider's
hosted checkout URL is returned and the browser redirects there → the donor pays on Coinbase's or
Binance's own page (we never see card/wallet details) → the provider calls
`/api/crypto/webhook/<provider>` → signature is verified against the **raw** body →
`mark_donation_succeeded` (or `_failed`) runs, which is the same DB function the pgTAP tests in
Stage 1 already cover for idempotency and campaign-total correctness.

Configure webhook endpoints in each provider's dashboard as:
```
https://<your-domain>/api/crypto/webhook/coinbase_commerce
https://<your-domain>/api/crypto/webhook/binance_pay
```

## Known gaps in Stage 2

- `/donate` is an honest "not switched on yet" page. No payment form exists until Stage 3.
- Legal pages are outlines of what each document must cover, clearly labelled as not legal text.
- Admin area is a role gate plus campaign counts; management screens, charts and the media library UI are Stage 4/6.
- Images use a plain `<img>` component with fallback and credit line; the provider abstraction, thumbnails and AVIF/WebP delivery are Stage 6.
- No lockfile is committed (none could be generated offline). Commit `web/package-lock.json` after the first install.
- Crypto is one-time only; no recurring/monthly crypto donations, no partial-payment or underpayment handling beyond what each provider's hosted page does natively, and no live call to either provider has ever been made from this environment.
- No rate limiting on forms beyond a honeypot; add it at the edge (Vercel/WAF) or in an Edge Function.
- Google/phone sign-in and two-factor authentication are not wired up yet.
- Every page reads the session (header), so pages render per request rather than from a static cache. Fine for now; optimise later.

## Known gaps in Stage 1

- Not yet executed (see status note above).
- Country list is a starter set; import full ISO 3166-1 before launch.
- No rate limiting, CSRF, security headers, or file-content scanning yet (those live in the app/Edge layer).
- Video uploads intentionally not enabled.
- Legal, tax-receipt, and registration requirements are organisational work, not code.

## Roadmap

| Stage | Scope |
|---|---|
| 1 ✅ | Schema, RLS, storage, audit, payment-safe functions, tests, CI |
| 2 ✅ (unverified build) | Next.js app: auth flows, public pages, campaign search/filter/detail, homepage ledger from `public_impact_stats()`, donor dashboard, admin gate, SEO, error states |
| Crypto ✅ (signature logic tested; APIs never called live) | Coinbase Commerce + Binance Pay adapters, charge creation, signed/idempotent webhooks, donor UI |
| 3 | Stripe (one-time + monthly) via Edge Functions; signed, idempotent webhook that calls `mark_donation_*`; receipts; confirmation emails |
| 4 | Donor dashboard; admin dashboard, user management, donation management, audit viewer |
| 5 | PayPal; payment-provider abstraction; regional/mobile-money and crypto providers as adapters |
| 6 | Image service + Media Library; attribution; transparency/impact pages; SEO; safeguarding pages |

## Before production launch

Payment-provider verification and live keys · organisation/legal registration · privacy and cookie
compliance for your operating countries · tax/donation-receipt requirements · child-safeguarding policy
and reporting process · fraud prevention and card-testing protection · financial controls and refund
approvals · webhook signature and replay tests · database backups and a restore drill · MFA for every
admin account. A fuller checklist ships with Stage 3.
