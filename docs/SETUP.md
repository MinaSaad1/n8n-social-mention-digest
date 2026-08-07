# Setup notes

Specifics that don't fit cleanly in the README's Quickstart. Read this before activating in production.

## Mention source comparison

There are five practical ways to find brand mentions on the public internet. Each has a cost profile and a coverage profile. Pick based on where your audience actually talks.

| Source | Cost | Coverage | Notes |
|---|---|---|---|
| **SerpAPI (Google Web + News)** | 100 free/month, then 50 USD/month for 5000 | Catches blog posts, forums, press coverage, syndicated mentions | The default in this workflow. Reliable, ToS-clean, fast. |
| **Google Custom Search Engine** | 100 free/day (3000/month) | Same Google index as SerpAPI but lower-quality result formatting | Free fallback if you exceed SerpAPI's free tier. Setup is more fiddly. |
| **Reddit API (official)** | Free, OAuth-required for write, public read works without auth | Reddit posts and comments, real-time | Free tier rate limits at 60 req/min. Plenty for daily cadence. |
| **Apify Twitter scraper** | ~5 to 10 USD/month for daily cadence | X/Twitter posts | Cheaper than the official X API. Lives in a ToS grey zone. |
| **Official X/Twitter API (basic tier)** | 100 USD/month for 10,000 reads | X/Twitter posts | Honest pricing note: this is now expensive. Skip unless X is your primary audience. |
| **LinkedIn** | No public API for mentions | None reliable | Sales Navigator has lookup but no public-mention search. Most users skip. |

### Recommended starter stack

For 80% of B2B SaaS and personal-brand use cases:

- **SerpAPI** for Google web + news (the default in this workflow)
- **SerpAPI's site-restricted Reddit search** (also default): uses Google's index of Reddit, which is ToS-clean and rate-limit-free
- **Skip X entirely** until you have a reason not to

Coverage of this stack: every blog post, news article, and Reddit thread that Google has indexed in the past 24 hours. For the price (3 USD/day at most, free if you stay under 100 searches/month), nothing else comes close.

If you specifically need real-time Twitter coverage and your audience is there, add an Apify Twitter scraper actor as a fourth parallel branch in the workflow. The official X API at 100 USD/month is hard to justify unless you have a contractual or reputational reason for clean ToS compliance.

## Brand keyword setup

The single biggest lever for digest quality is your brand string. Spend ten minutes here, save hours of noise later.

### Exact match vs fuzzy

Wrap the brand in quotes if it's a multi-word name or a common single word:

- `"Acumen CRM"` matches the exact phrase, drops mentions of the unrelated word "acumen"
- `"Mina Saad"` (multi-word person names) matches the full name, drops mentions of unrelated "Mina" or "Saad"
- `Pulsar` (single word, unique) needs no quoting

### False positive examples

| Brand | Problem | Fix |
|---|---|---|
| **Hero** (baby food) | Returns comics, Marvel, action movies, video games | `"Hero baby" OR "Hero MEA"` plus negatives `-comic -marvel -movie` |
| **Pulse** (a CRM, hypothetical) | Returns medical articles, fitness trackers, music news | `"Pulse CRM"` exact match required |
| **Apex** | Returns Apex Legends, Apex programming, anything | Switch the search term to the product name, not the company |
| **Mina** (person name in Arabic-speaking world) | Hugely common name, port name in Saudi Arabia | `"Mina Saad"` full name, plus a context-aware LLM prompt |
| **Acme** | Returns Looney Tunes, every other Acme | `"Acme <product>"` and accept that pure-brand searches won't work |

### Negative keywords in SerpAPI

SerpAPI passes Google search syntax through directly. Use minus-prefixed terms to exclude:

```
"Hero MEA" -comic -movie -game
"Pulse" -fitness -music
```

Test in the SerpAPI playground (serpapi.com/playground) before wiring into the workflow. You can iterate on the keyword in 30 seconds there, vs minutes per round in n8n.

### Number of keywords

Start with 1 to 3. Each keyword is a separate SerpAPI call (well, 3 calls per keyword if you keep web + Reddit + News parallel). The free tier dies fast if you go to 5+.

For multiple keywords, the simplest pattern is to extend the `keywords` array in Config and add a Split Out + Loop wrapper around the three search nodes. Or just clone the workflow per brand if you only have 2 to 3.

## Sentiment classification prompt

The default prompt in the **Classify Mention** node:

```
Classify this search result about a brand.

Title: {{ $json.title }}
Snippet: {{ $json.snippet }}
Source: {{ $json.source }}
Type: {{ $json.type }}

Return JSON with:
- sentiment: "positive" | "neutral" | "negative" | "mixed" | "irrelevant"
- relevance: "high" | "medium" | "low" (is this actually about the brand?)
- summary: max 15 words

Return only valid JSON. No markdown fencing.
```

