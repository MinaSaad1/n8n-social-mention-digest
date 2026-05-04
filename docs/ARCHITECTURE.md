# Architecture

## High-level

```
Schedule Trigger (cron, daily 07:00)
        │
        ▼
Config (Set node) ─── brand_name, keywords[], serp_api_key, digest_email
        │
        ├──────► SerpAPI Web Search (qdr:d, num=10)
        ├──────► SerpAPI Reddit Search (site:reddit.com, qdr:d, num=5)
        └──────► SerpAPI News Search (tbm=nws, qdr:d, num=5)
        │
        ▼
Compile All Results (Code) ─── unifies the three response shapes into one stream
        │
        ▼
Classify Mention (Claude chain) ─── per item: sentiment, relevance, summary
        │
        ▼
Parse Classification (Code) ─── extracts JSON from LLM output
        │
        ▼
Filter Relevant Only ─── drop low-relevance and irrelevant
        │
        ▼
Aggregate into Digest ─── recombine items into one
        │
        ▼
Build Digest Text (Code) ─── grouped sections, negative first
        │
        ▼
Email Digest (Gmail) ─── or swap for Slack/Resend/SMTP
```

The pipeline is deliberately linear after the parallel search fan-out. Every step is debuggable in isolation in n8n's executions panel, and any of the three search legs can fail without taking the whole digest down.

## Components

### Daily at 07:00 (Schedule Trigger)

Cron `0 7 * * *` fires once per day in the n8n instance timezone. 07:00 is intentional: it's the latest morning slot where the digest is on your phone before you sit down at the laptop, and the earliest where Reddit has accumulated overnight UTC content from US-time-zone users.

For weekday-only delivery, change to `0 7 * * 1-5`. Most teams find weekend mentions can wait until Monday's digest; the workflow naturally rolls them in.

### Config - Search Settings (Set node)

A single Set node holds every operator-tunable value: `brand_name`, `keywords`, `serp_api_key`, `digest_email`. The keys are referenced downstream via `$('Config - Search Settings').item.json.<key>`, so reconfiguring the workflow is one node, not a node-by-node sweep.

The `keywords` field is an expression-typed array, not yet wired into the three search nodes (which all use `brand_name`). This is deliberate: most users start with one brand string before they expand. To monitor multiple terms, fan-out the search nodes via a Split Out + Loop pattern, see the README configuration section.

### Three parallel SerpAPI searches

| Node | Endpoint shape | Why this lane |
|---|---|---|
| **SerpAPI - Web Mentions** | `q=<brand>`, `tbs=qdr:d`, `num=10` | Catches blog posts, forums outside Reddit, generic web hits |
| **SerpAPI - Reddit Mentions** | `q=<brand> site:reddit.com`, `tbs=qdr:d`, `num=5` | Reddit's surface area is enormous and consistently high signal for SaaS/B2B brands |
| **SerpAPI - News Mentions** | `q=<brand>`, `tbm=nws`, `tbs=qdr:d`, `num=5` | Press, industry news, syndicated mentions |

`tbs=qdr:d` restricts results to the past 24 hours, which keeps the digest fresh and the LLM cost bounded. `num` caps each lane to a sane volume; the LLM then filters down further.

The three nodes run in parallel because they're independent HTTP calls with no shared state. n8n's executor handles fan-out automatically.

### Compile All Results (Code)

JavaScript node that pulls `organic_results` from the Web and Reddit responses, `news_results` from the News response, and normalizes all three into a single shape: `{ title, snippet, link, source, type }`. The `type` field carries forward into the LLM prompt and the final digest.

If all three lanes return empty (slow news day, or your brand string is too narrow), the node emits a single placeholder item so downstream classification still runs cleanly.

### Classify Mention (Claude chain)

The `chainLlm` node calls Claude Haiku with a structured prompt: title + snippet + source + type, asking for a JSON response with `sentiment`, `relevance`, and a 15-word `summary`. The prompt explicitly forbids markdown fencing.

Haiku is the right tier here. Each classification is a small, well-bounded judgment call; Sonnet is overkill, and Opus is dramatically overkill. At ~20 mentions per day, monthly cost lands under 1 USD.

### Parse Classification (Code)

LLMs occasionally fence their JSON output despite instructions. This node strips fences, attempts `JSON.parse`, and falls back to `{ sentiment: 'neutral', relevance: 'low', summary: '' }` on parse failure. The fallback ensures one bad classification doesn't crash the whole digest.

