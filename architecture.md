# Architecture: a self-hosted scheduling app (a rebuild of Calendly's core features)

> Built differently from this plan in a few places (plain SQL instead of Drizzle, own sessions instead of Auth.js, a Postgres job table instead of Inngest, live busy-time reads instead of a cache). The table of changes and the reasons are in `backend.md`. The SQL moved to `app/db/migrations`.

Inputs: `replica/recon.md`, `replica/features.csv`. Own business use, web only.

Assumptions (the recon's open questions were not answered, so these are defaults, change them freely):
- One host first, but every owned row carries `user_id`, so teams can be added later without a rewrite.
- Google Calendar + Google Meet first. Outlook and Zoom second.
- Stripe is built but in the should-have milestone. SMS is deferred (needs a provider and consent handling).
- Teams, round-robin, collective, routing forms, groups and the admin center are not in the schema. They are all could/skip rows. See "Later" at the bottom.

## Stack

| layer | choice | why |
| --- | --- | --- |
| web | Next.js (App Router) + TypeScript | Guest booking pages need server rendering and fast first paint. One codebase for host app, public pages and API. |
| styling | Tailwind, tokens from replica-design | Matches the next skill's output. |
| database | Postgres on Neon | The double-booking guarantee is an exclusion constraint, which needs real Postgres. Branching helps testing. |
| ORM | Drizzle | SQL-shaped, keeps `schema.sql` and types aligned. The exclusion constraint goes in as raw SQL in a migration. |
| auth | Auth.js with Google provider | Own use: sign in with Google. Calendar scopes are a separate consent step (see risks). |
| payments | Stripe Checkout | Hosted payment page, webhooks confirm the booking. |
| email | Resend | Confirmation, reminder, follow-up. Send ICS attachments. |
| jobs | Inngest | Durable delayed steps ("send 24h before") without running a worker. Retries and idempotency keys built in. |
| calendar | Google Calendar API, Microsoft Graph | Official APIs only, the user's own OAuth client. |
| files | Cloudflare R2 (logo upload only) | Only needed for branding. Skip until then. |
| hosting | Vercel | Cron/webhooks, previews. Region next to the Neon region. |

No microservices, one database, no queue to run yourself.

## Schema

Tables: 24. Access rules: data-layer checks. Public guest routes are anonymous, so Postgres RLS does not help there. Every owner-scoped query goes through a helper that requires the signed-in `user_id`; guest actions use hashed random tokens (`manage_token_hash`, `token_hash`) and only ever touch the one booking, link or poll the token names.

Full SQL: `replica/schema.sql`. It loads cleanly on Postgres 16 and the key constraints were tested:
- an overlapping booking for the same host is rejected (`no_host_overlap`)
- an adjacent booking (10:30 right after 10:00 to 10:30) is allowed
- a cancelled booking frees its slot
- a one-on-one event with `max_invitees` greater than 1 is rejected

| area | tables |
| --- | --- |
| identity | users |
| calendars | calendar_connections, calendars, busy_blocks |
| availability | availability_schedules, availability_rules, date_overrides |
| event types | event_types, invitee_questions, free_busy_exceptions |
| bookings | bookings, booking_invitees, single_use_links |
| comms | workflows, workflow_event_types, message_log |
| money | payments |
| polls | meeting_polls, poll_options, poll_votes |
| API | webhook_subscriptions, webhook_deliveries, api_tokens, inbound_events |

Design decisions worth knowing:
- **A booking is a meeting; invitees hang off it.** A group event is one `bookings` row with many `booking_invitees`. That is what lets one host-level exclusion constraint stop double booking and still allow groups. Capacity (`max_invitees`) is checked inside the booking transaction under a row lock on the booking.
- **Weekly hours are wall-clock times in the schedule's time zone.** They are expanded into UTC instants at slot-generation time for the specific date, which is what makes DST right. Never store pre-expanded UTC weekly slots.
- **Buffers and limits are not in the constraint.** The constraint stops exact overlaps. Buffers, minimum notice, per-day/week/month limits and busy times are enforced by slot generation and re-checked inside the booking transaction.
- **Cancel/reschedule links use a random token, stored as a sha256 hash.** The token appears only in the email link. `rescheduled_from` links the new booking to the old one.
- **Idempotency is a unique key, not code.** `message_log` is unique per (booking, invitee, workflow, kind), `webhook_deliveries` per (subscription, event, booking), `inbound_events` per (provider, event_id).
- **Secrets** (`refresh_token_enc`, `signing_secret_enc`) are encrypted with an app key held in the host's env vars, never plaintext.

## API

Auth column: `host` = signed-in session, `token` = guest manage link, `pat` = personal access token (API), `none` = public, `sig` = provider signature.

| method path | does | who | input | output | flow |
| --- | --- | --- | --- | --- | --- |
| GET /{user} | host profile page, lists non-hidden active event types | none | slug | page | F03 |
| GET /{user}/{event} | booking page | none | slugs | page | F03 |
| GET /api/public/{user}/{event}/slots | open slots for a date range | none | from, to, invitee tz | list of start instants | F03 |
| POST /api/public/{user}/{event}/book | create booking (transaction, constraint, capacity, re-check slot) | none, rate-limited + captcha | slot, name, email, tz, answers, guests | booking + manage link, or Stripe session | F03 F11 |
| GET /m/{token} | manage page (view, cancel, reschedule) | token | token | page | F04 |
| POST /m/{token}/cancel | cancel booking, delete calendar event, email both | token | reason | ok | F04 |
| POST /m/{token}/reschedule | new booking linked by `rescheduled_from`, cancel old | token | slot | new booking | F04 |
| GET /s/{token} | single-use / one-off booking page | token | token | page (marks viewed) | F06 |
| GET /p/{token} | meeting poll voting page | token | token | page | F07 |
| POST /p/{token}/vote | record votes | token | options, name, email | ok | F07 |
| GET /embed.js | inline / popup / floating widget loader | none | none | script | F03 |
| GET /api/event-types | list own event types | host | none | list | F02 |
| POST /api/event-types | create event type | host | event type fields | row | F02 |
| PATCH /api/event-types/{id} | update (incl. limits, buffers, confirmation) | host | fields | row | F02 |
| DELETE /api/event-types/{id} | delete or deactivate (restrict if it has bookings) | host | none | ok | F02 |
| PUT /api/event-types/{id}/questions | replace invitee questions | host | ordered list | list | F02 |
| PUT /api/event-types/{id}/free-busy-exceptions | replace exceptions | host | titles | list | F05 |
| GET /api/availability | list schedules with rules and overrides | host | none | list | F05 |
| POST /api/availability | create schedule | host | name, tz, rules | row | F01 F05 |
| PUT /api/availability/{id} | replace weekly rules and date overrides | host | rules, overrides | row | F05 |
| DELETE /api/availability/{id} | delete (restrict if used by an event type) | host | none | ok | F05 |
| GET /api/calendars/connect/{provider} | start OAuth for calendar scopes | host | provider | redirect | F01 |
| GET /api/calendars/callback/{provider} | finish OAuth, store encrypted refresh token, list calendars | host | code, state | redirect | F01 |
| PATCH /api/calendars/{id} | set check-conflicts and booking-target flags | host | flags | row | F05 |
| DELETE /api/calendar-connections/{id} | disconnect, drop busy blocks | host | none | ok | F05 |
| GET /api/bookings | upcoming/past/filters | host | filters, cursor | list | F03 |
| POST /api/bookings/{id}/cancel | host cancels | host | reason | ok | F04 |
| POST /api/bookings/{id}/no-show | mark invitee no-show | host | invitee id | ok | S10 |
| GET /api/single-use-links | list | host | none | list | F06 |
| POST /api/single-use-links | create link (optionally with one-off offered slots) | host | event type, slots? | link | F06 |
| DELETE /api/single-use-links/{id} | revoke, release reserved slots | host | none | ok | F06 |
| GET /api/polls | list | host | none | list | F07 |
| POST /api/polls | create poll | host | title, duration, options | poll + link | F07 |
| POST /api/polls/{id}/close | pick a time and create the booking | host | option id | booking | F07 |
| GET /api/workflows | list | host | none | list | F08 |
| POST /api/workflows | create | host | trigger, offset, body, event types | row | F08 |
| PATCH /api/workflows/{id} | update | host | fields | row | F08 |
| DELETE /api/workflows/{id} | delete | host | none | ok | F08 |
| GET/PATCH /api/me | profile, slug, time zone, branding | host | fields | row | F01 |
| GET/POST/DELETE /api/webhook-subscriptions | manage outbound webhooks | host | url, events | row | F12 |
| GET/POST/DELETE /api/tokens | manage personal access tokens | host | name | token shown once | F12 |
| GET /api/v1/event-types | list event types | pat | none | list | F12 |
| GET /api/v1/scheduled-events | list bookings | pat | filters | list | F12 |
| GET /api/v1/scheduled-events/{id}/invitees | list invitees | pat | none | list | F12 |
| POST /api/v1/scheduled-events/{id}/cancel | cancel | pat | reason | ok | F12 |

45 route rows above (a few combine verbs, e.g. GET/PATCH /api/me), plus the 3 provider webhooks below: 48 in all.

Webhooks in:
- `POST /api/webhooks/stripe` (sig): `checkout.session.completed` confirms a `pending_payment` booking, `charge.refunded` marks refunded. Deduplicated through `inbound_events`.
- `POST /api/webhooks/google-calendar` (sig/channel token): push notification means "re-sync this calendar".
- `POST /api/webhooks/microsoft` (sig/clientState): same for Outlook via Graph subscriptions.

Webhooks out: `invitee.created`, `invitee.cancelled`, `invitee.rescheduled` to `webhook_subscriptions`, signed with HMAC-SHA256, retried with backoff, recorded in `webhook_deliveries`.

Jobs (Inngest):

| job | schedule | what it does |
| --- | --- | --- |
| `booking.created` | event | write calendar event, add meet link, send confirmations, schedule reminders and follow-ups, enqueue webhooks |
| `workflow.send` | delayed step, per booking | sleep until `start_at - offset` (or after end), re-check booking still confirmed, send, write `message_log` |
| `calendar.sync` | on push + every 15 min fallback | pull busy blocks incrementally with `sync_token`, full resync on 410 |
| `calendar.watch.renew` | daily | renew Google channels and Graph subscriptions before they expire |
| `payment.expire` | 15 min after a `pending_payment` booking | cancel unpaid bookings, release the slot |
| `webhook.deliver` | event, retry | POST to subscriber, exponential backoff, mark failed after N tries |
| `cleanup` | daily | expire single-use links and polls, purge old busy blocks |

## The parts that bite

- **Time zones and DST.** Store UTC instants, store the schedule's and the invitee's IANA zone name, never offsets. Expand weekly rules per date in the schedule's zone with a tz library (Luxon or Temporal). Spring-forward days have a missing hour, fall-back days a repeated one. Add tests on the two DST dates of at least two zones (e.g. America/New_York, Europe/London, and one southern-hemisphere zone), plus a half-hour offset zone (Asia/Kolkata).
- **Double booking.** The slot list is only advice. The truth is the transaction on `POST .../book`: re-validate the slot against rules, notice, limits and busy blocks, then insert. The exclusion constraint turns a race into a clean 409, and the UI answers "that time was just taken, pick another".
- **Busy time vs bookings.** Bookings made through the app also exist on the host's Google Calendar and come back through sync. Match on `provider_event_id` so they do not count twice.
- **Calendar sync correctness.** Use incremental sync tokens, handle token expiry (full resync), recurring events (expand instances, singleEvents=true), all-day events, declined events, and free/transparent events (not busy). Outlook working hours are not imported, so the schedule is the only source of hours.
- **OAuth scope upgrades.** Sign-in with Google needs only profile and email. Calendar read/write is a second consent with sensitive scopes. For own use keep the OAuth app in testing mode with the owner as test user; refresh tokens in testing mode expire after 7 days, so publish the app (still unverified is fine for one user) or use a Workspace-internal app. Handle `needs_reauth` visibly in the UI.
- **Idempotency.** Stripe and Google webhooks can arrive twice or out of order. `inbound_events` dedupes by event id. Reminder jobs retry safely through the unique key on `message_log`.
- **Reminders for moved or cancelled meetings.** Each delayed send re-reads the booking and exits if it is cancelled or moved. Do not rely on cancelling scheduled jobs.
- **Abuse.** The booking endpoint is public. Rate limit per IP and email, add a captcha challenge after suspicious bursts (Calendly added the same in March 2026), validate email, cap `answers` size.
- **Email deliverability.** SPF, DKIM and DMARC on the sending domain before launch. Send from your own domain, include ICS with the right method (REQUEST, CANCEL), and a stable UID so updates replace the invite.
- **Rate limits.** Google Calendar API and Graph quotas: batch busy-block reads (freebusy endpoint) for slot generation, cache per host for a minute.
- **Privacy / GDPR.** Guests give name and email. Provide a delete-by-email path (anonymise `booking_invitees`) and say what you store on the booking page.
- **Embeds.** The widget loads the booking page in an iframe; allow framing for `/{user}/*` only and keep CSP `frame-ancestors` permissive for those paths but strict everywhere else.
- **Multi-tenancy.** Not needed now. Keep every query scoped by `user_id` so adding an org layer later is additive.

## Build order

Screen IDs refer to `replica/recon.md`. Row names refer to `replica/features.csv`.

1. **Vertical slice (ugly, proves the stack).**
   Screens: S01, S02 (Google only), S11 (weekly hours only), S03 + S04 (basic one-on-one), S31-S34, S10 (list only).
   Tables: users, availability_schedules, availability_rules, event_types, bookings, booking_invitees, calendar_connections, calendars, busy_blocks.
   Routes: auth, `/api/calendars/connect|callback`, `POST /api/availability`, `POST /api/event-types`, `/api/public/.../slots`, `/api/public/.../book`, `GET /api/bookings`.
   Done when: you sign in, connect Google, create a 30 minute event, a guest books it from a private window in another time zone, the event shows on your Google Calendar, a second guest cannot take the same slot, and the invitee gets a confirmation email.

2. **Must-haves.**
   Add: date overrides, min notice, location options with Meet links, ICS confirmation emails to both sides, guest reschedule and cancel (S35, S36, `/m/{token}`), time zone switcher, confirmation page (S06), full scheduled events list, `invitee.*` calendar delete/update on cancel.
   Tables: date_overrides.
   Milestone check: all 17 `must` rows in features.csv flip to `yes`.

3. **Should-haves.**
   Buffers, limits, date range, start increments, custom invite questions (S05), custom slugs, group events, workflows with email reminders and follow-ups (S13), single-use and one-off links (S07, S08, S37), embeds (S40), Stripe payments (F11), webhooks and the API (F12), Outlook calendar, captcha, multiple conflict calendars.
   Tables: invitee_questions, workflows, workflow_event_types, message_log, single_use_links, payments, webhook_subscriptions, webhook_deliveries, api_tokens, inbound_events, free_busy_exceptions.

4. **Could-haves (backlog, only if wanted).**
   Meeting polls (S09, S38; tables meeting_polls, poll_options, poll_votes), branding, no-show, hidden events, redirect after booking, extension/add-in, contacts, analytics.

5. **The fixes** replica-entrepreneur finds, once it has run.

## Later (not in the schema yet)

If the app ever becomes a team tool, add as migrations: `organizations`, `memberships` (role), `groups`, `event_type_hosts` (for collective and round robin, plus a `kind` check change), `routing_forms` + `routing_routes`, `activity_log`. None of them force changes to existing tables beyond adding an `org_id` column.
