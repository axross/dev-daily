---
name: notion-article-database
description: Apply this skill whenever any agent reads from or writes to the Knowledge Base › Articles database in Notion (the dev-daily knowledge base of tech articles). Triggers include "save this article to Notion", "add to articles DB", "find unread articles", "mark as summarized", "what's in the articles database", "import to knowledge base", "create a new Topics option", "is this URL already in the database", or any operation that touches the Articles database in Notion. Covers the database / data source IDs, the property schema (Title, URL, Found At auto, Published At, Status, Topics), the Status lifecycle (New → Summarized → Archived) and which transitions each caller owns, the 12-option Topics controlled vocabulary plus the rule for adding a new option, the URL canonicalization rule used for both storage and dedup, the dedup contract (the URL property is NOT enforced unique by the schema), and the available views (Default, Recent, Unread, By Topic, Calendar).
---

# Notion Article Database

Apply this skill whenever any agent reads from or writes to the *Knowledge Base › Articles* Notion database — regardless of which caller is performing the operation. The skill governs the database's identity, schema, lifecycle, and read/write contracts so callers (the dev-daily content pipeline, summarizers, manual scripts) stay aligned on the same invariants.

This skill describes **the database itself**, not any particular pipeline that feeds it. Article discovery, time-range gating, source curation, and run-level reporting belong to upstream caller skills (e.g., [../tech-news-fetch/SKILL.md](../tech-news-fetch/SKILL.md) for the dev-daily import path); summarization and archival belong to downstream caller skills that own the `New → Summarized` and `Summarized → Archived` transitions. Callers MUST follow the rules below when their work touches the Articles database.

## Identity

- **Database** — `Knowledge Base › Articles`, parent page `Knowledge Base` (`https://www.notion.so/34ba88827414805d8ef8ea0e95f49bd6`).
- **Database ID** — `86483b1b-526c-4691-a2d4-9983bc6aa718`.
- **Data source ID** — `collection://7699d23b-2335-4e3c-a802-1305396188e6`.
- MUST query and write against the data source ID, not the database ID, when the host's Notion tooling distinguishes them — multi-source databases route writes by data source.

## Property Schema

The data source has six properties; their contracts are:

