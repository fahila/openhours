# Test plan: scheduling app (a rebuild of Calendly's core features)

Build: b7853ec plus the fixes listed in `bugs.md`  Date: 2026-10-21  Env: local, fresh database each test, Google / Resend / Stripe replaced by a local mock

Cases are numbered `<flow>-<type><n>`. H happy path, E edge case, N negative case, A accessibility, M mobile or zoom, K keyboard or labels. Every case below with a spec file is automated and its title in the spec starts with the same id, so `npx playwright test -g F03-E2` runs one. Flows F01 to F12 follow `recon.md`; F13 and F14 and SEC cut across them.

**Result of the last full run: 181 of 182 cases listed here pass, 1 is skipped on purpose (the screenshot case only runs when asked).** (Chromium only. Case counts here are test titles; the zone matrix in F03-E9 runs once per zone.)

## What this plan covers from `recon.md`

| flow | name | status |
| --- | --- | --- |
| F01 | Host onboarding | tested |
| F02 | Host creates an event type | tested |
| F03 | Guest books a meeting | tested |
| F04 | Guest reschedules or cancels | tested |
| F05 | Host sets availability and overrides | tested |
| F06 | One-off / single-use link | **not built**, so not tested |
| F07 | Meeting poll | **not built**, so not tested |
| F08 | Workflow sends reminder / follow-up | tested for the built part (emails, invites, a 24 hour reminder; no screen to edit workflows) |
| F09 | Routing form | **not built**, so not tested |
| F10 | Admin manages the organisation | single-host account settings tested; team admin **not built** |
| F11 | Payment at booking | tested |
| F12 | API / outbound webhooks | **not built**, so not tested (the inbound Stripe webhook is under F11 and SEC) |

## Cases

