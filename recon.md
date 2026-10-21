# Recon map: Calendly (web)

Scope: Everything on web (host dashboard, public booking pages, teams, workflows, routing, admin, API). Mobile apps excluded.
For: The user's own business use (internal scheduling tool, not for sale).
Date: 2026-10-21

## Evidence limits (read first)

- calendly.com and help.calendly.com were blocked by this session's network policy (proxy 403 / ENOTFOUND). No page was opened directly and no screenshots exist.
- Everything below comes from web-search result summaries of Calendly's help center, blog, community and a few third-party sources. Screens and flows are reconstructed from those summaries plus the feature set they describe.
- Confidence is marked per item. Anything marked "unverified" must be checked in the user's own account before replica-build relies on it.
- Pricing and plan gating conflict between sources. Plan tiers are informational only here. For own use, gating does not matter: build what you need.
- Terms check: Calendly's terms were not read. For own internal use this is low risk, but if the clone is ever sold, read the terms about building a competing product first.

## Sources

| # | source | URL | notes |
| --- | --- | --- | --- |
| 1 | help center: get to know Calendly | https://calendly.com/help/get-to-know-calendly | event type kinds, availability, routing, workflows overview (search summary only) |
| 2 | help: customize event types | https://calendly.com/help/how-to-customize-your-event-types | primary settings, more options, invitee form, confirmation page |
| 3 | help: meeting limits | https://calendly.com/help/how-to-set-meeting-limits | per-day/week/month limits |
| 4 | help: one-off meetings | https://help.calendly.com/hc/en-us/articles/4406031977623 | single-use links |
| 5 | help: troubleshoot available times | https://calendly.com/help/how-to-troubleshoot-available-times-that-should-be-unavailable | conflict checking behaviour |
| 6 | help: user roles | https://calendly.com/help/user-roles-and-permissions | Owner / Admin / Group admin / Team manager / User |
| 7 | help: admin center, company admin guide | https://calendly.com/help/admin-center | users, groups, branding, activity log |
| 8 | help: AI at Calendly | https://calendly.com/help/ai-at-calendly | Notetaker, Callie |
| 9 | blog: routing forms | https://calendly.com/blog/routing-forms | form to event type or URL |
| 10 | learn: workflows | https://calendly.com/learn/calendly-workflows | triggers, email/SMS |
| 11 | changelog March 13, 2026 | https://community.calendly.com/featured-tips-tricks-32/change-log-march-13-2026-5463 | captcha, Notetaker updates |
| 12 | pricing page | https://calendly.com/pricing | not opened; tiers via third-party summaries (Orb, Cal.com, Wise) |
| 13 | developer docs | https://developer.calendly.com | not opened; endpoints via third-party summaries |
| 14 | Chrome extension | https://calendly.com/integration/calendly-for-chrome | meetings and contacts tabs |

Not yet used and worth doing from the user's own account: the live dashboard, every booking-page state, the real confirmation and reminder emails, the Embed options.

## Core loop

A host publishes event types tied to their availability and connected calendar; a guest opens the link, picks a slot in their own time zone, fills a short form, and both get a confirmation with reschedule and cancel links.

## Screens

Confidence: medium (reconstructed, not seen). "Seen" column is empty on purpose: no states were observed.

Host app:

| ID | screen | route / how to reach | purpose | key components | states expected |
| --- | --- | --- | --- | --- | --- |
| S01 | Sign up / log in | /signup, /login | account creation | email, Google, password | empty, error |
| S02 | Onboarding: connect calendar | first-run | link Google/Outlook, set time zone | provider buttons, calendar checklist | empty, connected, error |
| S03 | Scheduling: event types list | Scheduling tab | list and manage event types | cards, copy link, toggle, share | empty, filled |
| S04 | Event type editor | edit event type | name, duration, location, availability, hosts, limits, buffers | form, panels, preview | new, editing, error |
| S05 | Invitee form editor | event type, More options | custom questions | question list, field types | default, edited |
| S06 | Confirmation page settings | event type, More options | what guest sees after booking | text, redirect (unverified) | default |
| S07 | Single-use links tab | Scheduling, Single-use links | create/manage one-time links | status Created / Viewed / Booked | empty, filled |
| S08 | One-off meeting creator | single-use links | offer times outside normal availability | time picker, reserve toggle | new |
| S09 | Meeting polls | Scheduling, Meeting polls | collect votes on offered times | slot list, vote results | empty, collecting, closed |
| S10 | Scheduled events | Meetings tab | upcoming / past / filters | list, detail drawer, cancel, no-show | empty, filled |
| S11 | Availability | Availability tab | weekly hours, date overrides, calendar conflict choices | schedule editor, override calendar | default, overrides |
| S12 | Calendar connections | Integrations, Calendars | add and pick conflict calendars | provider list, calendar picker | connected, disconnected, error |
| S13 | Workflows list + editor | Workflows | automate emails/SMS | trigger, action, event type assignment | empty, filled, draft |
| S14 | Integrations | Integrations & apps | Stripe, PayPal, Zapier, CRMs | cards | connected, off |
| S15 | Routing forms (editor + routes) | Routing | screen and route leads | form builder, route rules | empty, filled |
| S16 | Contacts | Contacts tab | history with people | list, detail | empty, filled |
| S17 | Account settings | profile | name, link, time zone, branding | form | default |
| S18 | Admin center: users | Admin | invite/remove, roles, CSV export | table, bulk actions | empty, filled |
| S19 | Admin center: groups | Admin | group admins, members | tree/list | empty, filled |
| S20 | Admin center: branding | Admin | logo and colors | upload, color picker | default |
| S21 | Admin center: activity log + analytics | Admin | audit trail, usage | table, charts | empty, filled |
| S22 | Billing / plans | Account | plan selection | plan cards | n/a for own use |

