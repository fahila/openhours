# What Calendly's users hate, and the angle: not done yet

**Sample size: 0 reviews.** Nothing below this line has evidence behind it, so this file ranks nothing, adds nothing to `features.csv`, and recommends no positioning angle.

## Why

This sandbox's network policy denied every place the skill reads from:

| source | host | result |
| --- | --- | --- |
| App Store (official reviews feed) | `itunes.apple.com` | denied |
| Hacker News (official API) | `hn.algolia.com` | denied |
| Reddit | `www.reddit.com` | denied |
| G2, Capterra, Trustpilot | `www.g2.com`, `www.capterra.com`, `www.trustpilot.com` | denied |
| Google Play | `play.google.com` | denied |
| Calendly's own community board and site | `community.calendly.com`, `calendly.com` | denied |

The web search tool does work, but it returns summaries written by the tool, not the reviewers' words with a link to each review. Under this skill's rules every quote must be verbatim and linked, so those summaries were not used. Nothing was invented to fill the gap, and `reviews.csv` is an empty sheet (header row only). `reviews.py` was run on it and correctly refused: "no usable rows".

## Two ways to unblock it

1. **Allow the two official feeds**: in the environment's network settings add `itunes.apple.com` and `hn.algolia.com` (both are named in the skill as fine within their terms). That gives an App Store sample and a Hacker News sample directly.
2. **Paste the rest in.** G2, Capterra, Trustpilot, Google Play and Reddit are best read by a person in a browser and copied across (many of their terms forbid automated access). Copy rows into `replica/reviews.csv` as `source,url,date,rating,text`, text exactly as written, one row per review, the link to that review or thread on every row.

## What to collect (the bar)

100 or more reviews across at least three sources, newest first. Read the 3 and 4 star reviews as well as the 1 and 2 star ones: "love it, but..." is where the best fixes are. A theme backed by fewer than 3 reviews, or by one source, will be reported as thin and not presented as a trend.

Searches that tend to find the useful ones:

- App Store and Google Play: the Calendly listing, Ratings and Reviews, sorted by newest. The App Store feed is `https://itunes.apple.com/us/rss/customerreviews/id=<APP_ID>/sortBy=mostRecent/json` (the id is the number in the listing's address).
- G2, Capterra, Trustpilot: the product's review pages, filtered to 1 to 3 stars as well.
- Reddit: "calendly alternative", "switched from calendly", "calendly sucks", "calendly vs".
- Hacker News: the same queries, through the API.
- Calendly's own community board: threads asking for features and the vote counts.
- Its changelog (`community.calendly.com`, "change log" posts), so nothing already fixed gets "fixed" again.

## Then

```bash
python3 ~/.claude/skills/replica-entrepreneur/reviews.py replica/reviews.csv --out replica/feedback.md
```

and run `/replica-entrepreneur` again: it will write the three ranked lists (what they hate, what is missing, what is unsolved), the fix plan with new `features.csv` rows, and three positioning options with one recommended.

## What can be said without reviews

Only this, and it is not review evidence: the gaps between the clone and what Calendly's own documentation describes are in `parity.md`. That is a list of features to build, not a reason for anyone to switch.

Because there is no angle yet, `/replica-brand` can still pick a name, palette and voice, but it has nothing to hang a positioning on. Waiting for this step is better.
