# Bugs

Found by the tests in this pass (`bugs` 001 to 016) and in earlier passes (017 to 022). Every one was reproduced, got a failing test first where one could be written, was fixed, and the test was kept. **Open: none. No open S1 or S2.**

Severity: S1 data loss, security hole, wrong payments or a blocked core flow. S2 a feature broken with no workaround. S3 broken with a workaround, or visibly wrong. S4 cosmetic.

| severity | open | fixed |
| --- | --- | --- |
| S1 | 0 | 1 |
| S2 | 0 | 3 |
| S3 | 0 | 14 |
| S4 | 0 | 4 |

Browser for every bug below: Chromium 1280px (Playwright), local server, fresh database, mocked Google, Resend and Stripe. Build is the commit before the fix.

### BUG-001: Google access revoked by the person is not shown to the host; guests just fail

- Severity: S3
- Flow / case: F01 / F01-E9
- Screen: S14, S31
- Build: b7853ec  Browser: Chromium, 1280px

Steps
1. Connect Google Calendar.
2. In Google, remove the app's access (the access token and the refresh token both stop working).
3. Open the public booking page, then the host's Integrations page.

Expected: The booking page can't offer times (503). The Integrations page says "Needs reconnecting" straight away, and the connection is marked so.
Actual: The booking page returned 503 but Integrations still showed "Connected" for up to an hour, until the cached access token would have expired.
Evidence: `F01-E9` failed; regression unit test in `tests/google.test.ts` ("a 401 on the cached access token…")
Suspected cause: A 401 from Google never made us drop the cached token or try a refresh, so `invalid_grant` (which flips the status) was only seen once the token expired.
Status: fixed in 8957c4a

### BUG-002: A name made only of emoji, or a second event with the same name, cannot get a link name

- Severity: S3
- Flow / case: F02 / F02-E2, F02-E3
- Screen: S04
- Build: b7853ec  Browser: Chromium, 1280px

Steps
1. New event type, name `☕☕☕`, leave Link name empty, Create.
2. Or create "Intro call" twice.

Expected: The event is created with a generated link name (`event`, `intro-call-2`).
Actual: Stays on the form with "Use lowercase letters, numbers and dashes" or "You already use that link name", about a field the host never touched.
Evidence: `F02-E2`, `F02-E3` failed; `tests/security.test.ts` (link names)
Suspected cause: Slug was built from the name only; nothing handled an empty result or a clash when the host left the field empty.
Status: fixed in 263724f

### BUG-003: After one failed save, a group event is silently saved as one-on-one

- Severity: S2
- Flow / case: F02 / F02-E7
- Screen: S04
- Build: b7853ec  Browser: Chromium, 1280px

Steps
1. New event type, choose "Several guests share a time", enter 1 place, Create. The error "A group needs between 2 and 500 places." shows.
2. Change places to 8, Create.

Expected: A group event for 8 guests is saved.
Actual: A one-on-one event with 1 place is saved while the page still showed the group fields.
Evidence: `F02-E7` failed (database row was `one_on_one`, 1)
Suspected cause: React 19 resets the form's DOM after a form action, which snapped the controlled "who can book" select back to its first option while the page state still said group. The forms now submit through `onSubmit`.
Status: fixed in 263724f

### BUG-004: The note a guest writes for the host is not visible anywhere in the app

- Severity: S3
- Flow / case: F03 / F03-H2
- Screen: S10
- Build: b7853ec  Browser: Chromium, 1280px

Steps
1. As a guest, book a time and write a note.
2. As the host, open Meetings.

Expected: The note is shown with the meeting.
Actual: Only in the email and the calendar event. The Meetings screen showed no note.
Evidence: `F03-H2` failed
Suspected cause: The list never rendered `answers.notes`.
Status: fixed in af6a37e

### BUG-005: An impossible date such as 2026-13-45 causes a server error

- Severity: S3
- Flow / case: F03 / F03-N2, F05-N1
- Screen: S31, S11
- Build: b7853ec  Browser: Chromium, 1280px

Steps
1. GET /api/public/sam/30-minute-meeting/slots?from=2026-13-45&to=2026-13-46
2. Or add a date change of 2026-02-30 on Availability (forged request).

Expected: 400 with a clear error.
Actual: 500. The text matched the date pattern, so it reached luxon and Postgres.
Evidence: `F03-N2` failed; `tests/security.test.ts` ("dates must exist on the calendar")
Suspected cause: Dates were checked by shape only.
Status: fixed in 263724f

### BUG-006: After "That time was just taken" the guest's chosen day is forgotten

- Severity: S4
- Flow / case: F03 / F03-E1
- Screen: S31, S32
- Build: b7853ec  Browser: Chromium, 1280px

Steps
1. Pick a day and time, fill in the details.
2. Have someone else book that time, then press Book.

