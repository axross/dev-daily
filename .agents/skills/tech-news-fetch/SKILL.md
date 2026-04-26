---
name: tech-news-fetch
description: Apply this skill when the user wants to find, retrieve, and store recent tech news, articles, or blog posts within a publish-date range into the dev-daily Knowledge Base › Articles database in Notion. Triggers include "fetch articles from the last 7 days", "grab today's React posts", "pull the latest serverless news", "save these to Notion", "import this week's articles", "import 2026-02-13 - 2026-03-28", or any retrieve-and-store request across the covered topics: TypeScript/JavaScript, React/Next.js, React Native/Expo, HTML/CSS, Flutter/Dart, web platform, software architecture, UI/UX design, serverless, agentic / context / harness engineering, and SWE career growth. Covers time-range parsing and asking the user when no range is given, the Notion target schema (Title, URL, Published At, Status=New, Topics; Found At is auto-set), topic-name mapping from covered domains to existing Topics options, when to create a new option, and dedup against existing entries.
---

# Tech News Fetch

Apply this skill when the user wants to retrieve recent tech news, articles, or blog posts within a specific publish-date range and store them as new entries in the dev-daily *Knowledge Base › Articles* Notion database.

This skill governs the **end-to-end fetch-and-store pipeline**: (1) resolve the user's time range or ask when none is given, (2) discover and fetch articles published within that range across the project's covered topics (TypeScript/JavaScript, React/Next.js, React Native/Expo, HTML/CSS, Flutter/Dart, web platform, software architecture, UI/UX design, serverless, agentic / context / harness engineering, SWE career growth), (3) dedup against rows already in the Articles database, (4) write each surviving entry with `Status="New"`. The database itself — identity, property schema, URL canonicalization, dedup contract, Status lifecycle, and Topics vocabulary — is governed by [../notion-article-database/SKILL.md](../notion-article-database/SKILL.md); load it before any read or write to the Articles database. Downstream summarization is out of scope; `Status="New"` hands each entry off to the next stage.

## Time-Range Handling

See [time-range-handling.md](./references/time-range-handling.md) for:

- Recognized phrasings — trailing windows ("in this 24 hours", "in the last 7 days"), offset windows ("from 5 days ago to 2 days ago"), absolute ranges ("2026-02-13 - 2026-03-28"), and single absolute dates
- The rule to ask the user when the prompt has no range, an unrecognized phrase, or only one endpoint — plus the question template to use
- Normalization to a closed `[start, end]` UTC interval, anchored to the project's current date and inclusive on both ends at day granularity
- Edge cases — future endpoints, time-zone-tagged inputs, oversized ranges, empty results, and the prohibition on widening to make the result non-empty

Load this file at the start of every fetch run, before any source iteration.

## Discovery Heuristics

See [discovery-heuristics.md](./references/discovery-heuristics.md) for:

- The format-preference order — RSS / Atom > JSON Feed > sitemap > web search + HTML inspection — and the rule against scraping when a feed exists
- Hard rejection rules for low-signal publishers (SEO content farms, *anonymous* personal-platform blogs, vendor-announcement re-scrapes, aggregator-only posts) — *named* personal blogs without institutional authority are NOT rejected here
- The trusted-authors allow-list at [trusted-authors.jsonl](./references/trusted-authors.jsonl) (JSONL, append-only) — domains short-circuit the authority check; auto-promoted on a 5-of-5 content-depth pass; user prunes by hand
- Author / publisher authority criteria, with the personal-blog escape route to the content-depth scorecard
- The five content-depth signals (code volume, length ≥1500 words, concrete external citations, original framing, specificity) used to accept methodology posts from named personal blogs that fail institutional-authority checks
- Publish-date verification when discovery is via web search — fetching the article HTML for `article:published_time`, JSON-LD, or rendered-date fallback, and the rule to reject articles whose publish date cannot be recovered
- The off-topic-rejection rule and the boundary between `General`-eligible and outright drop
- Search query patterns that bias toward primary writing rather than content farms

Load this file at step 2 of every fetch run, before applying the time-range filter to candidates.

## Notion Storage

See [notion-storage.md](./references/notion-storage.md) for:

- The fetch-pipeline-specific topic-mapping table — covered domains → existing options in the database's [Topics vocabulary](../notion-article-database/SKILL.md#topics-vocabulary), plus the rule to keep this table in sync when a new option is added to the database
- The write-call atomicity rule (Title, URL, Published At, Status, Topics in a single create call) and the `Status="New"` invariant for new fetch entries
- Per-run reporting requirements — written / skipped / rejected counts broken down by topic, including topics with zero in-window hits, plus content-depth signal counts for accepted methodology posts
- The post-run verification step — spot-check the first written entry in the `Recent` view, and confirm a freshly added `Topics` option appears on the written page

Load this file when writing entries; database-level reads and writes are governed by [../notion-article-database/SKILL.md](../notion-article-database/SKILL.md).