The parsed fields are merged onto the upstream item, so the downstream filter and digest builder see one object per mention with all metadata in one place.

### Filter Relevant Only

A simple two-condition filter: drop any item where `relevance === 'low'` OR `sentiment === 'irrelevant'`. The combinator is AND, so an item must pass both checks to survive.

This is where the LLM's relevance judgment pays off. SerpAPI doesn't know your brand context; Claude does, so it catches "different person with the same name" or "homonym product" cases that pure keyword matching can't.

### Aggregate into Digest

Standard n8n Aggregate node, `aggregateAllItemData` mode, output field `mentions`. Without this, every survived item would render as its own digest. Aggregate is what turns "N items" into "one digest with N mentions inside it."

### Build Digest Text (Code)

Buckets the mentions by sentiment, then renders sections in **negative-first** order: Negative, Mixed, Neutral, Positive. Negative-first is intentional: the things that need a response should be at the top of the email, not buried under the positives.

Output is a plain-text block plus three counters: `mention_count`, `negative_count`, `date`. The counters drive the email subject line so you can triage from the inbox preview without opening the message.

### Email Digest (Gmail)

Sends the digest to the configured `digest_email`. Subject line includes total mentions and negative count: `Mention Digest 2026-05-04 - 12 mentions, 2 negative`.

To swap for Slack: delete this node, drop in a Slack node, point it at a private channel, set `text={{ $json.digest }}`. Same data flow, different sink.

## Design decisions worth calling out

### Why three SerpAPI lanes and not one combined search

A single SerpAPI call returns ~10 organic results. Reddit and News are different result types in Google's index. Asking for them separately gives you predictable coverage of each surface; one combined call would overweight whichever surface ranks highest that day. The cost is the same (3 SerpAPI calls per run either way), the coverage is more even.

### Why Claude classifies one item at a time, not in batch

Batching would cut LLM cost roughly in half, but each item then shares a context window with the others, and the model occasionally lets sentiment from one mention bleed into another. Per-item classification keeps every judgment independent. At Haiku pricing the cost difference is rounding error.

### Why negatives go first in the digest

The whole point of the workflow is response time on warm leads and complaints. If positives are at the top, you read the good news, get a dopamine hit, close the email, and miss the complaint at the bottom. Negative-first keeps the digest aligned with what it's actually for.

### Why the workflow doesn't deduplicate across days

A proper dedup needs an external store (Airtable, Postgres, Redis) and adds two nodes plus a credential. For most users, daily-fresh results from SerpAPI's `qdr:d` filter naturally minimize repeats: Google's index doesn't reshuffle yesterday's hits into today's "past 24 hours" window often enough to be worth the build complexity. If you do see repeats, add the dedup; the README points to the right pattern.

### Why X/Twitter is not in the default pipeline

The official X API is now 100 USD/month for 10,000 reads. For a side project or a small team, that's an unjustifiable line item compared to the rest of the stack which costs cents. The `docs/SETUP.md` covers Apify-based alternatives that get most of the coverage at 5 to 10 USD/month.

## Performance notes

| Step | Latency expectation |
|---|---|
| Schedule fire | Instant |
| Config | <50 ms |
| 3x SerpAPI calls (parallel) | 800 ms to 2 sec |
| Compile All Results | <100 ms |
| Claude classify (per item, ~20 items) | 600 ms each, runs serially in the chain |
| Parse + Filter + Aggregate | <300 ms total |
| Build Digest Text | <100 ms |
| Email send | 500 ms to 1 sec |

End-to-end on a 20-mention day: ~15 to 20 seconds. The classification loop dominates; if you scale to 50+ mentions per run, consider batching the LLM calls.

## Observability

- **n8n Executions panel** is the primary debugging surface. Each scheduled run shows up there, click in to see exactly what the SerpAPI nodes returned and what Claude said about each item.
- The **sticky note inside the workflow** carries a live README. Edit it as you customize so future-you knows what's wired where.
- Consider adding a `Log to Sheet` step at the end that records `date`, `mention_count`, `negative_count`. After a few weeks you have your own trend data without having to compute it post-hoc.

## See also

- [SECURITY.md](SECURITY.md): threat model, layered defenses, what to lock down before production
- [SETUP.md](SETUP.md): source comparison, brand keyword setup, schedule patterns, Slack vs email format
- [Catalog architecture principles](https://github.com/MinaSaad1/n8n-ai-agents/blob/main/docs/architecture-principles.md): patterns shared across every template in the collection
