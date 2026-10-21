# Backend

Status as of 2026-10-21. Everything here was built and tested in a sandbox that has **no Google, Stripe or Resend account**. Where a real service would be involved, the code was run against a local mock that follows that service's documented API, so the first run against the real thing is the part that has not been proven. That list is in "Not yet proven against the real service" below. Read it before going live.

## What exists

| area | what it does | where |
| --- | --- | --- |
| database | Postgres 15+, SQL migrations run by a script, exclusion constraint that makes double booking impossible | `app/db/migrations`, `app/scripts/migrate.ts` |
| data layer | the same function names the screens already used, now on Postgres; every owned query takes the signed-in user's id | `app/lib/data` |
| sign-in | Google sign-in only, allow-list of emails, sessions in the database, http-only cookie, sign out here or everywhere, account deletion | `app/lib/auth`, `app/app/api/auth`, `app/app/login`, `app/app/(host)/settings` |
| calendars | Google Calendar: connect, choose which calendars block time and which receives bookings, live busy-time check, events with Meet links, delete on cancel | `app/lib/calendar`, `app/app/api/calendars`, `app/app/(host)/integrations` |
| email | Resend: confirmation, cancellation, move and reminder emails with calendar invites attached | `app/lib/email`, `app/lib/ics.ts` |
| jobs | Postgres job queue with retry, backoff, dead-letter state; runs from a scheduler hitting `/api/jobs/run` or from `npm run worker` | `app/lib/jobs`, `app/scripts/worker.ts` |
| payments | Stripe Checkout, test mode, signed idempotent webhooks, hold-and-release of the slot, late-payment refund | `app/lib/payments`, `app/app/api/webhooks/stripe` |
| seed | fake demo host with bookings; refuses to run in production | `app/scripts/seed.ts` |

## Decisions that differ from `architecture.md`

| architecture said | built | why |
| --- | --- | --- |
| Drizzle ORM | plain SQL migrations and `pg` | `schema.sql` already existed and was tested. Raw SQL keeps one source of truth and shows the exclusion constraint as written. |
| Auth.js | own small session layer | The only provider is Google with an allow-list. A session table and an encrypted OAuth cookie are less code than the library plus adapter, and every path is covered by tests. |
| Inngest | Postgres job table | No account needed, testable here, and jobs are written in the same transaction as the booking so they cannot be lost. Any scheduler that can send an HTTP header works. |
| busy times cached in `busy_blocks`, synced by push | read live from Google's freeBusy, cached 60 seconds | Removes push-channel renewal and sync-token handling. A booking always re-checks Google fresh. `busy_blocks` was dropped from the schema. |
| `schema.sql` in `replica/` | `app/db/migrations/0001_init.sql` | Migrations are the source of truth now. |

## Setup you have to do (Claude cannot and must not)

1. **Postgres.** Create a database (Neon, Supabase or your own, version 15 or newer). Put the connection string in `DATABASE_URL`. Run `npm run db:migrate`.
2. **Google Cloud project.** APIs & Services: enable the Google Calendar API. Create an OAuth client (Web application). Add both redirect URIs from `.env.example`. Put the id and secret in `.env.local`. Details in "Google verification" below.
3. **Resend.** Verify your sending domain (SPF and DKIM records), create an API key, set `RESEND_API_KEY` and `EMAIL_FROM`.
4. **Stripe (optional).** Test-mode secret key, then add a webhook endpoint `APP_URL/api/webhooks/stripe` for `checkout.session.completed`, `checkout.session.expired`, `charge.refunded` and copy its signing secret.
5. **Secrets.** `APP_ENCRYPTION_KEY` (32 random bytes, base64), `CRON_SECRET`, `ALLOWED_EMAILS`.
6. **A scheduler.** Something must call `POST APP_URL/api/jobs/run` with `Authorization: Bearer $CRON_SECRET` every minute (reminders and retries wait for it), or run `npm run worker` on a server.

`.env.example` lists every variable with no values. The app refuses to start in production if a required one is missing.

## Google verification (start this early)

Sign-in uses only `openid email profile`, which needs no review. The calendar connection asks for three scopes:

- `calendar.calendarlist.readonly` to list calendars
- `calendar.freebusy` to read busy times
- `calendar.events` to create and delete the meetings it books

