# Parity report: the scheduling app against Calendly

Date: 2026-10-21. Scope: web, everything, for the owner's own use (`recon.md`). Build: the current `main` of `/home/user/replica`.

## Verdict: not shippable yet, by the rule in this skill

The rule: any must-have not done means not shippable. 8 of 17 must-haves are `partial`. Six of those eight are finished and tested but only against a local mock of Google, Resend or Stripe, so they count half until a real-key run (`manual-checklist.md`) confirms them. The other two are genuinely incomplete (Zoom links, filters on the meetings list). There are **no open S1 or S2 bugs** (`bugs.md`), so bugs are not what blocks it.

## The numbers

| measure | result | how to read it |
| --- | --- | --- |
| Feature parity | **48.7 / 100** | 62 features counted, must-haves 9 of 17 done. This is the number to quote. |
| If the real-service run passes | **60.0 / 100** | Same matrix with the 10 "built, mock-verified, real-keys pending" rows counted as done. A forecast, not a result. Must-haves would be 15 of 17. |
| Layout parity | **not measured** | See "Layout" below. There is nothing to compare against yet. |
| Overall | **48.7** (features only) | The skill's 80/20 blend with layout needs a layout score. |

Scored as the tool scores: must 3, should 2, could 1, partial half, `skip` and added features left out. Ten rows are `skip` on purpose (SSO, SCIM, AI notetaker, mobile apps, Calendly's brand and user base, and so on, with reasons in `features.csv`).

### Feature score by area, weakest first

Not built at all (0): meeting polls, one-off and single-use links, embeds, routing forms, the API and outbound webhooks, browser extensions, contacts, team admin, analytics.
Partly built: integrations 20, workflows 25, payments 33, scheduled events 38, notifications 45, onboarding 50, security 50, event types 63, availability 75, booking page 75.
Nothing is at 100, because every area contains at least one row that is either mock-verified only or missing a small piece.

## What works as well as the original, by evidence from the tests

These are rows marked `yes` and covered by the browser suite: weekly hours and date overrides, event types (one-on-one and group, colour, buffers, notice, limits, link names), the public booking page with time zone detection and a 12/24 hour switch, booking as a guest with no account, a confirmation page, reschedule and cancel from a private link, and buffers/limits/minimum notice applied to what guests can book. It also handles things the recon did not list: double booking is impossible (database guard), daylight-saving days, and half-hour zones.

## Layout

**Not measured.** The layout tool compares the clone's screenshot with the original's, same viewport, same state. There are no screenshots of the original: Calendly's site was blocked from this sandbox, so `replica/screens/` is empty (its README says so). A quick comparison of two of the clone's own screenshots taken at different widths scored 44, which only shows the tool needs matching viewports, so it is not counted.

To get the layout score, from the owner's own Calendly account (not anyone else's) take these at **1440x900** and **390x844**, in the same state as the clone's:

| screen | clone's matching screenshot (same state, take at the same size) |
| --- | --- |
| S03 Event types list (3 event types) | `/event-types` |
| S04 Event type editor | `/event-types/<id>` |
| S10 Meetings, upcoming tab with 2 meetings | `/meetings` |
| S11 Availability, Mon to Fri 9 to 5 | `/availability` |
| S30 Host profile page | `/<link-name>` |
| S31/S32 Booking page, a day chosen | `/<link-name>/<event>` |
| S34 Confirmation | `/booked/<token>` |
| S36 Manage / cancel | `/m/<token>` |

Then, for each: `python3 imgdiff.py replica/screens/S31.png replica/clone-screens/S31.png --out replica/diffs/S31.png --json > replica/diffs/S31.json`, and finally `python3 parity.py replica/features.csv --visual replica/diffs/*.json --markdown > replica/parity.md`. Do not chase a high number here: the clone's colours and styling are meant to differ, and the layout mode ignores colour for that reason.

## Behaviour differences

Sources: Calendly's help centre, blog and community pages as returned by search (the recon's sources). The original was **not** used or observed directly, so the "original does" column is what its documentation says, not what was seen. "Clone does" is from the browser tests.

