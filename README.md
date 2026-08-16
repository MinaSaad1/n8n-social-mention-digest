# n8n Social Mention Digest

![n8n](https://img.shields.io/badge/n8n-template-EA4B71?logo=n8n) ![Schedule](https://img.shields.io/badge/Trigger-Schedule-555) ![Claude](https://img.shields.io/badge/Claude-Sonnet_4.6-D97757) ![Reddit](https://img.shields.io/badge/Reddit-source-FF4500?logo=reddit&logoColor=white) ![License](https://img.shields.io/badge/License-MIT-yellow.svg)

> A warm mention you miss for 72 hours is a lead lost. This workflow runs every morning, scans Google web, news, and Reddit for your brand and product names, asks Claude to score sentiment and relevance, and ships a single digest to Slack or email with negatives at the top.

> Part of the **[n8n-ai-agents catalog](https://github.com/MinaSaad1/n8n-ai-agents)**: see the catalog for shared [architecture principles](https://github.com/MinaSaad1/n8n-ai-agents/blob/main/docs/architecture-principles.md), [security framework](https://github.com/MinaSaad1/n8n-ai-agents/blob/main/docs/security-framework.md), and [output conventions](https://github.com/MinaSaad1/n8n-ai-agents/blob/main/docs/output-conventions.md) every template in the collection follows.

![Cover](screenshots/cover.png)

---

## What it does

- Fires daily at 07:00 (configurable cron) so the digest is waiting when you open your laptop
- Hits SerpAPI three times in parallel: general web search, Reddit-scoped search, Google News, all filtered to the past 24 hours
- Compiles every hit into a single stream, then asks Claude to classify each one as positive, neutral, negative, mixed, or irrelevant, plus a relevance score
- Drops anything irrelevant or low-relevance, groups the rest by sentiment with negatives first, and emails the digest

## Architecture

```
Daily at 07:00 (cron)
        │
        ▼
Config: brand name, keywords, SerpAPI key
        │
        ├──── SerpAPI Web (past 24h)
        ├──── SerpAPI Reddit (site:reddit.com)
        └──── SerpAPI News
        │
        ▼
Compile All Results (Code)
        │
        ▼
Claude classifies each item → sentiment + relevance + summary
        │
        ▼
Parse JSON → Filter out irrelevant/low → Aggregate
        │
        ▼
Build Digest (negative-first sections)
        │
        ▼
Email or Slack delivery
```

Twelve nodes plus a sticky README. The classification step is the only AI call in the pipeline, runs on Claude Haiku to keep cost in the cents-per-month range.

See [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) for the full component breakdown.

## Requirements

- **n8n** ≥ 1.78 (cloud or self-hosted)
- **SerpAPI** account, free tier gives 100 searches/month, enough for 33 days of the default three queries per run
- **Anthropic API key** for Claude Haiku classification (cost: under 1 USD/month for daily runs)
- **A delivery destination**: Gmail/SMTP for email, or a Slack node wired in if you prefer Slack
- **Optional**: Apify or RapidAPI Twitter scraper if you want X coverage. The official X API is now 100 USD/month for the basic tier, so most users skip it. See [`docs/SETUP.md`](docs/SETUP.md) for the trade-off matrix.

## Quickstart

### 1. Clone

```bash
git clone https://github.com/MinaSaad1/n8n-social-mention-digest.git
cd n8n-social-mention-digest
```

### 2. Get a SerpAPI key

Sign up at [serpapi.com](https://serpapi.com), confirm your email, copy the API key from the dashboard. The free tier allows 100 searches/month. The default workflow uses three searches per run, so daily runs land at ~90 searches/month.

### 3. Import the workflow into n8n

1. n8n → **Workflows** → **Import from File**
2. Select [`workflows/01-social-mention-digest.json`](workflows/01-social-mention-digest.json)
3. Open the imported workflow

### 4. Create credentials

| Node | Credential | Notes |
|---|---|---|
| `Claude - Classify Model` | Anthropic API | Get a key at console.anthropic.com. Haiku is the default model, cost is negligible. |
| `Email Digest` | Gmail OAuth2 | Or swap for Send Email (SMTP), Resend, or a Slack node. |

### 5. Configure your brand and keywords

Open the **Config - Search Settings** node and set:

- `brand_name`: your exact brand string, e.g. `"Acumen CRM"` or `"Mina Saad"`
- `keywords`: JSON array of brand and product names you want monitored
- `serp_api_key`: the SerpAPI key from step 2
- `digest_email`: where the digest lands

Start with 1 to 3 keywords. More keywords means more SerpAPI calls per run and faster free-tier burn. See [`docs/SETUP.md`](docs/SETUP.md) for false-positive handling on common-word brand names.

### 6. Confirm the schedule

The trigger fires at `0 7 * * *` (07:00 daily, n8n instance timezone). For most teams, weekday-only is enough: change the cron to `0 7 * * 1-5`. Set the n8n container's `TZ` env var if you need a specific timezone, see [`docs/SETUP.md`](docs/SETUP.md).

### 7. Test, then activate

Run **Execute Workflow** once. Check the Compile, Classify, and Build Digest nodes for sane outputs. Check the email arrives. Then flip the workflow to active.

## Configuration

- **Slack instead of email**: replace the `Email Digest` node with a Slack node. Use `Resource: Message`, `Operation: Send`, point at a private channel, and set the message body to `={{ $json.digest }}`. Plain text renders cleanly in Slack.
- **Add Twitter/X coverage**: insert a parallel HTTP Request node hitting an Apify Twitter scraper actor. Cheaper than the official API and the data shape merges into `Compile All Results` with one mapper line. See [`docs/SETUP.md`](docs/SETUP.md).
- **Add competitor monitoring**: extend the `keywords` array in the Config node with competitor names. Same pipeline, the digest now covers your share of voice too.
- **Reply drafting**: for negatives flagged as high relevance, add a second Claude chain that drafts a response. Send the draft to your inbox so you can review and send manually.
- **Weekly volume tracker**: log the `mention_count` and `negative_count` from `Build Digest Text` to a Google Sheet or Airtable. After 4 weeks you have trend data.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Digest is full of irrelevant hits | Brand name is a common word (e.g. "Hero", "Pulse") | Use exact-match quoting in the SerpAPI query, or add a negative keyword. See `docs/SETUP.md` false-positive section. |
| SerpAPI returns 429 or 401 | Free tier exhausted, or invalid key | Check serpapi.com dashboard. Drop to weekday-only schedule, or upgrade to the 50 USD/month tier (5000 searches). |
| Same mention appears multiple days | The workflow doesn't deduplicate against previous runs | Add an Airtable or Postgres log step at the end, then a "Lookup against log" filter at the start. The starter intentionally skips this so you can size dedup to your tools. |
| Claude returns malformed JSON | Rare model drift, especially on long snippets | The `Parse Classification` node already falls back to a neutral/low default. If it happens often, lower `maxTokens` and tighten the prompt. |
| Digest arrives but looks empty | Filter dropped everything as low relevance | Open the `Filter Relevant Only` node, temporarily disable the filter, see what's flowing in. Tune your brand string. |
| Cron fires at the wrong time | n8n's `TZ` env var is UTC | Set `TZ=Africa/Cairo` (or your zone) on the n8n instance, restart. See [`docs/SETUP.md`](docs/SETUP.md). |

## Security

Five things matter for this workflow:

1. **SerpAPI cost runaway**: a misconfigured cron or expanded keyword list can torch the free tier in a day. Cap with paid-tier alerts.
2. **Reddit terms of service**: public read endpoints are fine; do not hammer them or scrape user PII.
3. **X/Twitter API economics**: assume 100 USD/month minimum if you go official. Most teams skip X or use Apify scrapers.
4. **False-positive pollution**: common-word brands flood the digest with noise that the LLM sees and pays tokens for. Tighten keywords first.
5. **Credential rotation**: SerpAPI keys, Anthropic keys, Gmail OAuth tokens. Treat them like any other secret.

Full threat model and layered defenses in [`docs/SECURITY.md`](docs/SECURITY.md).

## Roadmap

- [ ] Sub-workflow for cross-day deduplication (Airtable-backed)
- [ ] Sentiment trend chart attached to the weekly digest
- [ ] Optional reply-draft chain for negative high-relevance mentions
- [ ] Apify X scraper recipe with the actor ID and field mapping documented

## License

MIT, see [LICENSE](LICENSE).

## Credits

Built by [Mina Saad](https://github.com/MinaSaad1). Part of the [n8n-ai-agents catalog](https://github.com/MinaSaad1/n8n-ai-agents).

---

## Need this running in your business?

This template is free and MIT, and it is meant to be forked. Getting one into
production against your real data, your credentials and your edge cases is a
different job, and it is the one I do.

I work out what is actually costing a business, then build whatever fixes it: an
AI agent, an automation, or a full application. Handed over so your team owns it.

[Book a call](https://cal.com/minasaad/60min) · [mina-saad.com](https://www.mina-saad.com)