Guest side:

| ID | screen | route / how to reach | purpose | key components | states expected |
| --- | --- | --- | --- | --- | --- |
| S30 | Host profile page | /{user} | lists event types | cards | one, many, none |
| S31 | Booking page: pick date | /{user}/{event} | choose day | month calendar, time zone select | no slots, loading |
| S32 | Booking page: pick time | same | choose time | slot list, 12/24h toggle | slot taken, past limit |
| S33 | Details form | same | name, email, answers, guests | form, add guests | error, filled |
| S34 | Confirmation | after booking | summary, add to calendar | summary card, links | default, redirect |
| S35 | Reschedule | link in email | pick new time | like S31/S32 | too late (policy) |
| S36 | Cancel | link in email | cancel with reason | form | confirmed, already past |
| S37 | One-off / single-use page | single link | book offered times | slot list | expired, already booked ("someone has already used this link") |
| S38 | Poll voting page | poll link | vote on times | slot checkboxes | closed |
| S39 | Routing form (guest) | form link | answer screening questions | form | qualified, disqualified thank-you |
| S40 | Embed wrappers | script on host site | inline / popup / floating | iframe-like widget | unverified |

Emails (not screens, but recon them too): confirmation, reminder, follow-up, cancellation, reschedule, poll vote notice, conflict alert.

## Flows

```
F01 Host onboarding
    S01 -> S02 -> S11 -> S03
    edge: calendar connect denied, wrong time zone

F02 Host creates an event type
    S03 -> S04 -> (S05, S06 optional) -> copy link
    edge: duplicate slug, no availability assigned

F03 Guest books a meeting (core loop)
    S31 -> S32 -> S33 -> S34
    happy path clicks: 4 (date, time, submit form, done)
    edge: no slots this week, guest time zone differs, slot taken while filling form,
          booking over limits, minimum-notice violation

F04 Guest reschedules or cancels
    email link -> S35 or S36 -> S34
    edge: already past, policy blocks, host calendar changed

F05 Host sets availability and overrides
    S11 -> S12
    edge: busy event on connected calendar, override for one date, free/busy exception

F06 Host sends a one-off / single-use link
    S07 -> S08 -> S37
    edge: link used twice, reserved placeholders cleared

F07 Meeting poll
    S09 -> S38 -> host picks time -> booking created
    edge: ties, unused slots released

F08 Workflow sends reminder / follow-up
    S13 -> triggers on booked / before / after / canceled
    edge: SMS deliverability, workflow not applied to one-off meetings

F09 Routing form
    S39 -> route to event type (S31) or URL or thank-you
    edge: disqualified lead

F10 Admin manages org
    S18 -> S19 -> S20 -> S21
    edge: role limits, group admin scope

F11 Payment at booking
    S33 -> Stripe/PayPal -> S34
    edge: payment fails, refund on cancel (unverified)

F12 API / webhook consumer
    create webhook subscription -> receive invitee.created / canceled / rescheduled
```

## Components

| component | variants | states | used on |
| --- | --- | --- | --- |
| Button | primary, secondary, ghost, danger | default, hover, focus, disabled, loading | all |
| Text / email / phone / dropdown / radio / checkbox | form fields | default, focus, error, disabled | S04, S05, S33 |
| Month calendar | guest, host date-override | today, selected, disabled, hover | S11, S31, S35 |
| Time slot list | 12h / 24h | available, selected, taken | S32, S35, S37 |
| Time zone selector | searchable | default, changed | S31 |
| Event type card | one-on-one, group, collective, round robin | active, inactive, secret (unverified) | S03, S30 |
| Toggle | | on, off | S03 |
| Tabs / sidebar nav | | active, inactive | host app |
| Modal / drawer | | open, closed | S10, S13 |
| Toast | success, error | | host app |
| Data table | sortable, filters | empty, loading, filled | S10, S18, S21 |
| Status badge | Created / Viewed / Booked, confirmed / canceled | | S07, S10 |
| Color picker, image upload | | | S04, S20 |
| Workflow step builder | trigger, action | | S13 |
| Form builder with routes | | | S15 |
| Embed code snippet block | inline, popup, floating | copy | S03 |

