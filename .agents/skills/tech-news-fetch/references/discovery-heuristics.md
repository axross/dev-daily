# Discovery Heuristics

Apply these rules during step 2 of the fetch pipeline (see [SKILL.md](../SKILL.md)) — discovering and selecting candidate articles before time-range gating, dedup, and Notion write. The rules apply equally whether discovery is via a known feed or via web search, and equally to English-language and Japanese-language sources.

## Language Coverage

- MUST attempt both English-language and Japanese-language (日本語) primary sources on every fetch run unless the user explicitly scopes to one. Coverage is bilingual by default; an English-only or Japanese-only result is a partial run and MUST be reported as such in the run report.
- MUST seed Japanese-language discovery with at least one Japanese-publisher feed and at least one Japanese-language search query (see [Search Query Patterns](#search-query-patterns)) on every run, even when the user's prompt is in English.
- MUST NOT translate Japanese titles, URLs, or content snippets when storing or searching; the original-language title is the canonical title for both dedup and downstream summarization.
- All depth, authority, and rejection rules below apply uniformly to both languages — the only language-specific concession is the character-count fallback in [Content Depth Signals](#content-depth-signals) Signal #2.

## Format Preference

- MUST attempt formats in this order and stop at the first one that resolves and parses cleanly: (1) RSS or Atom, (2) JSON Feed, (3) sitemap with per-URL `<lastmod>`, (4) web search followed by HTML inspection of each candidate page.
- MUST NOT default to web search when a publisher exposes a feed — feeds give canonical post IDs and publish timestamps without selector breakage on every redesign.
- SHOULD prefer Atom over RSS 2.0 when both are exposed by the same publisher — Atom's `<id>` is a stable dedup key; RSS 2.0's `<guid>` is optional and inconsistent.
- MUST treat `<updated>` (Atom) or `<pubDate>` (RSS) as the authoritative publish timestamp when reading from a feed; MUST NOT use the `Last-Modified` HTTP header as a content date — it reflects feed-file generation, not post publication.

## Reject Low-Signal Publishers

A candidate MUST be rejected before the dedup query if any of the following hold:

- **Listed on the publisher blocklist** — see [Publisher Blocklist](#publisher-blocklist). This is the only static gate; check it first.
- **SEO content farm** — auto-generated tutorials, "Top 10 X in Year" listicles, posts that recycle other primary content. Symptoms: keyword-stuffed titles, no author byline, mid-post stock images unrelated to the technical content, ad-dense layouts. When a host shows this pattern repeatedly, append it to the blocklist per [Publisher Blocklist](#publisher-blocklist).
- **Unattributable personal-platform blog** — Wordpress / Blogspot / Medium-personal posts where no recoverable identity exists. A *handle-only* author (e.g., `uhyo`, `mizchi`, `jxck`, `simonw`) is NOT unattributable as long as the site or post links to a stable external identity (GitHub, Mastodon, Twitter, talks index, or a publisher about page reachable in two clicks). Reject only if there is no byline AND no `<meta name="author">` AND no recoverable handle-with-identity-link.
- **Re-scrape of a primary announcement** — a third-party post that copies an AWS / GCP / Azure / vendor changelog without commentary. Substitute the vendor's own announcement page, or drop entirely.
- **Aggregator-only post** — a Hacker News / Reddit / Twitter / LinkedIn / はてなブックマーク thread is not a primary source. Follow it through to the publisher and use *that* URL; never store the aggregator URL.

## Publisher Blocklist

Maintained at [publisher-blocklist.jsonl](./publisher-blocklist.jsonl) — JSON Lines, one entry per line, append-only. The skill consults this file at the start of every run and may append new lines at run end. JSONL is chosen over a JSON array so the agent can append without re-serializing the whole file.

Each line carries:

- `domain` — canonical, lowercase, no scheme, no path. Match against the candidate URL's host after stripping `www.`.
- `reason` — one-line plain-language description of the rejection rationale (SEO farm, listicle network, etc.). The skill does not parse this, but it MUST be present so future readers and the user can audit and prune.
- `added_at` — ISO-8601 date the entry was added.

Application rules:

- MUST consult the blocklist as the *first* check in [Reject Low-Signal Publishers](#reject-low-signal-publishers). A match is a hard reject; no further checks run on that candidate.
- MUST NOT maintain any positive trust list. There is no allow-list, no trusted-authors registry, and no domain-based short-circuit of the authority or content-depth checks. Every non-blocked candidate flows through the same checks.
- Append-on-evidence: when the agent encounters a host that produces SEO-farm-pattern output and fails the content-depth scorecard, MUST append a blocklist entry at run end with a one-line `reason`.
- MUST NOT auto-remove blocklist entries — pruning is a deliberate user action.
- The blocklist is a **closed-world set of known bad sources**, not a closed-world set of known good ones. New publishers and new authors are accepted by default and judged on the merits of the post via [Content Depth Signals](#content-depth-signals); this is what keeps the discovery surface open to writers the user has not yet encountered.

## Per-User Publishing Platforms

Some hosts are platforms where each post is by a different author (the host is not the publisher). Treat the following as platforms, not publishers:

- `zenn.dev` — per-user articles and books at `zenn.dev/<user>/articles/<slug>` and `zenn.dev/<user>/books/<slug>`.
- `qiita.com` — per-user posts at `qiita.com/<user>/items/<id>`.
- `note.com` — per-user notes at `note.com/<user>/n/<id>`.
- Medium and its publication subdomains — already covered by [Publisher Blocklist](#publisher-blocklist); do not relitigate per-author.

Application rules:

- MUST NOT grant blanket trust to any per-user platform host. The host signals nothing about the post's depth or authority.
- MUST route every candidate from a per-user platform through [Content Depth Signals](#content-depth-signals), regardless of how prolific the author is or whether prior posts from the same user have been accepted.
- MAY use the per-user platform's RSS feed as a discovery mechanism (`zenn.dev/feed`, `qiita.com/popular-items/feed`, etc.) — that's allowed under [Format Preference](#format-preference) — but feed presence does not skip the depth check.
- The institutional-authority check in [Verify Author or Publisher Authority](#verify-author-or-publisher-authority) does not short-circuit per-user-platform candidates either; the depth check is the only gate.

## Verify Author or Publisher Authority

A candidate MUST clear at least one of these before being accepted, EXCEPT when the candidate is from a per-user publishing platform (see [Per-User Publishing Platforms](#per-user-publishing-platforms)) — those go straight to the content-depth scorecard:

- The publisher is the official engineering blog of the project the post is about (e.g., `react.dev/blog` for a React post, `expo.dev/blog` for an Expo post, `flutter.dev` / `blog.flutter.dev` for a Flutter post).
- The publisher is a well-known company engineering blog with editorial review (`engineering.mercari.com`, `developers.cyberagent.co.jp`, `techblog.lycorp.co.jp`, `engineering.dena.com`, `blog.cybozu.io`, `buildersbox.corp-sansan.com`, `tech.smarthr.jp`, `tech.pepabo.com`, `techblog.zozo.com`, `github.blog`, `blog.cloudflare.com`, `vercel.com/blog`, etc.). The list is illustrative, not exhaustive — analogous company blogs with the same editorial pattern qualify.
- The author has independently verifiable authority on the topic — talks at a recognized conference in the last 24 months, identifiable OSS commits to the project being discussed, or an employer affiliation aligned with the post's claims.
- The publisher is a known industry-news outlet with editorial review (InfoQ, TheNewStack, ACM Queue, IEEE Software, the free tier of The Pragmatic Engineer, Smashing Magazine, CSS-Tricks, Stack Overflow Blog).

A candidate that fails *all* of the above but is from a *named or handle-attributable* personal blog (not unattributable per [Reject Low-Signal Publishers](#reject-low-signal-publishers)) MUST be routed to [Content Depth Signals](#content-depth-signals) before being rejected — that section is where solo methodology writers are accepted on the merits of the post itself.

## Content Depth Signals

When a candidate fails [Verify Author or Publisher Authority](#verify-author-or-publisher-authority), or is from a per-user publishing platform, fetch the article and score it against the five signals below. A candidate clearing **at least 3 of 5** MUST be accepted as a methodology post; a candidate clearing fewer MUST be rejected.

The five signals:

1. **Code or output volume** — at least three fenced code blocks of 5+ lines each, OR one code block of 30+ lines, OR a non-trivial benchmark / configuration table. Code is a proxy for "the author wrote a runnable thing", which content farms rarely produce at depth.
2. **Length** — body text ≥**1500 words for English** or ≥**2,500 characters for Japanese** (excluding code blocks and front-matter). The two thresholds are equivalent in information density: 1 English word ≈ 1.5–2 Japanese characters in dense technical writing. Detect language by the dominant character class of the body (Latin alphabet → English threshold; CJK characters > 50% → Japanese threshold). Tweet-length posts and listicles fail this in either language.
3. **Concrete external citations** — at least three external links to specs / RFCs / source code on GitHub or similar / official docs / academic papers. Internal site links, affiliate links, and `<link rel="canonical">` do not count.
4. **Original framing** — the post performs analysis, derivation, or step-through of the author's own work; it is not a summary of someone else's primary post. Detect via first-person reasoning about specific bugs / commits / experiments, original diagrams, or "I tried X and Y happened" structure.
5. **Specificity** — concrete versions, exact behavior, real numbers. "X took 4.2 ms vs Y's 7.8 ms on Node 22.6 with N=10k" passes; "X is faster" fails.

Application rules:

- MUST fetch the full article HTML for the depth check; counting code blocks from a search snippet is unreliable.
- MUST apply the same five signals uniformly — there is no per-author short-circuit, no cache of "previously cleared" authors, and no domain-based skip. Familiarity with an author is irrelevant; the post stands or falls on its own.
- MUST record the signal-clear count for each accepted methodology post in the run report (e.g., "`joshwcomeau.com` — 4 of 5 signals cleared, accepted").
- These signals are heuristic and partly gameable. SHOULD revisit them when SEO content farms start producing code-shaped output that clears the bar.

## Per-Run Exploration Mandate

The skill is biased toward open discovery: every run MUST surface fresh sources, not just re-poll the same hosts.

- Every fetch run MUST surface at least **one** candidate from a publisher or author whose canonical URL host (after stripping `www.`) does not appear in any existing row of the Articles database.
- When the run produces zero candidates from new hosts, MUST flag a **discovery anomaly** in the run report and list which discovery channels (RSS feeds, search queries, aggregator follow-throughs) were exercised. A persistent anomaly across consecutive runs is a signal that the discovery surface has narrowed and SHOULD prompt the user to broaden search seeds.
- This mandate exists to counteract the natural tendency of any agent — even one without an allow-list — to re-fetch the same RSS feeds run after run. A live, growing knowledge base requires deliberate exploration, not just incremental harvesting.

## Verify Publish Date

When discovery is via web search rather than a feed, search-result snippets routinely lie about dates — Google surfaces older articles with recent crawl timestamps. Always verify the date on the article page itself:

- MUST fetch the article HTML and read `<meta property="article:published_time">`, `<meta name="date">`, `<meta name="datePublished">`, or JSON-LD `Article.datePublished`. MUST NOT trust the search-result snippet's date for window gating.
- When the article exposes both `published_time` and `updated_time` (or `dateModified`), MUST use `published_time` for window gating — an old article that was lightly edited yesterday is still old.
- When no machine-readable publish date is exposed and only a human-rendered date appears in the page body, MAY parse the rendered date but MUST flag the entry as "publish-date inferred" in the run report so the user can audit it. Japanese pages frequently render dates as `2026年4月23日` or `2026/04/23` — both are parseable.
- An article with no recoverable publish date MUST be rejected — both window-gating and downstream sorting depend on this field.

## Off-Topic Articles

- MUST drop a candidate that does not fit any of the project's covered topics, as enumerated in [topics.jsonl](./topics.jsonl) — the single source of truth (schema and rules in the companion [covered-topics.md](./covered-topics.md)). MUST NOT squeeze a security-only / DevOps-only / business article into an adjacent covered topic — wrong tags pollute the by-topic view and the topic-based ranking downstream.
- A clearly on-scope article that does not fit any *specific* covered topic MAY be tagged `General` per [covered-topics.md → Application Rules](./covered-topics.md#application-rules); a tangentially-on-scope article MUST be dropped instead.
- The dropped count and the reason (off-topic vs low-signal vs missing date) SHOULD appear in the run report so the user can distinguish "no articles published" from "articles published but rejected".

## Search Query Patterns

- SHOULD bias date-restricted queries with the calendar date in the query itself in addition to the host's date filter (Google `tbs=qdr:d` etc.) — date filters silently miss publishers whose pages don't expose a parseable date. For English-language searches, the calendar date format is `April 2026` or `Apr 2026` (e.g., `React Server Components April 2026`). For Japanese-language searches, the calendar date format is `2026年4月` or `2026/04` (e.g., `React Server Components 2026年4月`).
- SHOULD run separate query passes per language: an English-keyword pass (e.g., `React Server Components April 2026`) AND a Japanese-keyword pass (e.g., `React Server Components 2026年4月` or topic-localized variants like `Reactサーバーコンポーネント 2026年4月`). Running only the English pass silently excludes Japanese-language sources.
- SHOULD chain `site:` filters on known primary publishers when running per-topic searches, rather than open queries that surface SEO farms first. Useful Japanese-publisher seeds (illustrative, not exhaustive): `site:zenn.dev`, `site:qiita.com`, `site:engineering.mercari.com`, `site:developers.cyberagent.co.jp`, `site:techblog.lycorp.co.jp`, `site:engineering.dena.com`, `site:blog.cybozu.io`, `site:buildersbox.corp-sansan.com`, `site:tech.smarthr.jp`, `site:tech.pepabo.com`, `site:techblog.zozo.com`, `site:blog.jxck.io`.
- SHOULD search Hacker News (`hn.algolia.com/?q=<keyword>&dateRange=...`) for the time window directly to find articles that already passed at least one round of community filtering — but always follow through to the primary publisher before storing. The Japanese equivalent is はてなブックマーク (`b.hatena.ne.jp/q/<keyword>`); same rule applies — follow through to the primary publisher.
- MUST NOT include filler-keyword padding ("best", "ultimate", "complete guide", "in 2026", "決定版", "完全ガイド") in queries — these terms bias rankings toward content farms over primary engineering writing in either language.