This works as a baseline. To customize:

- **Add brand context**: prepend "The brand is `<X>`, which sells `<Y>` to `<Z>`." This dramatically improves relevance scoring on common-word brands.
- **Tune the sentiment scale**: if your team responds differently to "complaint" vs "feature request", split negative into those two buckets in the schema and update the digest builder to render both.
- **Lower temperature**: Claude Haiku already runs near-deterministic on classification tasks, but if you see drift, the LLM node accepts a `temperature` option in the model parameters.

Token usage on the default prompt is ~200 input + 50 output per item. At Haiku pricing (~0.25 USD per million input tokens, ~1.25 USD per million output tokens at the time of writing), 600 classifications/month lands at well under 1 USD.

## Digest schedule

### When to fire

**Morning is best.** 06:00 to 08:00 local time, depending on when your team starts. The whole point is to have the digest waiting on your phone or laptop when you sit down. Anything later misses the early-morning response window for warm leads.

**Weekday-only is enough for most teams.** Weekend mentions roll into Monday's digest naturally; very few B2B brands have weekend-urgent mentions. Change the cron from `0 7 * * *` to `0 7 * * 1-5` and reclaim two SerpAPI calls per week.

**Avoid hourly.** Hourly cadence triples your SerpAPI cost, doubles your inbox volume, and the marginal gain in response time over daily-at-07:00 is small. The exception is brands actively in a PR situation; in that case run hourly for the duration of the situation, then scale back.

### Timezone configuration

The Schedule Trigger fires in n8n's instance timezone, not the workflow's. If your container is UTC and your cron says `0 7 * * *`, you get 07:00 UTC, not 07:00 local.

To fix this for the whole n8n instance:

```bash
# In your docker-compose.yml or wherever you run n8n
environment:
  - TZ=Africa/Cairo            # or America/New_York, Europe/London, etc.
  - GENERIC_TIMEZONE=Africa/Cairo
```

Restart n8n. Confirm the schedule's "next run" time in the UI now matches your wall clock.

If you can't change the instance timezone (shared n8n, multi-team), express the cron in UTC offset. For Cairo (UTC+3, no DST since 2014), `0 7 * * *` local is `0 4 * * *` UTC.

## Slack vs email format trade-offs

The default delivery is Gmail. Most teams are better served by Slack. Both are 5-minute swaps; pick based on how your team actually reads.

### Slack

**Pros**:
- Threaded responses if multiple people on the team triage mentions
- Fast skim, especially on mobile
- Easy to react with emoji to mark "I'll handle this"

**Cons**:
- Plain-text formatting only renders ok in Slack; rich digest layouts (color, bold per section) require Block Kit JSON which is more work
- Slack channel notifications batter your focus if you're not careful. Use a dedicated channel and mute it except for digest hour

**How to swap**: delete the `Email Digest` node, add a Slack node, set:
- Resource: Message
- Operation: Send
- Channel: your private channel (start with a test channel)
- Text: `={{ $json.digest }}`

For richer formatting, use Block Kit. The digest builder already produces section-grouped output that maps cleanly to Block Kit sections.

### Email

**Pros**:
- Full HTML rendering if you build the digest as HTML instead of plain text
- Searchable archive in Gmail or whatever inbox you use
- Doesn't compete with workplace chatter

**Cons**:
- Inbox fatigue is real; if you already have 200 emails/day the digest gets lost
- Slower to triage than a Slack channel where the whole team can pile in

**How to swap to HTML**: edit `Build Digest Text` to also emit an `html` field with proper `<h2>`, `<ul>`, `<li>` markup. Then in the Email Digest node, use the HTML body field instead of plain text.

### Both

Some teams want the digest in Slack for triage and in email for archive. Add both nodes after `Build Digest Text` and wire them in parallel. The cost is one extra Slack send per day, the benefit is searchable history without reverse-engineering Slack's archive.

## Adding the Anthropic credential

In n8n: **Credentials → New → Anthropic API**. Paste the key from console.anthropic.com. Bind to the `Claude - Classify Model` node.

If you prefer OpenRouter (one credential for multiple models, slightly cheaper at small scale), swap the LM Chat Anthropic node for an HTTP Request node hitting `https://openrouter.ai/api/v1/chat/completions`. Same prompt, same JSON parsing downstream.

## Optional: cross-day deduplication

The starter workflow doesn't deduplicate against previous days. Most users don't need it because Google's `qdr:d` filter is naturally fresh. If you do see repeats:

1. Add an Airtable or Postgres credential
2. After `Build Digest Text`, add a step that writes each mention's `link` (URL is the most reliable dedup key) plus today's date into a `mentions_log` table
3. Before `Filter Relevant Only`, add a lookup against the same table; drop any item whose link appeared in the past 7 days

The starter intentionally skips this so you size the dedup store to your stack instead of forcing Airtable on Postgres-only users.