| case | flow | type | scenario and what must happen | spec | result |
| --- | --- | --- | --- | --- | --- |
| F01-E1 | host signs in and connects a calendar | edge case |  | `f01-onboarding.spec.ts` | pass |
| F01-E2 | host signs in and connects a calendar | edge case | no prompt when the browser zone matches the account | `f01-onboarding.spec.ts` | pass |
| F01-E3 | host signs in and connects a calendar | edge case | signing in on a second device keeps one account and adds a session | `f01-onboarding.spec.ts` | pass |
| F01-E4 | host signs in and connects a calendar | edge case | the allow list ignores capitalisation and maps to the same account | `f01-onboarding.spec.ts` | pass |
| F01-E5 | host signs in and connects a calendar | edge case | two people with the same name get different link names | `f01-onboarding.spec.ts` | pass |
| F01-E6 | host signs in and connects a calendar | edge case | a tampered session cookie is signed out, not an error | `f01-onboarding.spec.ts` | pass |
| F01-E7 | host signs in and connects a calendar | edge case | every host screen sends a signed-out visitor to /login | `f01-onboarding.spec.ts` | pass |
| F01-E8 | host signs in and connects a calendar | edge case | an expired session during use sends you to sign in, not to an error page | `f01-onboarding.spec.ts` | pass |
| F01-E9 | host signs in and connects a calendar | edge case | removing a calendar permission in Google is detected and the reconnect button works | `f01-onboarding.spec.ts` | pass |
| F01-E10 | host signs in and connects a calendar | edge case |  | `f01-onboarding.spec.ts` | pass |
| F01-E11 | host signs in and connects a calendar | edge case | the same Google account cannot be connected twice | `f01-onboarding.spec.ts` | pass |
| F01-E12 | host signs in and connects a calendar | edge case | signed-in visitors skip the login page | `f01-onboarding.spec.ts` | pass |
| F01-E13 | host signs in and connects a calendar | edge case | mock outage: Google unreachable during sign-in gives a message, not a crash | `f01-onboarding.spec.ts` | pass |
| F01-H1 | host signs in and connects a calendar | happy path | first sign-in creates the account, starter event and hours; the zone prompt fixes the zone | `f01-onboarding.spec.ts` | pass |
| F01-H2 | host signs in and connects a calendar | happy path | connecting Google Calendar lists calendars with the primary one as target | `f01-onboarding.spec.ts` | pass |
| F01-N1 | host signs in and connects a calendar | negative case | an account that is not on the allow list is refused and nothing is created | `f01-onboarding.spec.ts` | pass |
| F01-N2 | host signs in and connects a calendar | negative case | an unverified Google email is refused | `f01-onboarding.spec.ts` | pass |
| F01-N3 | host signs in and connects a calendar | negative case | cancelling on Google | `f01-onboarding.spec.ts` | pass |
| F01-N4 | host signs in and connects a calendar | negative case | a forged or replayed callback does nothing | `f01-onboarding.spec.ts` | pass |
| F01-N5 | host signs in and connects a calendar | negative case | declining calendar access leaves the account unconnected with a clear message | `f01-onboarding.spec.ts` | pass |
| F01-N6 | host signs in and connects a calendar | negative case | a calendar callback started by someone else is refused | `f01-onboarding.spec.ts` | pass |
| F02-E1 | host creates and manages an event type | edge case | accents and emoji in the name work, and the link name is generated sensibly | `f02-event-types.spec.ts` | pass |
| F02-E2 | host creates and manages an event type | edge case | a name made only of emoji or symbols still creates an event with a working link | `f02-event-types.spec.ts` | pass |
| F02-E3 | host creates and manages an event type | edge case | two event types with the same name get different link names without an error | `f02-event-types.spec.ts` | pass |
| F02-E4 | host creates and manages an event type | edge case | typing an existing link name is refused with a clear message | `f02-event-types.spec.ts` | pass |
| F02-E5 | host creates and manages an event type | edge case | 120 characters are accepted and wrap; 121 are refused | `f02-event-types.spec.ts` | pass |
| F02-E6 | host creates and manages an event type | edge case | HTML in a name is shown as text, never run | `f02-event-types.spec.ts` | pass |
| F02-E7 | host creates and manages an event type | edge case | a group event asks for places; fewer than 2 is refused | `f02-event-types.spec.ts` | pass |
| F02-E8 | host creates and manages an event type | edge case | in-person and custom places need details | `f02-event-types.spec.ts` | pass |
| F02-E9 | host creates and manages an event type | edge case | double-clicking Create makes one event type | `f02-event-types.spec.ts` | pass |
| F02-E10 | host creates and manages an event type | edge case | numbers that make no sense are refused: negative buffers, decimals, huge limits | `f02-event-types.spec.ts` | pass |
| F02-E11 | host creates and manages an event type | edge case | prices: too low, not a number and group events are refused; a valid price is stored in cents | `f02-event-types.spec.ts` | pass |
| F02-E12 | host creates and manages an event type | edge case | changing the link name breaks the old link, the new one works, and guests | `f02-event-types.spec.ts` | pass |
| F02-E13 | host creates and manages an event type | edge case | a failed save keeps what was typed | `f02-event-types.spec.ts` | pass |
| F02-E14 | host creates and manages an event type | edge case | two tabs editing the same event: the last save wins and nothing breaks | `f02-event-types.spec.ts` | pass |
| F02-E15 | host creates and manages an event type | edge case | an event type can be created with the keyboard alone | `f02-event-types.spec.ts` | pass |
| F02-E16 | host creates and manages an event type | edge case | colour choice is saved and shown on the booking page | `f02-event-types.spec.ts` | pass |
| F02-E17 | host creates and manages an event type | edge case | limits and buffers are applied to what guests can book | `f02-event-types.spec.ts` | pass |
| F02-H1 | host creates and manages an event type | happy path | create an event type; it is listed and its public link works | `f02-event-types.spec.ts` | pass |
| F02-H2 | host creates and manages an event type | happy path | editing the name, duration and place shows up on the booking page | `f02-event-types.spec.ts` | pass |
| F02-H3 | host creates and manages an event type | happy path | turning an event off hides it and its link; turning it on restores both | `f02-event-types.spec.ts` | pass |
| F02-H4 | host creates and manages an event type | happy path | an event type with no bookings can be deleted | `f02-event-types.spec.ts` | pass |
| F02-N1 | host creates and manages an event type | negative case | deleting an event type that has bookings is refused with a reason | `f02-event-types.spec.ts` | pass |
| F02-N2 | host creates and manages an event type | negative case | the  | `f02-event-types.spec.ts` | pass |
| F03-A1 | guest books a meeting | accessibility | no accessibility violations on the pick, details, error and confirmation steps | `f03-booking.spec.ts` | pass |
| F03-E1 | guest books a meeting | edge case | someone else takes the slot while you fill the form: clear message, pick again, one booking | `f03-booking.spec.ts` | pass |
| F03-E2 | guest books a meeting | edge case | two guests press  | `f03-booking.spec.ts` | pass |
| F03-E3 | guest books a meeting | edge case | double-clicking  | `f03-booking.spec.ts` | pass |
| F03-E4 | guest books a meeting | edge case | the back button after booking returns to the booking page without booking again | `f03-booking.spec.ts` | pass |
| F03-E5 | guest books a meeting | edge case | refreshing while typing your details starts over cleanly | `f03-booking.spec.ts` | pass |
| F03-E6 | guest books a meeting | edge case | on a slow connection a loading state shows, then the times | `f03-booking.spec.ts` | pass |
| F03-E7 | guest books a meeting | edge case | offline when submitting: a message, the form is kept, and retrying works once back online | `f03-booking.spec.ts` | pass |
| F03-E8 | guest books a meeting | edge case | the 24-hour switch changes how times are written | `f03-booking.spec.ts` | pass |
| F03-E9 | guest books a meeting | edge case | guest in Pacific/Kiritimati: the booked time is shown the same everywhere and stored in UTC | `f03-booking.spec.ts` | pass |
| F03-E9 | guest books a meeting | edge case | guest in Pacific/Pago_Pago: the booked time is shown the same everywhere and stored in UTC | `f03-booking.spec.ts` | pass |
| F03-E9 | guest books a meeting | edge case | guest in Asia/Kathmandu: the booked time is shown the same everywhere and stored in UTC | `f03-booking.spec.ts` | pass |
| F03-E9 | guest books a meeting | edge case | guest in America/St_Johns: the booked time is shown the same everywhere and stored in UTC | `f03-booking.spec.ts` | pass |
| F03-E10 | guest books a meeting | edge case | accents, emoji and a very long name are kept exactly and shown to the host | `f03-booking.spec.ts` | pass |
| F03-E11 | guest books a meeting | edge case | HTML in a guest | `f03-booking.spec.ts` | pass |
| F03-E12 | guest books a meeting | edge case | missing or invalid details are explained next to the field and nothing is lost | `f03-booking.spec.ts` | pass |
| F03-E13 | guest books a meeting | edge case | month buttons work in both directions and never leave the page broken | `f03-booking.spec.ts` | pass |
| F03-E14 | guest books a meeting | edge case | the minimum notice hides times that are too soon | `f03-booking.spec.ts` | pass |
| F03-E15 | guest books a meeting | edge case | pages that should not exist return 404: turned-off event, unknown event, unknown host | `f03-booking.spec.ts` | pass |
| F03-E16 | guest books a meeting | edge case | a rate-limit answer is explained to the guest | `f03-booking.spec.ts` | pass |
| F03-E17 | guest books a meeting | edge case | a group event shows places left and closes when full | `f03-booking.spec.ts` | pass |
| F03-E18 | guest books a meeting | edge case | a hidden event is not listed but its link works | `f03-booking.spec.ts` | pass |
| F03-E19 | guest books a meeting | edge case | changing the guest | `f03-booking.spec.ts` | pass |
| F03-E20 | guest books a meeting | edge case | a long description with line breaks wraps and keeps its breaks | `f03-booking.spec.ts` | pass |
| F03-H1 | guest books a meeting | happy path | a guest in another zone books; the confirmation, the host | `f03-booking.spec.ts` | pass |
| F03-H2 | guest books a meeting | happy path | what the guest writes in the note is visible to the host in the app | `f03-booking.spec.ts` | pass |
| F03-H3 | guest books a meeting | happy path | the host profile page lists active events and leads to the booking page | `f03-booking.spec.ts` | pass |
| F03-K1 | guest books a meeting | keyboard / labels | a whole booking can be made with the keyboard alone | `f03-booking.spec.ts` | pass |
| F03-M1 | guest books a meeting | mobile / zoom | the whole flow works at phone width with no sideways scrolling at any step | `f03-booking.spec.ts` | pass |
| F03-N1 | guest books a meeting | negative case | the booking API refuses times outside the hours, in the past, malformed times, bad zones and wrong methods | `f03-booking.spec.ts` | pass |
| F03-N2 | guest books a meeting | negative case | the slots API refuses nonsense ranges | `f03-booking.spec.ts` | pass |
| F04-A1 | guest changes or cancels a booking | accessibility | manage, cancelled, finished, dialog and reschedule screens pass an accessibility scan | `f04-manage.spec.ts` | pass |
| F04-E1 | guest changes or cancels a booking | edge case | moving to the very same time is allowed and changes nothing for the host | `f04-manage.spec.ts` | pass |
| F04-E2 | guest changes or cancels a booking | edge case | a cancelled booking | `f04-manage.spec.ts` | pass |
| F04-E3 | guest changes or cancels a booking | edge case | someone takes the new time while you are choosing it: message, nothing is lost | `f04-manage.spec.ts` | pass |
| F04-E4 | guest changes or cancels a booking | edge case | a meeting that already happened can no longer be changed | `f04-manage.spec.ts` | pass |
| F04-E5 | guest changes or cancels a booking | edge case | cancelling from two tabs at once ends cancelled with no error | `f04-manage.spec.ts` | pass |
| F04-E6 | guest changes or cancels a booking | edge case | a rescheduled booking keeps the guest | `f04-manage.spec.ts` | pass |
| F04-E7 | guest changes or cancels a booking | edge case | one guest leaving a group leaves the others booked, and the host sees the smaller group | `f04-manage.spec.ts` | pass |
| F04-E8 | guest changes or cancels a booking | edge case | the cancel dialog closes with Escape and  | `f04-manage.spec.ts` | pass |
| F04-E9 | guest changes or cancels a booking | edge case | a group guest | `f04-manage.spec.ts` | pass |
| F04-E10 | guest changes or cancels a booking | edge case | tampered, truncated and oversized links are 404, not errors | `f04-manage.spec.ts` | pass |
| F04-E11 | guest changes or cancels a booking | edge case | the manage link works from any device, with no sign-in | `f04-manage.spec.ts` | pass |
| F04-E12 | guest changes or cancels a booking | edge case | rescheduling across a month boundary works | `f04-manage.spec.ts` | pass |
| F04-H1 | guest changes or cancels a booking | happy path | reschedule: the guest picks a new time, the old time opens up, the host and guest are told once | `f04-manage.spec.ts` | pass |
| F04-H2 | guest changes or cancels a booking | happy path | cancel with a reason: the host sees it, the time opens up, both are emailed | `f04-manage.spec.ts` | pass |
| F04-M1 | guest changes or cancels a booking | mobile / zoom | manage and reschedule pages fit a phone | `f04-manage.spec.ts` | pass |
| F04-N1 | guest changes or cancels a booking | negative case | a cancel reason over 500 characters is refused, not truncated silently | `f04-manage.spec.ts` | pass |
| F04-N2 | guest changes or cancels a booking | negative case | guessing links is rate limited | `f04-manage.spec.ts` | pass |
| F05-E1 | host sets availability | edge case | overlapping ranges on one day are refused with a reason, nothing is saved | `f05-availability.spec.ts` | pass |
| F05-E2 | host sets availability | edge case | an end before the start is refused | `f05-availability.spec.ts` | pass |
| F05-E3 | host sets availability | edge case | two ranges in a day leave a gap, for lunch | `f05-availability.spec.ts` | pass |
| F05-E4 | host sets availability | edge case | removing every range for a day marks it unavailable | `f05-availability.spec.ts` | pass |
| F05-E5 | host sets availability | edge case | moving the hours to another zone moves the times: 9 to 5 in Los Angeles is 12 to 8 in New York | `f05-availability.spec.ts` | pass |
| F05-E6 | host sets availability | edge case | closing a day that already has a booking keeps the booking and offers nothing more | `f05-availability.spec.ts` | pass |
| F05-E7 | host sets availability | edge case | the day clocks go back (New York, 1 Nov 2026): 3 real hours is 6 slots, none twice | `f05-availability.spec.ts` | pass |
| F05-E8 | host sets availability | edge case | the day clocks go forward (New York, 14 Mar 2027): the missing hour is never offered | `f05-availability.spec.ts` | pass |
| F05-E9 | host sets availability | edge case | a host in a half-hour zone (Kolkata) gets correct instants | `f05-availability.spec.ts` | pass |
| F05-E10 | host sets availability | edge case | a southern-hemisphere host across the end of daylight time (Sydney, 4 Apr 2027) | `f05-availability.spec.ts` | pass |
| F05-E11 | host sets availability | edge case | saving without changes still says saved, and the hours survive a reload | `f05-availability.spec.ts` | pass |
| F05-E12 | host sets availability | edge case | one set of hours drives every event type | `f05-availability.spec.ts` | pass |
| F05-E13 | host sets availability | edge case | the profile zone and the hours zone are separate: changing the profile does not move working hours | `f05-availability.spec.ts` | pass |
| F05-E14 | host sets availability | edge case | a date change for a past or far-future date is accepted and does no harm | `f05-availability.spec.ts` | pass |
| F05-E15 | host sets availability | edge case | the weekly editor works with the keyboard and every time field has a name | `f05-availability.spec.ts` | pass |
| F05-H1 | host sets availability | happy path | changing the weekly hours in the screen changes what guests can book | `f05-availability.spec.ts` | pass |
| F05-H2 | host sets availability | happy path | a day off removes that day, and only that day | `f05-availability.spec.ts` | pass |
| F05-H3 | host sets availability | happy path | special hours on a normally closed day open that day | `f05-availability.spec.ts` | pass |
| F05-N1 | host sets availability | negative case | an impossible override date is refused, not a server error | `f05-availability.spec.ts` | pass |
| F08-E1 | emails, invites and reminders | edge case | a booking made less than a day ahead gets no reminder, only the confirmation | `f08-notifications.spec.ts` | pass |
| F08-E2 | emails, invites and reminders | edge case | when the email service fails twice, the guest still gets exactly one confirmation | `f08-notifications.spec.ts` | pass |
| F08-E3 | emails, invites and reminders | edge case | moving a meeting updates the guest | `f08-notifications.spec.ts` | pass |
| F08-E4 | emails, invites and reminders | edge case | cancelling sends a cancel notice that targets the same calendar event, even after a move | `f08-notifications.spec.ts` | pass |
| F08-E5 | emails, invites and reminders | edge case | names with accents, emoji and apostrophes arrive intact in subject and body, with a text version | `f08-notifications.spec.ts` | pass |
| F08-E6 | emails, invites and reminders | edge case | running the jobs again and again never sends anything twice | `f08-notifications.spec.ts` | pass |
| F08-E7 | emails, invites and reminders | edge case | a cancelled meeting is never reminded about | `f08-notifications.spec.ts` | pass |
| F08-E8 | emails, invites and reminders | edge case | the jobs endpoint needs the secret, in a header, for GET and POST alike | `f08-notifications.spec.ts` | pass |
| F08-E9 | emails, invites and reminders | edge case | guest emails reply to the host | `f08-notifications.spec.ts` | pass |
| F08-E10 | emails, invites and reminders | edge case | line breaks and control characters in a guest | `f08-notifications.spec.ts` | pass |
| F08-E11 | emails, invites and reminders | edge case | a dead job (the email service down for good) stays visible with its reason and nothing is sent | `f08-notifications.spec.ts` | pass |
| F08-H1 | emails, invites and reminders | happy path | the guest | `f08-notifications.spec.ts` | pass |
| F08-H2 | emails, invites and reminders | happy path | the invite attachment is a valid calendar event with the right times | `f08-notifications.spec.ts` | pass |
| F08-H3 | emails, invites and reminders | happy path | a reminder is sent a day before with a working manage link | `f08-notifications.spec.ts` | pass |
| F10-A1 | host manages their account (settings) | accessibility | the settings screen passes an accessibility scan, with and without errors | `f10-settings.spec.ts` | pass |
| F10-E1 | host manages their account (settings) | edge case | reserved words, spaces, capitals, symbols and too-short link names are refused with a reason | `f10-settings.spec.ts` | pass |
| F10-E2 | host manages their account (settings) | edge case | a link name another person already uses is refused | `f10-settings.spec.ts` | pass |
| F10-E3 | host manages their account (settings) | edge case | a failed save keeps what you typed (no silent reset to the old values) | `f10-settings.spec.ts` | pass |
| F10-E4 | host manages their account (settings) | edge case | an empty name and an over-long welcome message are refused | `f10-settings.spec.ts` | pass |
| F10-E5 | host manages their account (settings) | edge case | changing the link name keeps old guests | `f10-settings.spec.ts` | pass |
| F10-E6 | host manages their account (settings) | edge case |  | `f10-settings.spec.ts` | pass |
| F10-E7 | host manages their account (settings) | edge case | deleting the account needs the exact email, and removes everything including paid bookings | `f10-settings.spec.ts` | pass |
| F10-H1 | host manages their account (settings) | happy path | name, link name, zone and welcome message save and show on the public page | `f10-settings.spec.ts` | pass |
| F10-N1 | host manages their account (settings) | negative case |  | `f10-settings.spec.ts` | pass |
| F11-A1 | guest pays to book | accessibility | the paid booking screens pass an accessibility scan | `f11-payments.spec.ts` | pass |
| F11-E1 | guest pays to book | edge case | money that arrives after the hold ran out is refunded and the booking stays cancelled | `f11-payments.spec.ts` | pass |
| F11-E2 | guest pays to book | edge case | paying twice, or the payment event arriving twice, gives one booking and one email and no refund | `f11-payments.spec.ts` | pass |
| F11-E3 | guest pays to book | edge case | Stripe being down: the guest is told nothing was charged, the time is free again, and a retry works | `f11-payments.spec.ts` | pass |
| F11-E4 | guest pays to book | edge case | the waiting page confirms by itself once the payment lands | `f11-payments.spec.ts` | pass |
| F11-E5 | guest pays to book | edge case | while someone is paying, the time cannot be taken by another guest | `f11-payments.spec.ts` | pass |
| F11-E6 | guest pays to book | edge case | cancelling before paying closes the Stripe session and frees the time | `f11-payments.spec.ts` | pass |
| F11-E7 | guest pays to book | edge case | moving a paid booking does not charge again | `f11-payments.spec.ts` | pass |
| F11-E8 | guest pays to book | edge case | changing the price later does not change what an earlier guest paid | `f11-payments.spec.ts` | pass |
| F11-H1 | guest pays to book | happy path | pay through checkout: pending while paying, confirmed after, one confirmation email, host sees it paid | `f11-payments.spec.ts` | pass |
| F11-H2 | guest pays to book | happy path | prices are written in the right currency | `f11-payments.spec.ts` | pass |
| F11-N1 | guest pays to book | negative case | backing out of the payment page frees the time at once and says so | `f11-payments.spec.ts` | pass |
| F13-A1 | every screen: accessibility, phone width, keyboard | accessibility | every host screen has no accessibility violations (with data on screen) | `f13-a11y-responsive.spec.ts` | pass |
| F13-A2 | every screen: accessibility, phone width, keyboard | accessibility | the guest screens and error pages have no accessibility violations | `f13-a11y-responsive.spec.ts` | pass |
| F13-A3 | every screen: accessibility, phone width, keyboard | accessibility | dialogs: opening moves focus inside, Tab stays inside, Escape closes and focus returns | `f13-a11y-responsive.spec.ts` | pass |
| F13-A4 | every screen: accessibility, phone width, keyboard | accessibility | the delete and disconnect dialogs are labelled and keyboard operable | `f13-a11y-responsive.spec.ts` | pass |
| F13-K1 | every screen: accessibility, phone width, keyboard | keyboard / labels | the whole host nav and the main actions are reachable by keyboard with a visible focus | `f13-a11y-responsive.spec.ts` | pass |
| F13-K2 | every screen: accessibility, phone width, keyboard | keyboard / labels | colour is never the only signal: statuses and errors always have text | `f13-a11y-responsive.spec.ts` | pass |
| F13-K3 | every screen: accessibility, phone width, keyboard | keyboard / labels | page titles say where you are | `f13-a11y-responsive.spec.ts` | pass |
| F13-M1 | every screen: accessibility, phone width, keyboard | mobile / zoom | every host and guest screen fits a 390px phone with no sideways scrolling | `f13-a11y-responsive.spec.ts` | pass |
| F13-M2 | every screen: accessibility, phone width, keyboard | mobile / zoom | on a phone the host navigation is reachable and every nav link works | `f13-a11y-responsive.spec.ts` | pass |
| F13-M3 | every screen: accessibility, phone width, keyboard | mobile / zoom | text can be enlarged to 200% without losing content or overflowing | `f13-a11y-responsive.spec.ts` | pass |
| F14-E1 | time zones and daylight saving | edge case | the day clocks go back, the same wall-clock time happens twice: the page tells the two apart | `f14-timezones.spec.ts` | pass |
| F14-E2 | time zones and daylight saving | edge case | a booking made in the repeated hour is stored at the right instant and shown with its zone name | `f14-timezones.spec.ts` | pass |
| F14-E3 | time zones and daylight saving | edge case | half-hour and 45-minute zones read correctly: a Kolkata host seen from Kathmandu | `f14-timezones.spec.ts` | pass |
| F14-E4 | time zones and daylight saving | edge case | changing the hours | `f14-timezones.spec.ts` | pass |
| F14-E5 | time zones and daylight saving | edge case | the host sees meetings in their profile zone, whatever the guest | `f14-timezones.spec.ts` | pass |
| F14-E6 | time zones and daylight saving | edge case | a guest | `f14-timezones.spec.ts` | pass |
| F14-H1 | time zones and daylight saving | happy path | a London host and a New York guest: the gap between them changes on 25 Oct and 1 Nov, and the page follows | `f14-timezones.spec.ts` | pass |
| F15-S1 | screenshots for a human to look over | screenshots | screenshots of every screen, desktop and phone | `f15-screens.spec.ts` | skipped (needs SHOTS) |
| SEC-1 | security and isolation between hosts | security | a second host cannot see or open the first host | `f12-security.spec.ts` | pass |
| SEC-2 | security and isolation between hosts | security | a signed-in host cannot cancel, change or delete another host | `f12-security.spec.ts` | pass |
| SEC-3 | security and isolation between hosts | security | signing a host out does not sign anyone else out, and a signed-out browser cannot reach host pages | `f12-security.spec.ts` | pass |
| SEC-4 | security and isolation between hosts | security | odd addresses are 404, never a crash or a leak: traversal, NUL, very long, encoded slashes | `f12-security.spec.ts` | pass |
| SEC-5 | security and isolation between hosts | security | secrets and source are not served: env files, git data, migrations, tests, package files | `f12-security.spec.ts` | pass |
| SEC-6 | security and isolation between hosts | security | no secret value appears in any script the browser downloads | `f12-security.spec.ts` | pass |
| SEC-7 | security and isolation between hosts | security | every kind of page carries the security headers; private links are not cached or sent as referrers; no wildcard CORS | `f12-security.spec.ts` | pass |
| SEC-8 | security and isolation between hosts | security | pages with private links, and the host app, ask search engines not to index them | `f12-security.spec.ts` | pass |
| SEC-9 | security and isolation between hosts | security | the session cookie is httpOnly and SameSite, and cannot be read by page scripts | `f12-security.spec.ts` | pass |
| SEC-10 | security and isolation between hosts | security | hostile text in every host-editable field is stored and shown as text, never run or fatal | `f12-security.spec.ts` | pass |
| SEC-11 | security and isolation between hosts | security | the login page never redirects to an address taken from the URL | `f12-security.spec.ts` | pass |
| SEC-12 | security and isolation between hosts | security | hammering the sign-in start is rate limited | `f12-security.spec.ts` | pass |
| SEC-13 | security and isolation between hosts | security | the Stripe webhook accepts only signed events | `f12-security.spec.ts` | pass |
| SEC-14 | security and isolation between hosts | security | an address removed from the allow list loses access at once (existing session ends) | `f12-security.spec.ts` | pass |
| SEC-15 | security and isolation between hosts | security | booking data is not leaked through the public slots or profile pages | `f12-security.spec.ts` | pass |
| SMOKE-1 | the test stack starts and a host can sign in | check | the stack is up: sign in, land on event types, starter data exists | `00-smoke.spec.ts` | pass |

