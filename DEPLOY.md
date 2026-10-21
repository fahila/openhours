# Deploy Openhour (quick steps)

The app is the `app/` folder: Next.js 16, Postgres 15+. These steps assume Vercel and Neon, but any Node host and any Postgres work. You create every account and key yourself; none are in this repo.

Status: the code passes its tests against local mocks of Google, Resend and Stripe. It has never run against the real services. Do `manual-checklist.md` after the first deploy.

## 1. Database
Create a Postgres database (Neon, Supabase or your own). Copy its connection string (with `?sslmode=require`).

## 2. Keys and settings
All names and notes are in `app/.env.example`. You need:

| variable | where it comes from |
| --- | --- |
| `DATABASE_URL` | step 1 |
| `APP_URL` | your final https address, no trailing slash |
| `APP_ENCRYPTION_KEY` | `node -e "console.log(require('crypto').randomBytes(32).toString('base64'))"`. Keep a copy: losing it makes stored calendar tokens unreadable |
| `ALLOWED_EMAILS` | your sign-in email(s), comma separated. Empty means nobody can sign in |
| `CRON_SECRET` | any long random string |
| `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET` | your Google Cloud project, OAuth client (Web). Redirect URIs: `{APP_URL}/api/auth/google/callback` and `{APP_URL}/api/calendars/callback/google`. Enable the Google Calendar API |
| `RESEND_API_KEY`, `EMAIL_FROM` | Resend, with your sending domain verified |
| `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET` | optional. Leave empty to turn payments off. Use test keys first. Webhook `{APP_URL}/api/webhooks/stripe`, events `checkout.session.completed`, `checkout.session.expired`, `charge.refunded` |

Do not set `DEV_LOGIN` in production: the app refuses to start with it.

Google note: while the OAuth consent screen is in "Testing", Google may expire refresh tokens after about 7 days, so check the current Google docs and consider moving it to "In production" for your own account.

## 3. Host (Vercel)
1. Push this repo to your own GitHub repo (this session has not pushed anything).
2. Import it in Vercel. Set **Root Directory** to `app`.
3. Add the variables from step 2.
4. Deploy. The production start refuses to boot if a required variable is missing, and the log says which.

## 4. Migrate
Once, from your machine, with the production `DATABASE_URL`:
```
cd app && npm ci && DATABASE_URL="..." npm run db:migrate
```
Run it again after any release that adds a file to `app/db/migrations`.

## 5. Reminders and emails need a timer
Emails and reminders go through a job queue. Something must call `GET` or `POST {APP_URL}/api/jobs/run` about once a minute with the header `Authorization: Bearer <CRON_SECRET>`. Options: a free external scheduler (cron-job.org or similar), a GitHub Actions schedule (5 minute minimum), Vercel Cron if your plan allows per-minute runs (check current limits), or `npm run worker` on a small always-on server. Without it, confirmation emails and reminders do not send.

## 6. Domain
Add the domain in the host, then the DNS records it shows (A for `@`, CNAME for `www`). Add the SPF and DKIM records Resend gives you, and a DMARC record such as `v=DMARC1; p=none; rua=mailto:you@yourdomain`. Update `APP_URL` and the Google redirect URIs to the final address.

## 7. Check
Sign in, connect your calendar, make an event type, book it from a second email, reschedule, cancel. Then go through `manual-checklist.md`.
