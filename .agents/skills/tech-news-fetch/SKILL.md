---
name: tech-news-fetch
description: Apply this skill when the user wants to find, retrieve, and store recent tech news, articles, or blog posts (English or Japanese / 日本語) within a publish-date range into the dev-daily Knowledge Base › Articles database in Notion. Triggers include "fetch articles from the last 7 days", "grab today's React posts", "pull the latest serverless news", "save these to Notion", "import this week's articles", "import 2026-02-13 - 2026-03-28", "日本語の記事も取り込んで", or any retrieve-and-store request across the project's covered topics (canonical data in `references/topics.jsonl`; schema and procedure in `references/covered-topics.md` — JavaScript/TypeScript, React/Next.js, React Native/Expo, HTML/CSS, Flutter/Dart, web technology, UI/UX design, serverless, agentic/context/harness engineering, SWE career growth). Covers time-range parsing and asking the user when no range is given, the bilingual language-coverage mandate (English + Japanese on every run unless the user scopes one), the Notion target schema (Title, URL, Published At, Status=New, Topics; Found At is auto-set), the topic-name mapping from covered topics to existing Notion Topics options (data in `references/topics.jsonl`, application rules in `references/covered-topics.md`), when to create a new option, and dedup against existing entries.
---

# Tech News Fetch

Apply this skill when the user wants to retrieve recent tech news, articles, or blog posts within a specific publish-date range and store them as new entries in the dev-daily *Knowledge Base › Articles* Notion database. Coverage is bilingual: English-language and Japanese-language (日本語) primary sources are equally in scope on every run.

This skill governs the **end-to-end fetch-and-store pipeline**: (1) resolve the user's time range or ask when none is given, (2) discover and fetch articles published within that range across the project's covered topics — enumerated in the canonical data file [references/topics.jsonl](./references/topics.jsonl) (schema and rules in [references/covered-topics.md](./references/covered-topics.md)) — and across both English and Japanese sources, (3) dedup against rows already in the Articles database, (4) write each surviving entry with `Status="New"`. The database itself — identity, property schema, URL canonicalization, dedup contract, Status lifecycle, and Topics vocabulary — is governed by [../notion-article-database/SKILL.md](../notion-article-database/SKILL.md); load it before any read or write to the Articles database. Downstream summarization is out of scope; `Status="New"` hands each entry off to the next stage.

## Covered Topics

The project's editorial scope is split across a structured-data file and a documentation companion:

- [topics.jsonl](./references/topics.jsonl) — canonical data: the topic list. Each line carries `name` (the exact Notion `Topics` option string, byte-for-byte) and `keywords` (a short array, ≥1 entry, spelling out what falls under the topic — e.g., `Agentic/AI Coding` → `["Context Engineering", "Harness Engineering", "Agent Skills"]`, or `React` → `["React"]`). Both fields are required. Agents MUST consult `keywords` to decide which Notion `Topics` tags to apply to each article — that's the canonical scope spec — but MUST NOT treat it as a literal substring-match gate. Parse this file to enumerate scope and look up which Notion option to write; do NOT paraphrase from the markdown.
- [covered-topics.md](./references/covered-topics.md) — schema and prose: field definitions, the rationale for JSONL, application rules for picking `Topics` values (match the post not the publisher, max three options, `General` only as a last resort), reserved-but-not-yet-used optional fields, and the procedure for adding / removing / renaming a covered topic.

Load both at step 2 of every fetch run (before discovery candidates are scored against scope) and at step 4 (before writing any entry's `Topics` property). Skills that decide whether a candidate is in-scope MUST consult these files rather than enumerating topics inline.

## Time-Range Handling

See [time-range-handling.md](./references/time-range-handling.md) for:

- Recognized phrasings — trailing windows ("in this 24 hours", "in the last 7 days"), offset windows ("from 5 days ago to 2 days ago"), absolute ranges ("2026-02-13 - 2026-03-28"), and single absolute dates
- The rule to ask the user when the prompt has no range, an unrecognized phrase, or only one endpoint — plus the question template to use
- Normalization to a closed `[start, end]` UTC interval, anchored to the project's current date and inclusive on both ends at day granularity
- Edge cases — future endpoints, time-zone-tagged inputs, oversized ranges, empty results, and the prohibition on widening to make the result non-empty

Load this file at the start of every fetch run, before any source iteration.

## Discovery Heuristics

See [discovery-heuristics.md](./references/discovery-heuristics.md) for:

- The bilingual language-coverage mandate — English and Japanese on every run unless the user scopes one
- The format-preference order — RSS / Atom > JSON Feed > sitemap > web search + HTML inspection — and the rule against scraping when a feed exists
- Hard rejection rules for low-signal publishers (SEO content farms, *unattributable* personal-platform blogs, vendor-announcement re-scrapes, aggregator-only posts) — a stable handle plus an external identity link counts as recoverable identity
- The publisher blocklist at [publisher-blocklist.jsonl](./references/publisher-blocklist.jsonl) (JSONL, append-only) — the only static gate; everything not on it flows through the same content-depth scorecard
- The per-run exploration mandate — every run MUST surface at least one candidate from a publisher/author not previously written to the Articles DB; report a discovery anomaly if none surfaces
- Author / publisher authority criteria, with the personal-blog escape route to the content-depth scorecard
- The five content-depth signals (code volume, length — ≥1500 English words OR ≥2,500 Japanese characters, concrete external citations, original framing, specificity) applied uniformly to every personal-blog candidate
- Per-user publishing platforms (`zenn.dev`, `qiita.com`, `note.com`) — host is the platform, not the author; always route through the content-depth scorecard, never grant blanket trust
- Publish-date verification when discovery is via web search — fetching the article HTML for `article:published_time`, JSON-LD, or rendered-date fallback, and the rule to reject articles whose publish date cannot be recovered
- The off-topic-rejection rule and the boundary between `General`-eligible and outright drop
- Search query patterns that bias toward primary writing rather than content farms — including Japanese-language query templates for Japanese-source discovery

Load this file at step 2 of every fetch run, before applying the time-range filter to candidates.

## Notion Storage

See [notion-storage.md](./references/notion-storage.md) for:

- The write-call atomicity rule (Title, URL, Published At, Status, Topics in a single create call) and the `Status="New"` invariant for new fetch entries — the topic-mapping data itself lives in [topics.jsonl](./references/topics.jsonl) (rules in [covered-topics.md](./references/covered-topics.md))
- Per-run reporting requirements — written / skipped / rejected counts broken down by topic AND by language (English vs Japanese), including topics with zero in-window hits, plus content-depth signal counts for accepted methodology posts and a discovery-anomaly flag if zero new sources surfaced
- The post-run verification step — spot-check the first written entry in the `Recent` view, and confirm a freshly added `Topics` option appears on the written page

Load this file when writing entries; database-level reads and writes are governed by [../notion-article-database/SKILL.md](../notion-article-database/SKILL.md).