- **Title** (`title`) — the article's published title. Callers SHOULD trim trailing publisher suffixes before writing. Recognize both ASCII separators (`|`, `-`) and full-width separators commonly used by Japanese publishers (`｜`, `―`, `ー`, `─`, `–`). Examples to trim: `" | Vercel Blog"`, `" - DEV Community"`, `" | ZOZO TECH BLOG"`, `" ｜ サイボウズ Inside Out"`, `" ― メルカリエンジニアリング"`. Preserve the original-language title verbatim otherwise — MUST NOT translate.
- **URL** (Notion-internal name `userDefined:URL`, type `url`) — the canonical publisher URL after the rule in [URL Canonicalization](#url-canonicalization). Callers MUST NOT store aggregator wrappers, tracker proxies, or trailing UTM params.
- **Published At** (`date`) — the article's actual publish date or datetime in UTC. Callers MUST source from `<updated>` (Atom) / `<pubDate>` (RSS) when reading from a feed, or from `<meta property="article:published_time">` / JSON-LD `Article.datePublished` when reading from HTML. MUST NOT use the `Last-Modified` HTTP header — it reflects file modification, not post publication. Set `is_datetime: 1` only when a full timestamp is available; otherwise `is_datetime: 0` so the calendar view groups by date cleanly.
- **Found At** (`created_time`) — auto-set by Notion when the row is created. Agents MUST NOT attempt to write this; the property is read-only.
- **Status** (`select`) — one of `New`, `Summarized`, `Archived`; see [Status Lifecycle](#status-lifecycle).
- **Topics** (`multi_select`) — one or more of the 12 options listed in [Topics Vocabulary](#topics-vocabulary).

## URL Canonicalization

Apply these steps in order before storing a URL in the `URL` property and before any dedup comparison; the result is the canonical URL used everywhere downstream:

1. Lowercase the scheme and host.
2. Drop the URL fragment (`#…`).
3. Strip tracking query params: `utm_*`, `fbclid`, `gclid`, `ref`, `mc_cid`, `mc_eid`.
4. Remove a single trailing slash from the path unless the path is just `/`.
5. Unwrap aggregator / tracker redirect proxies (e.g., `t.co`, `news.google.com/articles`, `lnkd.in`) to the publisher's own URL.

- MUST canonicalize at fetch time, not just at write time, so the same URL identity is used by every caller in the pipeline (dedup, write, audit logs).
- Canonicalization MUST be deterministic — two callers normalizing the same input MUST produce byte-identical output.

## Dedup Contract

- The `URL` property is NOT enforced unique by the schema (the SQLite-shape `url UNIQUE` column refers to the Notion page URL, not to this property). Notion silently accepts duplicates if not pre-checked.
- MUST query the data source for any existing row with the same canonical URL before writing a new entry. Treat any match — regardless of `Status` — as a duplicate and skip the write.
- MUST NOT dedup by `Title` alone — distinct articles share generic titles like `"Release notes"` or `"Weekly update"`.
- SHOULD batch dedup queries when the host's Notion tooling supports a multi-URL query, to avoid one round trip per entry.

## Status Lifecycle

The `Status` select drives the entry's place in the pipeline. Flow: `New → Summarized → Archived`. Ownership rules:

- **New** — set by the importer when a fresh article lands. Importers MUST write `Status=New` and MUST NOT set `Summarized` or `Archived` directly.
- **Summarized** — set by the summarization stage once a summary has been produced for the entry. Summarizers MUST transition only from `New`, never from `Archived` (regression).
- **Archived** — set when the entry no longer surfaces in the `Unread` view. Archival MAY be performed manually (the user) or by a scheduled cleanup; callers writing this transition SHOULD do so only when they own the cleanup.
- A caller MUST NOT regress the lifecycle (e.g., `Summarized → New` or `Archived → New`); regressions break downstream queue logic and resurface noise.
- MUST NOT add new Status options without updating this skill — `Status` is a fixed three-state lifecycle, not a free-form tag.

## Topics Vocabulary

The `Topics` multi-select uses a controlled vocabulary. This is the **Notion-side tag set**, a superset of the project's editorial scope (the covered topics enumerated in the structured-data file [../tech-news-fetch/references/topics.jsonl](../tech-news-fetch/references/topics.jsonl); schema and rules in the companion [covered-topics.md](../tech-news-fetch/references/covered-topics.md)). Each JSONL entry's `name` field IS the exact Notion option string — the two are unified, byte-for-byte, so there is no separate mapping table. Cross-cutting options (`Software Architecture`, `General`) appear in this vocabulary without a corresponding JSONL entry; they exist for tagging *within* an in-scope post and are not part of the editorial scope.

Current options (17), exact strings:

Active (mirror the editorial scope in [topics.jsonl](../tech-news-fetch/references/topics.jsonl), plus cross-cutting): `JavaScript/TypeScript`, `React`, `Next.js`, `React Native/Expo`, `Flutter/Dart`, `HTML/CSS`, `Web Technology`, `UI/UX Design`, `Serverless`, `Agentic/AI Coding`, `Software Architecture` (cross-cutting), `Career`, `General` (cross-cutting last-resort).

Legacy (kept so historical entries don't lose their tags; new writes MUST NOT use these): `React & Next.js`, `React Native & Expo`, `Flutter & Dart`, `HTML & CSS`. These were superseded when the editorial scope split `React`/`Next.js` and adopted `X/Y` form throughout. They will be removed once all historical entries are migrated to the active equivalents (a deliberate user-driven cleanup).

- Callers MUST pick option(s) matching the *post itself*, not the publisher's primary topic. SHOULD NOT exceed 3 options per entry.
- `General` is a last-resort fallback for clearly on-scope posts that fit no specific option. MUST NOT use as a synonym for "I didn't classify this".
- When no existing option fits and `General` would be misleading, callers MAY add a new option via the data-source update tool; MUST add the option *before* writing any entry that references it (Notion rejects writes referencing unknown option names) and MUST list every existing option in the `ALTER COLUMN ... SET MULTI_SELECT(...)` statement, since the host's update tool replaces the option set rather than appending.
- After adding a new option, MUST verify all pre-existing option names still appear in the post-mutation schema before any write proceeds — a missing option means the replace clobbered data and MUST stop the run for the user to recover.
- New option names SHOULD follow the existing format conventions: title case, ampersand for two-word pairs (`HTML & CSS`), slash for synonyms (`UI/UX Design`); when all 10 Notion option colors are already used, MAY pick `default` over a topic-specific color so the new option is visually distinguishable from canonical-color options.
- Caller skills that mirror this vocabulary MUST stay aligned with the database. The fetch pipeline's editorial scope lives in [../tech-news-fetch/references/topics.jsonl](../tech-news-fetch/references/topics.jsonl), where each entry's `name` is the exact Notion option string; add or rename a JSONL entry in the same change that introduces or renames the corresponding Notion option here.

## Views

Five pre-configured views are available. Callers SHOULD use these rather than re-implementing equivalent filters and sorts:

- **Default view** — table, all rows, all properties shown.
- **Recent** — table, sorted by `Published At` desc; use when surfacing recently-published items regardless of `Status`.
- **Unread** — table, filtered to `Status=New`, sorted by `Published At` desc. This is the work queue for the summarization stage.
- **By Topic** — board, grouped by `Topics`; use for editorial overviews and topic-coverage audits.
- **Calendar** — calendar by `Published At`; use for date-based exploration.