Expected: The message appears and the list of times for the same day is shown again.
Actual: The message appears but the calendar shows no selected day and no times, so the guest has to find their day again.
Evidence: `F03-E1` failed
Suspected cause: The effect that clears a day with no times ran while the times were reloading and the set of open days was briefly empty.
Status: fixed in af6a37e

### BUG-007: A guest who leaves a group meeting lands on "We can't find that page"

- Severity: S3
- Flow / case: F04 / F04-E7
- Screen: S36
- Build: b7853ec  Browser: Chromium, 1280px

Steps
1. Two guests book the same group time.
2. One of them cancels from their link.

Expected: A confirmation that they have left, the meeting goes ahead for the others.
Actual: "We can't find that page", because their own link no longer exists.
Evidence: `F04-E7` (updated to expect "You've left this meeting")
Suspected cause: The client navigated back to the manage link of a guest who had just been removed. There is now a `/left` page.
Status: fixed in af6a37e

### BUG-008: A moved meeting makes a second event in the guest's calendar app

- Severity: S3
- Flow / case: F08 / F08-E3, F08-E4
- Screen: emails
- Build: b7853ec  Browser: Chromium, 1280px

Steps
1. Book a meeting, open the invite attached to the confirmation.
2. Move the meeting and open the invite in the "Moved" email.
3. Cancel and open the cancel notice.

Expected: All three invites share one UID and the sequence rises, so the calendar app updates one event and removes it.
Actual: Each booking row had its own UID, so the moved invite had a different UID: the old event stays and a second one is added, and the cancel notice targets the wrong one.
Evidence: `F08-E3`, `F08-E4` failed; `tests/email-jobs.test.ts` ("a meeting keeps one calendar identity…")
Suspected cause: UID was the booking id, and a move is a new booking. It is now the first booking in the chain, with SEQUENCE equal to the number of moves.
Status: fixed in af6a37e

### BUG-009: A NUL character in a guest's name is a server error; line breaks in names reach email subjects

- Severity: S3
- Flow / case: F08 / F08-E10
- Screen: S33
- Build: b7853ec  Browser: Chromium, 1280px

Steps
1. POST the booking API with name `Evil\r\nBcc: attacker@example.com\u0000`.

Expected: The booking is made with the name cleaned to one line, or refused with a message.
Actual: 500 (Postgres rejects NUL). Without the NUL, the line break went into the host's email subject.
Evidence: `F08-E10` failed; `tests/security.test.ts` ("NUL bytes in guest text…")
Suspected cause: No cleaning of control characters. Names are now single-line, free text may keep line breaks but not control characters.
Status: fixed in 263724f

### BUG-010: The host cannot see whether a booking was paid

- Severity: S3
- Flow / case: F11 / F11-H1
- Screen: S10
- Build: b7853ec  Browser: Chromium, 1280px

Steps
1. Guest books a paid event and pays.
2. Host opens Meetings.

Expected: "Paid $25.00" next to the meeting, "Waiting for payment" while unpaid, "Refunded" after a refund.
Actual: Nothing about payment on the screen; the host had to open the Stripe dashboard.
Evidence: `F11-H1` failed
Suspected cause: Payment status was stored but never shown.
Status: fixed in af6a37e

### BUG-011: Backing out of Stripe's page keeps the time held for 35 minutes, with no message

- Severity: S3
- Flow / case: F11 / F11-N1
- Screen: S33
- Build: b7853ec  Browser: Chromium, 1280px

Steps
1. Choose a paid time, continue to payment, then use the back link on the payment page.

