# Security & Hardening

## Threat model

What we assume:
- The n8n instance itself is reasonably hardened (auth on the UI, HTTPS, credentials stored encrypted at rest)
- The SerpAPI key, Anthropic API key, and Gmail credential live in n8n's credential store, not in the workflow JSON
- The digest destination (private inbox or private Slack channel) is access-controlled

What we don't protect against:
- A compromised n8n instance, once an attacker has admin on n8n, they own every workflow and credential
- Public-source data being public, every snippet the workflow reads from SerpAPI or Reddit is already indexed by Google
- Insider exfiltration via the digest output, anyone on the inbox or Slack channel can read what's in it

## Layered defenses (ordered by impact)

### Layer 1: SerpAPI cost runaway

**Problem**: SerpAPI's free tier is 100 searches/month. The default workflow uses 3 per run, which lands at ~90/month on daily cadence. Two things blow this budget: a misconfigured cron firing more than once per day, or expanding the keyword list and turning each run into 10+ searches.

**Fix**:
- Read the cron once before activating. `0 7 * * *` is daily. `0 * * * *` is hourly and will exhaust the free tier in 14 hours.
- If you expand keywords beyond 3 brands, upgrade to the paid tier (50 USD/month for 5000 searches) before activating.
- Set a budget alert in the SerpAPI dashboard. They have email alerts on usage thresholds.

**Caveat**: If you hit the limit mid-month, the workflow's HTTP nodes start returning 429 errors and the digest goes silent. Add a Slack alert on workflow failure so you notice within a day, not at month-end.

### Layer 2: Reddit terms of service

**Problem**: This workflow doesn't hit Reddit's API directly, it reads Reddit results from SerpAPI's Google index, which is fully ToS-clean. But operators sometimes "improve" the workflow by swapping in direct Reddit scraping, which is where ToS issues start.

**Fix**: If you do swap to direct Reddit access, use the official Reddit API (free, requires OAuth) with proper user-agent identification. Do not scrape `old.reddit.com` HTML, do not impersonate a browser, and do not pull user PII (usernames, emails harvested from comments) into your digest. Aggregated counts and post titles are fine; rolling up a target list of users to contact is not.

**Caveat**: Reddit's free API has rate limits (60 req/min OAuth, 10 req/min unauthenticated). At daily cadence this is irrelevant, but if you scale to hourly digests, plan for it.

### Layer 3: X/Twitter API economics (an honest note, not a defense)

**Problem**: This is not a security problem so much as a cost-honesty note. The official X API is now 100 USD/month minimum for the basic tier (10,000 reads/month). Cheaper Apify-based scrapers exist, and they work, but they sit in a grey zone: X's ToS prohibit scraping, Apify's actors do it anyway, and X has been litigious about this.

**Fix**: Either (a) pay for the official API and stay clean, (b) use Apify and accept the small risk that an actor breaks when X changes their HTML, or (c) skip X coverage entirely. Option (c) is what most users in this template's audience do.

**Caveat**: If your brand is heavily talked about on X specifically, the official API may be worth the 100 USD. For most B2B SaaS where Reddit and Google News are higher-signal, skipping X loses very little.

### Layer 4: Brand-mention false positives polluting the digest

**Problem**: Common-word brands (e.g. "Hero", "Pulse", "Acme") return huge volumes of irrelevant results. Every irrelevant result costs an LLM classification call, and noisy digests train you to skim instead of read.

**Fix**:
- Wrap the brand string in quotes: `"Acumen CRM"` not `Acumen CRM`. SerpAPI passes this through to Google as an exact-phrase search.
- Add negative keywords in the SerpAPI query string: `"Hero" -comics -movie -game` for a baby food brand named Hero.
- Tighten the LLM prompt with one or two sentences of context: "This brand sells X to Y. Mark anything not about X as irrelevant."
- Spot-check the first week of digests; tune until the noise floor drops below 20%.

**Caveat**: Over-tightening hides real mentions. The right calibration is "occasional false positive is fine, false negatives are catastrophic." Err toward looser keywords plus a stronger LLM prompt.

### Layer 5: Credential rotation

**Problem**: SerpAPI keys, Anthropic keys, and Gmail OAuth refresh tokens leak the same way every other secret leaks: screenshots, exported workflow JSON with embedded creds, terminated employees with admin access to the n8n instance.

**Fix**:
- Rotate the SerpAPI key when an employee with n8n access leaves. SerpAPI lets you regenerate keys instantly.
- Rotate the Anthropic key on a 90-day cadence. Anthropic's console supports multiple active keys, so you can roll without downtime.
- Never export the workflow JSON with credentials embedded; n8n's export is supposed to strip them, verify before sharing.
- The Gmail OAuth credential is bound to the user account that authorized it. Audit the connected apps list periodically.

**Caveat**: Rotating mid-week breaks the workflow until the new key is updated in n8n's credential store. Do it during a quiet hour, not Monday morning before the digest is supposed to fire.

## Priority if implementing only some

If you can only do a few:

1. ✅ **Cost runaway prevention**: read the cron one more time before clicking activate. Set a SerpAPI usage alert.
2. ✅ **Brand keyword tightening**: quote the brand string, add negatives if it's a common word.
3. ⬜ **Output destination secrecy**: confirm the inbox or Slack channel matches the data sensitivity.
4. ⬜ **Credential rotation**: schedule for the 90-day mark.
5. ⬜ **Dedup logging**: once you've been bitten by repeated mentions across days.

## What about wiring this into a public dashboard?

Don't. The digest output can include negative customer feedback and unannounced press coverage, neither of which belongs on a public surface. Keep the pipeline pointed at private inboxes or private Slack channels.

## Reporting security issues

If you find a vulnerability in this template (not a misuse, an actual flaw), please open a [GitHub security advisory](https://github.com/MinaSaad1/n8n-social-mention-digest/security/advisories/new). Don't open a public issue.
