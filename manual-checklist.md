# Manual checks

What a computer in a sandbox cannot do. Everything automatable is in `app/e2e/specs` (browser) and `app/tests` (database and services against local mocks). These are the checks that need real accounts, real mail or a real person. Do them once, in order, before telling anyone the app works. Tick each and note the date.

Set up first: a real `.env.local` (see `app/.env.example`), `npm run db:migrate`, `npm run dev` or a deploy, and the scheduler calling `/api/jobs/run` every minute (or `npm run worker`).

## Google sign-in and calendar (needs your Google Cloud project)

- [ ] Sign in with an address in `ALLOWED_EMAILS`. You land on Event types.
- [ ] Sign in with a different Google account. You are refused, and no account is created.
- [ ] On the consent screen, untick one calendar permission. The connection is refused with a clear message and nothing half-works.
- [ ] Connect a calendar with all permissions. Your calendars are listed; the primary one blocks time and receives bookings.
- [ ] Create a busy event on that calendar. Reload your booking page: the times around it disappear (allow up to a minute for the short cache).
- [ ] Book a time as a guest. An event appears on your calendar with a Meet link and the guest's name. The guest is **not** sent a second invitation by Google.
- [ ] Cancel the booking from the guest link. The event disappears from your calendar.
- [ ] Move the booking. The old event goes, one new event appears.
- [ ] In your Google account settings, remove the app's access. Within a minute the Integrations page says "Needs reconnecting", and the booking page stops offering times rather than guessing. Reconnect fixes it.
- [ ] Workspace accounts: confirm Meet links are created. If an admin has turned Meet off, note what the guest's email shows.

## Email (needs your Resend domain)

- [ ] Book as a guest using a real address of yours (try Gmail and one other provider). Confirmation arrives in the inbox, not spam, from your sender address.
- [ ] Open the attached invite in Gmail, Apple Calendar and Outlook. It shows the right time in your zone and the place.
- [ ] Move the meeting. The existing event in each calendar app **updates** (does not duplicate).
- [ ] Cancel it. The event is removed or marked cancelled in each app.
- [ ] Wait for (or pull forward) the 24-hour reminder. It arrives with a working "Change or cancel" link.
- [ ] Check the headers of one email in Gmail ("Show original"): SPF, DKIM and DMARC all say PASS.
- [ ] Reply to a confirmation email. The reply goes to the host's address.

## Payments (Stripe test mode, needs your keys and a webhook endpoint or `stripe listen`)

- [ ] Turn on payment for an event type. The booking page shows the price. Booking sends you to Stripe.
- [ ] Pay with `4242 4242 4242 4242`. You return to a confirmed booking; the confirmation email arrives only after that; the host's Meetings list shows "Paid".
- [ ] Book again and pay with `4000 0000 0000 0002` (declined). Stripe shows the decline; use Stripe's back link. The time is free again and the booking page says you were not charged.
- [ ] Book again and just close the Stripe tab. After 35 minutes the time is free again and the Stripe session is expired.
- [ ] Pay with `4000 0027 6000 3184` (asks for 3-D Secure). Completing the check confirms the booking.
- [ ] Refund a payment from the Stripe dashboard. The payment shows "Refunded" in the app after the webhook arrives.
- [ ] Look at the webhook's delivery log in Stripe: all events show 200.

## By eye, on real devices

- [ ] iPhone Safari and Android Chrome: the full booking flow, including the time zone menu and the date pickers on the Availability page.
- [ ] Firefox and Safari on a desktop: booking and the host screens (the automated suite runs Chromium only).
- [ ] A screen reader (VoiceOver or NVDA) through the booking flow: the calendar announces dates and "no times available", errors are read out, and the confirmation page makes sense.
- [ ] Zoom the browser to 200%: nothing is cut off.
- [ ] Dark mode in the operating system: the pages are still readable (the app has a light theme only; note anything unreadable).

## Operations

- [ ] Restore a backup of the database into a scratch database and open the app against it.
- [ ] Stop the scheduler for an hour, then start it: reminders and retries catch up without sending duplicates.
- [ ] Make the email service fail on purpose (wrong API key for a minute): the booking still succeeds, the confirmation arrives once the key is fixed, and a dead job shows up in the log as `"msg":"job dead"` if it stays broken.
- [ ] Check the host's access logs: manage links contain secret tokens in the path. Confirm those logs are not shared.