Expected: Back on the booking page: "Payment cancelled. You haven't been charged, and the time is free again." The time can be booked at once.
Actual: Back on the booking page with no message. The time stayed held, so choosing it again said "just taken" (it was the guest's own hold).
Evidence: `F11-N1` failed
Suspected cause: The cancel address carried no way to identify the hold. It now carries the guest's token, and the page releases an unpaid hold (and closes the Stripe session) when they return.
Status: fixed in af6a37e

### BUG-012: When Stripe is down the guest gets a vague error, not "nothing was charged"

- Severity: S3
- Flow / case: F11 / F11-E3
- Screen: S33
- Build: b7853ec  Browser: Chromium, 1280px

Steps
1. Make Stripe's checkout creation fail, choose a paid time and continue.

Expected: "We couldn't start the payment. Nothing was charged." and the form is kept.
Actual: "Something went wrong on our side." The server's explanatory message was thrown away for 502.
Evidence: `F11-E3` failed
Suspected cause: The booking page only showed the server's message for a few statuses.
Status: fixed in af6a37e

### BUG-013: Pages with private links and the host app are not marked noindex

- Severity: S4
- Flow / case: SEC / SEC-8
- Screen: S35, S36, S10…
- Build: b7853ec  Browser: Chromium, 1280px

Steps
1. Request /m/<token>, /booked/<token>, /reschedule/<token>, /login and any host page; look for X-Robots-Tag or a robots meta tag.

Expected: noindex, nofollow on all of them.
Actual: Nothing. A manage link that ended up on a public page could be indexed.
Evidence: `SEC-8` failed
Suspected cause: No robots header or metadata was set.
Status: fixed in 4285024

### BUG-014: A NUL byte in an address (/%00) makes a database error

- Severity: S3
- Flow / case: SEC / SEC-4
- Screen: S30, S31
- Build: b7853ec  Browser: Chromium, 1280px

Steps
1. GET /%00 (or /sam/%00).

Expected: 404.
Actual: Postgres "invalid byte sequence for encoding UTF8: 0x00" thrown from the profile page, which is a 500 in production.
Evidence: `SEC-4` failed; `tests/security.test.ts` ("link names from the address bar…")
Suspected cause: Link names from the URL were passed to the query unchecked. They are now shape-checked first.
Status: fixed in 263724f

### BUG-015: Empty states under a page title skip a heading level

- Severity: S4
- Flow / case: F13 / F13-A1
- Screen: S03, S10, S30…
- Build: b7853ec  Browser: Chromium, 1280px

Steps
1. Open any list screen with no items, run an accessibility scan.

Expected: No violations.
Actual: axe `heading-order`: an h3 straight after the h1.
Evidence: `F13-A1` failed
Suspected cause: The empty state always used an h3. It now takes a level (h2 under a page title).
Status: fixed in 4285024

### BUG-016: On the day clocks go back, a guest sees two identical time buttons

- Severity: S3
- Flow / case: F14 / F14-E1
- Screen: S32
- Build: b7853ec  Browser: Chromium, 1280px

Steps
1. Host works 00:00 to 03:00 on Sunday 1 Nov 2026 (New York). A guest in New York opens that day.

Expected: The two 1:00 AM and the two 1:30 AM are told apart (1:00 AM EDT, 1:00 AM EST).
Actual: Eight buttons, two pairs reading exactly the same, so the guest cannot tell which one they pick.
Evidence: `F14-E1` failed
Suspected cause: Times were formatted without a zone name. Only the repeated labels now carry one.
Status: fixed in af6a37e

## Found and fixed in earlier passes

| id | severity | what was wrong | caught by | fixed in | note |
| --- | --- | --- | --- | --- | --- |
| BUG-017 | S1 | Two guests could book the same time (double booking) | F03 / `tests/data.test.ts` race | bd79549 | An `await` between the slot check and the insert let three simultaneous requests all pass the check. A three-way race test caught it; now no `await` sits between check and insert, the host row is locked, and the database refuses overlaps. |
| BUG-018 | S2 | Deleting an account failed with a 500 when a paid booking existed | F10 / F10-E7 | 9923fea | Payment rows were protected from deletion. They are now removed first. The browser run caught it; the unit test had no payment rows. |
| BUG-019 | S3 | A failed booking cleared what the guest had typed | F03 / F03-E7, F03-E12 | d94e993 | React 19 resets a form after an action; the booking form now submits through `onSubmit`. |
| BUG-020 | S2 | `.env.local` was not git-ignored, so secrets could be committed | security checklist | b7853ec | Found with `git check-ignore` while ticking the checklist. |
| BUG-021 | S3 | Production started "healthy" with a missing variable and failed on the first request | `tests/env.test.ts`, manual start | 9923fea | Environment is now validated at boot (`instrumentation.ts`). |
| BUG-022 | S4 | Reminders stayed queued for meetings that were cancelled | F04 / browser run | 9923fea | Harmless (they were skipped when due) but untidy; cancelling now clears them. |

## To check (not reproduced as defects, or outside what could be tested here)

- **Everything that depends on a real service** (Google sign-in and calendar, Resend delivery, Stripe test cards including the declined card): `manual-checklist.md`. Nothing in it has been run.
- **Browsers other than Chromium** (Safari, Firefox) and real phones. Only Chromium at 1280px and 390px was used.
- **A real screen reader.** The suite checks names, labels, roles, focus and axe rules, not what is read aloud.
- **The calendar always starts the week on Sunday**, whatever the guest's locale. Someone in Germany would expect Monday. Cosmetic; not changed.
- **Weekly hours entry uses the browser's time picker**, whose look differs by browser and locale (12 or 24 hour). Worked in Chromium.
- **Link names changed in Settings or on an event** break the old public address (404) and the "Book a new time" link inside emails already sent. Manage links keep working. By design, worth a decision.
- **Performance with many bookings** (hundreds per host) was not measured. The booking transaction reads about 70 days of one host's bookings.
- **The in-memory busy-time cache is per server process**; with several instances each asks Google separately (still correct, slower).
- **The dev server's own overlay** (the round "N" badge) shows in test screenshots. It does not exist in a production build.