`calendar.events` is a **sensitive** scope. While the OAuth consent screen is in "Testing", Google expires refresh tokens after **7 days** and limits you to 100 listed test users, so calendar access would silently stop weekly. For a single-owner tool there are two ways out:

- **Publish the app to "In production" without verification.** Works for you as the only user; Google shows an "unverified app" warning and caps unverified apps at 100 users. This is the practical route for own use.
- **Verification** (needed only if other people will connect their calendars): a public privacy policy, a homepage on a domain you own, a demo video of the consent flow, and a justification per scope. Review commonly takes weeks. Start it before launch if that is the plan.

Granular consent is handled: if someone unticks a calendar permission on the consent screen, the connection is refused with a clear message instead of half working.

## Security checklist

- [x] **Secrets only in env vars, `.env*` ignored, nothing in client bundles.** `.gitignore` ignores every `.env*` except `.env.example` (checked with `git check-ignore`; the first version of the ignore file missed `.env.local` and was fixed). Server-only modules read `process.env`; no secret has a `NEXT_PUBLIC_` name.
- [x] **Environment validated at boot.** `instrumentation.ts` runs `env()` when the server starts. Production refuses to start, naming each missing variable, if any is missing, if `APP_URL` is not https, if `DEV_LOGIN` is set, or if a mock-server override is set (checked by starting the production build three ways).
- [x] **Input validated on the server.** zod schemas bound every request body, query and server-action payload (`lib/schemas.ts`); form-based screens use `lib/validate.ts`. Ids from URLs and forms are checked as UUIDs first, so a bad id is a 404, not a database error.
- [x] **Authorisation on every read and write, tested with a second user.** Every owned query takes the signed-in user's id. `tests/data.test.ts` has a second user trying to read, change, delete and cancel the first user's rows and getting nothing. Guest access is by hashed random token and reaches one booking only.
- [x] **Sign-in.** Google only, verified email required, allow-list checked at sign-in *and on every request* (removing an address locks that person out immediately). PKCE and an encrypted, expiring `state` cookie. Sessions are random tokens stored as hashes; cookie is httpOnly, SameSite=Lax, and `__Host-` prefixed plus Secure in production. Sign out and sign out everywhere both delete rows (replaying an old cookie was tested).
- [x] **Rate limits** on sign-in (per IP), on booking (per IP, per guest email and per host, in Postgres so they hold across instances), and on guest cancel and reschedule. Booking is the only path that sends email to an address supplied by a stranger, which is why it has the per-email cap. Behind a proxy that overwrites `X-Forwarded-For` only.
- [x] **Webhooks verify signatures.** Stripe: official verifier on the raw body, 5-minute replay window, event id recorded in the same transaction as its effects.
- [x] **Idempotent everywhere it matters.** Webhook events, emails (`message_log` unique key plus the provider's idempotency header), calendar events (id derived from the booking), jobs (dedupe keys), refunds.
- [x] **Secrets at rest.** Calendar refresh tokens and guests' manage tokens are AES-256-GCM encrypted with `APP_ENCRYPTION_KEY`; lookup uses a SHA-256 hash. A test confirms no plaintext token is stored.
- [x] **Uploads.** None exist, so there is nothing to limit.
- [x] **No user data in URLs or logs.** Guests' manage links are capability URLs by design. Those pages send `Referrer-Policy: no-referrer` and `Cache-Control: no-store`. The dev email transport logs the subject and recipient but never the body. Production logs contain booking ids, not names or addresses. **Your host's access logs will contain the manage tokens in request paths;** keep them private or short-lived.
- [x] **Headers.** nosniff, `X-Frame-Options: DENY`, a CSP subset (`frame-ancestors 'none'; form-action 'self'; base-uri 'self'; object-src 'none'`), Permissions-Policy, HSTS in production, no `X-Powered-By`. A full script-src CSP is not set (it needs per-request nonces). Booking pages are not frameable, so the embed widget (not built) will need that relaxed for those paths.
- [x] **Dependencies audited.** `npm audit` reports 0 vulnerabilities, including dev dependencies.
- [x] **Privacy page** at `/privacy` lists the processors that actually touch guests' details (the host, the hosting and database providers, the email service, Google Calendar, and Stripe when payments are on). **It is a starting point. Read it against your real setup, and have it reviewed before relying on it.**
- [x] **Account deletion actually deletes.** Revokes Google tokens, then removes the user, event types, hours, bookings, guests, payments, workflows and sessions. The browser run checks every table is empty afterwards. (It caught a real bug: payment rows blocked the delete.)
- [ ] **Backups.** Cannot be checked from here. Turn on and test point-in-time recovery or daily backups at your Postgres host (Neon and Supabase both offer it), and write down how to restore.
- [ ] **Log retention and alerting.** Dead jobs are logged as one JSON line each (`"msg":"job dead"`); set an alert on that string, and on `GET /api/jobs/run` returning non-200 from your scheduler.

## Verification

All of it runs locally with a Postgres 15+ database.

| suite | what it covers | count |
| --- | --- | --- |
| `tests/slots.test.ts` | slot rules, DST, buffers, limits, groups, busy times | 15 |
| `tests/data.test.ts` | real Postgres: ten-way booking race, database overlap guard, second-user isolation, jobs written with the change, account deletion | 17 |
| `tests/email-jobs.test.ts` | confirmation, cancel, move and reminder emails, retries with backoff, dead letters, escaping, the Resend request, calendar identity across moves | 11 |
| `tests/google.test.ts` | OAuth and PKCE, connection storage, busy times, Meet events, outages, revoked access (including a 401 on a cached token), disconnect (against a mock Google) | 21 |
| `tests/stripe.test.ts` | Checkout request, signed webhooks, replay and tamper rejection, duplicates, expiry, late-payment refund (against a mock Stripe) | 15 |
| `tests/env.test.ts` | production environment rules, cookie settings | 6 |
| `tests/security.test.ts` | rate limits, allow-list, malformed ids and link names, SQL and NUL bytes in guest input, body bounds, dates, link-name generation | 13 |
| `app/e2e/specs` (Playwright, 15 files) | the whole app in a real browser against a real Next server: sign-in and sessions, calendar, booking, moving and cancelling, availability and DST, emails and invites, payments, security, accessibility, phone width, keyboard. Plan in `test-plan.md`, bugs in `bugs.md` | 182 |

```bash
export TEST_DATABASE_URL=postgres://user@host:5432/scheduler_test   # a scratch database, it is wiped
cd app && npm test                                                   # 98 tests
TEST_PG=postgres://user@host:5432 npm run e2e                        # browser suite, needs Chromium; starts its own database, mock and server
```

Bugs the tests found in the first drafts of the backend, all fixed (the browser pass found 16 more, see `bugs.md`): a double-booking race (an `await` between the slot check and the insert), account deletion failing when a paid booking existed, a failed booking wiping the guest's typed details, reminder jobs left queued for cancelled meetings, `.env.local` not git-ignored, and production starting "healthy" with a missing variable.

## Not yet proven against the real service

Run each once with real keys, in this order, before telling anyone the backend works:

1. **Google sign-in and calendar connect** with your own client. The OAuth code, PKCE, token refresh, `freeBusy`, `events.insert` with a client-chosen id and `conferenceData`, and `events.delete` all follow Google's documentation and were run against a mock of it. Things a mock cannot tell us: exact consent-screen behaviour, whether Google rejects the Meet `createRequest` for some account types (Workspace admins can disable Meet), and real freeBusy error shapes.
2. **Resend** with your verified domain: the request shape was checked against a mock, and the idempotency header is sent. Deliverability (SPF, DKIM, DMARC, spam folder) can only be checked with real mail.
3. **Stripe test mode**: Checkout creation and the webhook use the official `stripe` library; signatures are verified with the library's own verifier and the tests sign events with its own signing helper. Check once with a real test card and the Stripe CLI (`stripe listen --forward-to`).

## Open decisions

- **Refunds.** Cancelling a paid booking does not refund automatically. The host refunds in the Stripe dashboard and `charge.refunded` marks the payment. A late payment (money arriving after the slot was released) is refunded automatically. Say if you want a refund-on-cancel rule.
- **Hold time.** An unpaid slot is held 35 minutes (Stripe's minimum Checkout lifetime is 30).
- **SMS reminders** are not built. The workflow rows exist but only the email channel runs.
- **Teams.** One host per instance. Adding teams means adding the organisation tables listed at the end of `architecture.md`.