| flow | original does (documented) | clone does | fix or keep |
| --- | --- | --- | --- |
| Core booking flow | pick a day, a time, a short form, done | 3 clicks (day, time, Book) plus two typed fields; no account | keep. Not more clicks than the original. |
| Guest adds other attendees | yes | no | build (could). |
| Custom questions on the form | a question builder | one fixed optional note, shown to the host in Meetings and in the email | build the builder (should). |
| Video link | Zoom, Google Meet, Teams and others added to the invite | Google Meet only, created with the calendar event; Zoom and phone are just text | decide whether Zoom is needed; keep Meet. |
| After a "just taken" conflict | not known | the guest returns to the times with their day kept (fixed in testing), but **name and email are cleared** and must be typed again | fix: keep the typed details (small, S4). |
| Reminders and follow-ups | editable Workflows, including SMS | one fixed email 24 hours before, no screen to change it, no SMS | build the Workflows screen (should); keep SMS out. |
| Reschedule / cancel | from the emailed link | same; the invite uses one calendar identity, so calendar apps update instead of duplicating | keep. |
| One-off and single-use links, polls, routing forms | yes | not built | build one-off links (should); polls and routing are could. |
| Embed on a website | inline, popup, floating | not built | build (should). |
| Payment | Stripe and PayPal at booking | Stripe only; the time is held 35 minutes while paying; backing out releases it at once; the host sees Paid / Waiting / Refunded | keep. Add PayPal only if wanted. |
| Where emails come from | Calendly's address, or the user's connected Gmail/Outlook | the owner's own verified domain through Resend | keep. |
| Teams, round robin, admin | yes | single host only | out of scope for own use. |
| Public API, outbound webhooks, Zapier | yes | none (an inbound Stripe webhook only) | build if other tools need to react to bookings (should). |

## Better than the original?

Not yet claimed. A few behaviours look at least as careful (nobody can pay for a time that was just taken, daylight-saving days are disambiguated, a calendar that cannot be checked stops bookings rather than guessing), but without observing the original they cannot honestly be called improvements. That is the job of `/replica-entrepreneur`, which reads what the original's users complain about, and it can only start once the clone is shippable.

## Top five to do next

1. **Run `manual-checklist.md` with real Google, Resend and Stripe accounts.** Worth +11.3 points and turns 6 of the 8 open must-haves into done. This is verification, not building, and everything else assumes it.
2. **Finish the last two must-haves** (+2.6): meeting filters on the Meetings screen, and either a live Zoom link or an explicit decision that Meet, phone and in-person are enough. After 1 and 2, all 17 must-haves are done.
3. **API, outbound webhooks and Zapier** (+5.2, the biggest single gap): lets the owner's other tools react to bookings. The inbound Stripe webhook and the job queue are already there to build on.
4. **Workflows screen** (+3.5): make the reminder and follow-up emails editable, with the existing engine behind it.
5. **One-off and single-use links** (+3.5), then embeds and the question builder (+2.6).

Also small and cheap: keep the guest's typed name and email after a "just taken" conflict.

## Appendix: the tools' own output

### Tool output: parity 48.7 / 100

features 48.7  (62 counted, must-haves 9 of 17 done)

Not shippable yet: 8 must-have features are not done.

### By area, weakest first
- meeting polls                  0.0  (1 features)
- single-use links               0.0  (2 features)
- embeds                         0.0  (1 features)
- routing                        0.0  (1 features)
- api                            0.0  (2 features)
- extensions                     0.0  (2 features)
- contacts                       0.0  (1 features)
- admin                          0.0  (5 features)
- analytics                      0.0  (1 features)
- integrations                  20.0  (3 features)
- workflows                     25.0  (3 features)
- payments                      33.3  (2 features)
- scheduled events              37.5  (2 features)
- notifications                 45.5  (5 features)
- onboarding                    50.0  (2 features)
- security                      50.0  (1 features)
- event types                   63.3  (8 features)
- availability                  75.0  (10 features)
- booking page                  75.0  (10 features)