## Inferred data model

```
User         id, email, name, slug, time_zone, plan, role (owner|admin|group_admin|team_manager|user), organization_id
             evidence: S01, S17, S18, help "User roles and permissions"
             confidence: high

Organization id, name, branding (logo, colors), settings
             evidence: S18, S20
             confidence: medium

Group        id, organization_id, name, members[]
             evidence: S19
             confidence: medium

CalendarConnection  id, user_id, provider (google|outlook|office365), account, check_conflicts_ids[], write_to_calendar_id
             evidence: S02, S12, help "troubleshoot available times"
             confidence: high

AvailabilitySchedule  id, user_id, name, weekly_hours[], time_zone
DateOverride          id, schedule_id, date, hours[] (empty = off)
             evidence: S11, community posts on date overrides
             confidence: high

EventType    id, owner_id, kind (one_on_one|group|collective|round_robin), name, slug, color, duration_min, location,
             availability_schedule_id, buffer_before, buffer_after, min_notice, date_range, start_increment,
             limits (per_day|per_week|per_month), free_busy_exceptions[], confirmation_text, redirect_url?, active, secret?
             evidence: S04-S06, help "customize event types", "meeting limits"
             confidence: high (secret/redirect unverified)

EventTypeHost  event_type_id, user_id, priority          (collective / round robin)
             confidence: medium

InviteeQuestion  id, event_type_id, label, type, required, order
             evidence: S05
             confidence: high

Booking      id, event_type_id, host_id, start_at, end_at, status (confirmed|canceled|rescheduled|no_show),
             location_details, cancel_reason, created_at
             evidence: S10, API scheduled events
             confidence: high

Invitee      id, booking_id, name, email, answers (json), time_zone, is_guest, no_show
             evidence: S33, API invitees ("mark as no-show")
             confidence: high

SingleUseLink   id, event_type_id?, status (created|viewed|booked), reserved_slots[], expires_at
             evidence: S07, help "one-off meeting"
             confidence: medium

MeetingPoll  id, owner_id, options[], votes[], status
             evidence: S09, help
             confidence: medium

Workflow     id, owner_id, name, trigger (before_event|after_event|on_booked|on_canceled), offset, channel (email|sms),
             template, event_type_ids[]
             evidence: S13, learn/calendly-workflows
             confidence: medium

RoutingForm  id, org_id, questions[], routes[] (conditions -> event_type | url | message)
             evidence: S15, S39, blog
             confidence: medium

Payment      id, booking_id, provider (stripe|paypal), amount, status
             confidence: guess

WebhookSubscription  id, scope (org|user), events[], url, signing_key
             evidence: API summaries
             confidence: high

Contact      id, owner_id, email, name
             evidence: Chrome extension "Contacts" tab
             confidence: medium

ActivityLogEntry  id, org_id, actor_id, action, at
             confidence: medium
```

Relationships: Organization 1-n Group, Group n-n User, User 1-n AvailabilitySchedule, User 1-n CalendarConnection, User 1-n EventType, EventType n-n User (hosts), EventType 1-n InviteeQuestion, EventType 1-n Booking, Booking 1-n Invitee, EventType n-n Workflow, RoutingForm routes to EventType.

## Feature matrix

See `features.csv`. Must: 17, should: 19, could: 36 (72 rows). Ten of the could rows are marked `skip` in the clone column: they are never built, with a reason in notes.

## Out of scope (cannot or should not be cloned)

- Calendly's brand, logo, copy, email templates, help content. Never copy.
- Its user base and marketplace/integration directory listings (network effect).
- Native Salesforce/HubSpot ownership routing, SAML SSO, SCIM: need paid third-party tenants and add nothing for own use.
- AI Notetaker and Callie: paid add-on products with recording, transcription and LLM work. Use an existing notetaker tool instead.
- Mobile apps: excluded by scope.
- Integrations must use official APIs only (Google Calendar API, Microsoft Graph, Zoom, Stripe). No private endpoints.

## Size

Screens 22 host + 11 guest (33, plus ~7 email templates), flows 12, entities 18. Hard parts: two-way calendar sync with correct busy/free and time zone and DST math, slot generation (limits, buffers, notice, overrides, round-robin fairness), concurrency on double booking, email/SMS delivery with reminders scheduled in a job queue, Stripe payments, OAuth to Google and Microsoft with token refresh, embeds. Size: XL for "everything". Rescoped to the core loop (all the must rows plus S01-S06, S10-S12, S30-S36, confirmation/reminder emails) it is M. Recommended path: build M first, add should rows, treat could rows as a backlog since this is own use.

## Open questions for the user

1. Is it one host or a team? One host drops all of Admin center (S18-S21), groups and round-robin.
2. Which calendar and video provider do you use? That decides which integration gets built first.
3. Do you need payments or SMS?
