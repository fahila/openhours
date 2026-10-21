# Brand: Openhour (working name)

Status: **working name, not cleared.** Every availability check below is "to run": this sandbox cannot reach trademark offices, registrars, app stores or social sites, so nothing here says "available". Before money is spent on a domain, logo or listing, run the checks and have a trademark lawyer do a proper search.

## Angle

`replica/fixes.md` has no angle: the entrepreneur step found 0 reviews, so there is no evidence-based positioning. The name therefore only has to stay out of the original's way and fit what the product does (self-hosted scheduling for one owner's own business). If a real angle turns up later, revisit the shortlist.

## Name

Twenty candidates, by style:

- descriptive: Booklink, Slotwise, Openhour, Freebusy, Timeslot
- compound: Slotmark, Holdopen, Meetlane, Calmeet, Opendesk
- invented: Calvo, Tavelo, Zenda, Mitra, Orvia
- metaphor: Quayside, Tavolo, Lantern, Harbor, Ferry

Cut to five. Cuts: anything ending "-ly" or echoing the original's sound or look (Calmeet, Calvo); crowded or generic words (Harbor, Lantern, Booklink, Timeslot, Freebusy: also a Google API term); awkward in a big language or hard to spell after hearing it once (Orvia, Zenda, Tavelo, Mitra, Ferry, Meetlane, Opendesk).

| # | name | style | why it stays | main risk |
| --- | --- | --- | --- | --- |
| 1 | **Openhour** (recommended working name) | descriptive | says what the product offers: hours when you are open to be booked; easy to spell and say | plain English words, so the trademark is weak and many apps may use "open hour(s)" |
| 2 | Slotmark | compound | "mark a slot" | "slot" has gambling overtones in some contexts |
| 3 | Holdopen | compound | a held time slot; matches the 35-minute payment hold | clumsy to say |
| 4 | Tavolo | metaphor | Italian "table", where meetings happen | already a common brand name in food and hospitality |
| 5 | Quayside | metaphor | a place where people meet | long; crowded with real-estate names |

Recommendation: Openhour, because it is the one that describes the product. The weak-trademark risk is real: a descriptive name is easy to find but hard to protect, so decide whether protection matters before committing.

The skill's generic words ("Event types", "Workflows", "Availability", "Integrations") were kept on purpose. They name common scheduling concepts, are not distinctive to the original, and renaming them would hurt switchers. Flagged for the owner's decision, not changed.

### Checks to run (all "to run", none run)

| check | where | Openhour | Slotmark | Holdopen | Tavolo | Quayside |
| --- | --- | --- | --- | --- | --- | --- |
| US trademark (classes 9, 42) | tmsearch.uspto.gov | to run | to run | to run | to run | to run |
| EU trademark | euipo.europa.eu eSearch / TMview | to run | to run | to run | to run | to run |
| Canada | ised-isde.canada.ca | to run | to run | to run | to run | to run |
| global | WIPO Global Brand Database | to run | to run | to run | to run | to run |
| domain (.com, .app, .io) | registrar search / `whois` | to run | to run | to run | to run | to run |
| App Store, Google Play | search the exact name | to run | to run | to run | to run | to run |
| handles | X, Instagram, TikTok, GitHub | to run | to run | to run | to run | to run |
| the web | search "name + scheduling" | to run | to run | to run | to run | to run |

These are screening checks, not legal clearance.

## Palette

Roles are unchanged (`design/tokens.json`); only values changed.

- Brand hue: teal `#0f766e` (hover `#0b5e58`, tint `#e6f4f2`). The original's brand blue is around hue 215 to 220; this is around hue 175, a different family, not a nearby blue.
- Contrast: `python3 ~/.claude/skills/replica-design/contrast.py replica/design/tokens.json` reports 27 pairs, 0 failing AA.
- The sweep checks that `#006bff`, `#0069ff` and `#0b3558` do not appear anywhere in the app.
- The 8 event colours include one blue (`#2f5bea`). It is a generic UI blue and not the original's value, but it is the nearest thing to it. Swap it for another hue if that is too close for a lawyer's comfort.

## Logo brief

For whoever makes it (the owner, a designer or an image model). Not a logo.

- Idea in one line: an open door of time, a clock face with a gap in it, for "open hour".
- Mark type: symbol plus wordmark; the symbol must also work alone as the app icon.
- Must work at 16px (favicon) and at 1024px (app icon). Test both before accepting.
- Deliverables: SVG; app icon 1024x1024 with no transparency (iOS); favicon set; social image 1200x630.
- Must not resemble the original's mark: no shared shape, colour pair or letterform trick. Put the original's logo next to every draft and check by eye. This could not be done here (no access to the original's assets), so it is on the owner.
- Palette: the teal above on a white or very dark ground. No gradients that echo the original's.
- Not done: no logo exists yet. The app currently shows the name as plain text and has no custom favicon or OG image.

## Voice

Three words, each with what it does not mean:

1. **Plain**, not curt. Say the thing in short words; keep the courtesy.
2. **Calm**, not sleepy. Errors and cancellations never shout or panic; nothing is "Oops!".
3. **Exact**, not technical. Give the real time, date and next step; leave out jargon and codes.

Do and don't:

| do | don't |
| --- | --- |
| "You're booked." | "Awesome! Your meeting has been successfully scheduled!" |
| "We couldn't load the times. Try again." | "Oops! Something went wrong." |
| "No event types yet." | "Nothing to see here." |
| "Too many attempts. Wait a minute." | "Rate limit exceeded (429)." |
| "Cancel this meeting?" | "Are you sure you want to proceed with cancellation?" |

### Audit of the most-seen strings

Ten strings that users hit most, read against the voice. The app was written fresh from the start (no original copy was used), so no string needed rewriting. None was changed, to avoid churn in 182 e2e tests for no gain.

| string | where | verdict |
| --- | --- | --- |
| New event type | dashboard button | keep: plain |
| You're booked | confirmation | keep: plain, calm |
| Pick another time | booking page, after a conflict | keep |
| Cancel this meeting? | cancel dialog | keep |
| Meeting cancelled | cancellation result | keep |
| No event types yet | empty state | keep |
| We couldn't load the times. | slots error | keep: calm, but add a retry hint if tests allow later |
| We couldn't reach the server. Check your connection and try again. | network error | keep: exact next step |
| Too many attempts. Wait a minute. | rate limit | keep |
| Your details are used only to arrange this meeting. | booking form note | keep: plain |

One genuine inconsistency: "Too many attempts. Wait a minute." and "Too many attempts. Wait a minute and try again." both exist. Harmless; unify when someone next touches those files.

## Where the name lives in code

`app/lib/brand.ts` is the single place: `BRAND.name`, `BRAND.slug`, `BRAND.icsDomain`. Used for page titles, the host layout, the login page, ICS PRODID and UID domain. `package.json` name, `.env.example` sender example and the token file's description also carry it. Renaming the product is a one-file change, plus those three static spots.

## Sweep

`python3 ~/.claude/skills/replica-brand/sweep.py app --config replica/brand.json` : clean. It only searches for the names, domain and colours listed in `brand.json`. It cannot tell whether a layout or phrase echoes the original, so the by-eye check of favicon, titles, email templates and the OG image is still the owner's to do.

## Open items for the owner

1. Run the checks above (or hand them to a trademark lawyer) before committing to Openhour.
2. Decide about the generic scheduling terms and the one blue event colour.
3. Commission or make the logo against the brief.
4. Replace the placeholder sender domain in `.env.example` with the real one.