### Missing, in build order
- [must] availability: Write bookings to a chosen calendar, partial  (built; verified against a local mock, real-keys run pending)
- [must] booking page: Guest reschedule or cancel from emailed link, partial  (Links are in the emails; built; verified against a local mock, real-keys run pending)
- [must] event types: Location options (Zoom / Meet / phone / in person / custom), partial  (Options, address and Google Meet links (Meet created with the calendar event); Zoom and phone are labels only, no live Zoom link)
- [must] notifications: Calendar invite (ICS) sent to invitee, partial  (Attached to confirmation and cancellation emails; built; verified against a local mock, real-keys run pending)
- [must] notifications: Confirmation email to host and invitee, partial  (built; verified against a local mock, real-keys run pending)
- [must] onboarding: Connect a calendar to read busy times (Google / Outlook / Office 365), partial  (Google only, no Outlook; built; verified against a local mock, real-keys run pending)
- [must] onboarding: Sign up with email or Google, partial  (Google sign-in with an email allow-list (no email/password); built; verified against a local mock, real-keys run pending)
- [must] scheduled events: Scheduled events list (upcoming / past / filters), partial  (Upcoming and past tabs, cancel with a reason, the guest's note, and a paid / waiting / refunded badge; no filters, no no-show marking)
- [should] api: Public REST API v2 (event types / scheduled events / invitees), no  (PAT + OAuth auth)
- [should] api: Webhooks invitee.created / canceled / rescheduled, no
- [should] embeds: Embed: inline / popup / floating widget, no  (Existence confirmed; behaviors unverified)
- [should] integrations: Zapier / webhooks, no
- [should] single-use links: One-off meetings (single-use link outside normal availability), no
- [should] single-use links: Single-use links from any event type (status Created / Viewed / Booked), no
- [should] availability: Check multiple calendars for conflicts, partial  (Several calendars across several Google accounts; built; verified against a local mock, real-keys run pending)
- [should] availability: Date range (rolling days or fixed range), partial  (Rolling days only; no fixed start/end dates)
- [should] availability: Meeting limits per day / week / month (per type or all types), partial  (Per event type only; not across all event types)
- [should] booking page: Custom invite form questions, partial  (One fixed optional notes field; no question builder (S05))
- [should] integrations: Video link auto-added to invite, partial  (Google Meet only, no Zoom; built; verified against a local mock, real-keys run pending)
- [should] notifications: Email reminders before meeting, partial  (24 hour reminder created for every new account; no screen to edit it (S13); built; verified against a local mock, real-keys run pending)
- [should] notifications: Follow-up emails after meeting, partial  (The job engine supports it; no screen to create one (S13))
- [should] payments: Stripe payments at booking, partial  (Checkout, signed webhooks, hold and release, late-payment refund; test mode; built; verified against a local mock, real-keys run pending)
- [should] security: Captcha on suspicious bookings, partial  (Database-backed rate limits per IP, per email and per host; no captcha)
- [should] workflows: Workflows: triggers before / after event / on booked / on canceled, partial  (All four triggers run; no screen to manage them (S13))
- [could] admin: Admin activity log, no
- [could] admin: Branding: logo + colors, no
- [could] admin: Groups (department / team), no
- [could] admin: Invitee data deletion requests, no  (Hosts can delete their whole account; no per-guest deletion screen)
- [could] admin: Team / org management (users, invites, roles: Owner, Admin, Group admin, Team manager, User), no  (Own use may be single user)
- [could] analytics: Admin analytics / dashboard, no
- [could] availability: Free/busy rules (allow booking over chosen events), no  (Match by title exception)
- [could] booking page: Add guests to a booking, no
- [could] booking page: Cancellation policy text, no
- [could] booking page: Redirect to external URL after booking, no  (Not confirmed in sources)
- [could] contacts: Contacts tab (meeting history with people), no
- [could] event types: Cannot convert event type kind (create new), no  (Constraint observed in help; optional to mirror)
- [could] event types: Collective event type (many hosts one invitee), no  (Teams)
- [could] event types: Round-robin event type (auto-assign host), no  (Teams; own use = only if multi-person)
- [could] event types: Secret / hidden event type, no  (Not confirmed in sources)
- [could] extensions: Chrome / Edge / Firefox extension, no
- [could] extensions: Outlook add-in, no
- [could] integrations: HubSpot / Mailchimp integrations, no
- [could] meeting polls: Meeting polls (vote on times), no  (Holds times on calendars until booked)
- [could] notifications: SMS reminders, no  (Reportedly Teams tier; needs SMS provider)
- [could] payments: PayPal payments at booking, no  (Standard tier)
- [could] routing: Routing forms (answers choose event type or URL), no  (Admin only; unqualified page option)
- [could] scheduled events: Mark invitee as no-show, no  (Seen in API)
- [could] workflows: Workflow templates (e.g. email reminder with cancel+reschedule links), no
- [could] workflows: Workflows send from user's connected Gmail/Outlook, no  (Sends from the app's own verified domain through Resend instead)

## Left out on purpose (not scored)
- CRM-owner routing (Salesforce / HubSpot): Needs CRM; out of scope for own use
- Salesforce integration: Teams tier; no need for own use
- Mobile apps (iOS / Android): Out of scope: web only
- SSO / SAML: Teams+; not needed for own use
- SCIM provisioning: Enterprise
- AI notetaker (recordings / transcripts / recaps): Paid add-on; huge scope; use a third-party tool
- AI email assistant (Callie): Beta
- Calendly trademarks / logo / copy / templates: Never copied
- Calendly user base and marketplace listings: Cannot be cloned
- Enterprise plan: Own use: no billing needed