## Not automated (manual, needs real services or a real person)

See `manual-checklist.md`: real Google sign-in and calendar, real email delivery and calendar apps, Stripe test cards including `4000 0000 0000 0002`, other browsers and devices, a screen reader, backups. **None of those have been run.**

## Edge cases from the standard list and where they are covered

| edge case | covered by |
| --- | --- |
| empty input | F03-E12, F02 (name), F10-E4 |
| very long input | F02-E5, F03-E10, F03-E20, F10-E4 |
| emoji and accents | F02-E1, F02-E2, F03-E10, F08-E5 |
| two tabs at once | F02-E14, F03-E2, F04-E5 |
| double click on submit | F02-E9, F03-E3 |
| back button mid-flow | F03-E4 |
| refresh mid-flow | F03-E5 |
| slow network | F03-E6 |
| offline | F03-E7 |
| expired session | F01-E8, SEC-14 |
| second user's data | SEC-1, SEC-2, F02, `tests/data.test.ts` |
| time zones and daylight saving | F14, F05-E5 to E10, F03-E9 |
| mobile width | F03-M1, F04-M1, F13-M1 to M3 |
| keyboard only | F02-E15, F03-K1, F05-E15, F13-K1 |
| screen reader labels | F13-A1 to A4 and every other `-A` case (axe), F13-K3 |
| wrong password | not applicable: sign-in is Google only |
| card declined | F11-N1 (back-out) and F11-E3 (Stripe down) automated; the real decline card is in the manual checklist |
| permission denied | F01-N1, F01-N2, F01-E7, SEC-1 |
| deleted record | F02-N1, F03-E15, F04-E2, F10-E7 |
