# Build log

The data layer is Postgres now (see `backend.md`). The table below is the state after the backend pass.

Run it: set up `.env.local` from `.env.example`, `npm run db:migrate`, optionally `npm run db:seed` (needs `DEV_LOGIN=1` to sign in as demo@example.com), then `npm run dev`. Host app at `/event-types`, guest page at `/{your-link-name}`.

| ID | screen | date | status | missing / notes |
| --- | --- | --- | --- | --- |
| shell | host layout, nav, stubs for unbuilt routes | 2026-10-21 | done | Sidebar on desktop, scrolling top nav on mobile. |
| S01 | Sign in | 2026-10-21 | done | Google sign-in, allow-list, sessions. Dev login behind `DEV_LOGIN` outside production. |
| S02 | Onboarding: connect calendar | 2026-10-21 | partial | No separate first-run screen. A time zone prompt appears in the app, and calendars connect from Integrations. |
| S03 | Event types list | 2026-10-21 | done | Copy link, edit, on/off switch, empty state, long names wrap. No hidden/secret toggle. |
| S04 | Event type editor | 2026-10-21 | partial | All fields in the architecture except custom questions (S05) and redirect URL. Validation inline, create, edit, delete. |
| S05 | Invitee form editor | 2026-10-21 | not built | Guests get one fixed optional notes field. |
| S06 | Confirmation settings | 2026-10-21 | partial | A confirmation message field lives in S04. No redirect. |
| S07 | One-time links | 2026-10-21 | stub | Should-have milestone. |
| S08 | One-off meeting creator | 2026-10-21 | not built | Should-have milestone. |
| S09 | Meeting polls | 2026-10-21 | stub | Could-have. |
| S10 | Meetings | 2026-10-21 | partial | Upcoming and past tabs, cancel with reason, empty states. No filters, no no-show marking, no detail drawer. |
| S11 | Availability | 2026-10-21 | done | Weekly ranges, multiple per day, date overrides, time zone. One schedule only. |
| S12 | Calendar connections | 2026-10-21 | done | On the Integrations page: connect, choose conflict calendars and the booking calendar, refresh, disconnect, reconnect banner. |
| S13 | Workflows | 2026-10-21 | stub | The engine and a default 24 hour reminder exist; no screen to edit them. |
| S14 | Integrations | 2026-10-21 | partial | Calendars and a Stripe status line. Webhook and API token screens are not built. |
| S15, S16, S18-S22 | routing, contacts, admin, billing | 2026-10-21 | stub or not built | Later milestones or out of scope. |
| S17 | Settings | 2026-10-21 | done | Profile, link name, time zone, sign out, sign out everywhere, delete account. |
| S30 | Host profile page | 2026-10-21 | done | Lists active, non-hidden event types. Empty state. |
| S31 | Booking page: pick day | 2026-10-21 | done | Loading skeleton, error with retry, empty month message, jumps ahead to the first month with times. |
| S32 | Booking page: pick time | 2026-10-21 | done | 12/24h toggle, guest time zone detection. Slots refresh after a conflict. |
| S33 | Details form | 2026-10-21 | partial | Name, email, notes, link to the privacy page; paid events continue to Stripe. Field errors, network error keeps the form. No add-guests. |
| S34 | Confirmation | 2026-10-21 | done | Summary in the guest's zone, ICS download, links to change or cancel. |
| S35 | Reschedule | 2026-10-21 | done | At `/reschedule/{token}`, reuses the booking flow. The booking's own time counts as free. |
| S36 | Cancel | 2026-10-21 | done | A dialog on the manage page `/m/{token}`, not its own page. |
| S37 | Single-use page | 2026-10-21 | not built | Should-have milestone. |
| S38 | Poll voting | 2026-10-21 | not built | Could-have. |
| S39 | Routing form (guest) | 2026-10-21 | not built | Could-have. |
| S40 | Embed wrappers | 2026-10-21 | not built | Should-have milestone. |

## Harder than it looked

- **Double booking was easy to get wrong.** The first draft of `createBooking` awaited the slot check before inserting, so three simultaneous requests could all pass the check. A test that fires three at once caught it. The check and insert are now synchronous with no `await` between, and a comment says why. In Postgres the exclusion constraint does this job.
- **React 19 wipes uncontrolled fields after a form action.** A failed booking cleared what the guest typed. The booking form now uses `onSubmit`. The editor keeps its values by echoing them back from the action.
- **Time zones.** Weekly hours are wall-clock times expanded per date, so DST is handled in one place. Tests cover the spring-forward gap, the fall-back repeated hour and a half-hour offset zone. Chromium reports Kolkata as `Asia/Calcutta`, so the zone picker shows that older name; it is a valid zone, cosmetic only.
- **Guest day vs host day.** A guest's calendar day can be a different date from the host's. The slots API takes host dates, so the page fetches one extra day either side and groups by the guest's zone.

## Decisions to revisit

- The slug `reserved` list in `lib/validate.ts` protects route names (`event-types`, `m`, ...). A new top-level route means a new entry.
- `/reschedule/{token}` exists because Next.js cannot have a page and a POST handler at the same path as `/m/{token}/reschedule`.
- Manage tokens are stored as sha256 hashes already, so the real data layer needs no change there.
- Emails are sent through Resend now. The backend notes list what has only been proven against a local mock.

## Added in the backend pass

- `/privacy` page, written for what this app actually collects and which processors touch it.
- A time zone prompt: new accounts start in UTC, so "9 to 5" would silently mean UTC. The prompt moves the account and the default hours to the browser's zone in one click.
- Paid events: price on the editor and booking page, hand-off to Stripe, a confirming screen that refreshes until the webhook lands.

## Verification (frontend pass)

- 22 unit tests (`npm test`): slot engine including DST, buffers, limits, groups, overrides; data layer including a three-way booking race.
- Browser run (Playwright, real Chromium): 43 checks across guest booking from another time zone, a second guest blocked with 409, reschedule, cancel, host screens, validation, keyboard-only booking, no horizontal scroll at 390px, axe clean on the main screens. 6 more checks for loading, error, taken-slot and offline states.

## Verification (backend pass)

See `backend.md`: 92 tests against real Postgres and a 75-check browser run through the real OAuth callbacks, calendar, email, payment and deletion paths.
