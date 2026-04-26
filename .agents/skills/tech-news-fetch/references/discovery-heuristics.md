# Discovery Heuristics

Apply these rules during step 2 of the fetch pipeline (see [SKILL.md](../SKILL.md)) — discovering and selecting candidate articles before time-range gating, dedup, and Notion write. The rules apply equally whether discovery is via a known feed or via web search.

## Format Preference

- MUST attempt formats in this order and stop at the first one that resolves and parses cleanly: (1) RSS or Atom, (2) JSON Feed, (3) sitemap with per-URL `<lastmod>`, (4) web search followed by HTML inspection of each candidate page.
- MUST NOT default to web search when a publisher exposes a feed — feeds give canonical post IDs and publish timestamps without selector breakage on every redesign.
- SHOULD prefer Atom over RSS 2.0 when both are exposed by the same publisher — Atom's `<id>` is a stable dedup key; RSS 2.0's `<guid>` is optional and inconsistent.
- MUST treat `<updated>` (Atom) or `<pubDate>` (RSS) as the authoritative publish timestamp when reading from a feed; MUST NOT use the `Last-Modified` HTTP header as a content date — it reflects feed-file generation, not post publication.

## Reject Low-Signal Publishers

A candidate MUST be rejected before the dedup query if any of the following hold:

- **SEO content farm** — auto-generated tutorials, "Top 10 X in Year" listicles, posts that recycle other primary content. Symptoms: keyword-stuffed titles, no author byline, mid-post stock images unrelated to the technical content, ad-dense layouts.
- **Anonymous personal-platform blog with no recoverable author identity** — Wordpress / Blogspot / Medium-personal posts where no author name appears (no byline, no `<meta name="author">`, no about page reachable in two clicks). Truly anonymous posts MUST be rejected. A *named* personal blog whose author lacks the institutional signals listed in [Verify Author or Publisher Authority](#verify-author-or-publisher-authority) MUST NOT be rejected here — route it to [Content Depth Signals](#content-depth-signals), which is where solo methodology writers are accepted on the merits of the post.
- **Re-scrape of a primary announcement** — a third-party post that copies an AWS / GCP / Azure / vendor changelog without commentary. Substitute the vendor's own announcement page, or drop entirely.
- **Aggregator-only post** — a Hacker News / Reddit / Twitter / LinkedIn thread is not a primary source. Follow it through to the publisher and use *that* URL; never store the aggregator URL.

## Trusted-Authors Allow-List

Maintained at [trusted-authors.jsonl](./trusted-authors.jsonl) — JSON Lines, one entry per line, append-only with last-write-wins per `domain` on read. The skill consults this file at the start of every run and may append new lines at run end. JSONL is chosen over a JSON array so the agent can append without re-serializing the whole file, and over plain text so each entry can carry the metadata the skill and the user both need to audit trust.

Each line carries:

- `domain` — canonical, lowercase, no scheme, no path. Match against the candidate URL's host after stripping `www.`.
- `author` — human-readable name; required for audit even though the skill matches on `domain` only.
- `topics` — list of canonical [Topic options](./notion-storage.md#topic-mapping) the author writes deeply on. The author is trusted to publish in this set; posts outside it remain subject to [Off-Topic Articles](#off-topic-articles).
- `added_at` — ISO-8601 date the entry was added.
- `notes` — one-line plain-language scope; the skill does not parse this, but it MUST be present so a future reader can prune the file confidently.

Application rules:

- MUST consult the allow-list *after* [Reject Low-Signal Publishers](#reject-low-signal-publishers) but *before* [Verify Author or Publisher Authority](#verify-author-or-publisher-authority). An allow-listed candidate skips the authority check.
- MUST still apply [Reject Low-Signal Publishers](#reject-low-signal-publishers), [Verify Publish Date](#verify-publish-date), and [Off-Topic Articles](#off-topic-articles) to allow-listed candidates — the file certifies the author, not every individual post.
- Promotion: a candidate that clears all 5 [Content Depth Signals](#content-depth-signals) on first encounter MUST be promoted to the allow-list at run end. The agent appends a new line; if a line for the same `domain` already exists, the agent MUST consolidate (read-merge-write) so the file does not accumulate duplicates.
- On read, when the file contains multiple lines for the same `domain` due to interrupted writes, the skill MUST take the *latest* line and ignore the earlier ones.
- Demotion is manual only — the user prunes the file by hand. The skill MUST NOT auto-remove entries; trust, once granted, is sticky until the user revokes it.

## Verify Author or Publisher Authority

A candidate from an unfamiliar publisher MUST clear at least one of these before being accepted:

- The candidate's domain (after stripping `www.`) is already listed in [trusted-authors.jsonl](./trusted-authors.jsonl) — the skill has previously accepted this author or the user has manually added them.
- The publisher is the official engineering blog of the project the post is about (e.g., `react.dev/blog` for a React post, `expo.dev/blog` for an Expo post).
- The author has independently verifiable authority on the topic — talks at a recognized conference in the last 24 months, identifiable OSS commits to the project being discussed, or an employer affiliation aligned with the post's claims.
- The publisher is a known industry-news outlet with editorial review (InfoQ, TheNewStack, ACM Queue, IEEE Software, the free tier of The Pragmatic Engineer).

A candidate that fails *all four* but is from a *named* personal blog (not anonymous) MUST be routed to [Content Depth Signals](#content-depth-signals) before being rejected — that section is where solo methodology writers are accepted on the merits of the post itself.

## Content Depth Signals

When a candidate fails [Verify Author or Publisher Authority](#verify-author-or-publisher-authority) but is from a named personal blog, fetch the article and score it against the five signals below. A candidate clearing **at least 3 of 5** MUST be accepted as a methodology post; a candidate clearing fewer MUST be rejected.

The five signals:

1. **Code or output volume** — at least three fenced code blocks of 5+ lines each, OR one code block of 30+ lines, OR a non-trivial benchmark / configuration table. Code is a proxy for "the author wrote a runnable thing", which content farms rarely produce at depth.
2. **Length** — body text ≥1500 words, excluding code blocks and front-matter. Tweet-length posts and listicles fail this.
3. **Concrete external citations** — at least three external links to specs / RFCs / source code on GitHub or similar / official docs / academic papers. Internal site links, affiliate links, and `<link rel="canonical">` do not count.
4. **Original framing** — the post performs analysis, derivation, or step-through of the author's own work; it is not a summary of someone else's primary post. Detect via first-person reasoning about specific bugs / commits / experiments, original diagrams, or "I tried X and Y happened" structure.
5. **Specificity** — concrete versions, exact behavior, real numbers. "X took 4.2 ms vs Y's 7.8 ms on Node 22.6 with N=10k" passes; "X is faster" fails.

Application rules:

- MUST fetch the full article HTML for the depth check; counting code blocks from a search snippet is unreliable.
- SHOULD cap content-depth fetches at one per candidate per run, and skip the check entirely for domains already in the [Trusted-Authors Allow-List](#trusted-authors-allow-list).
- MUST record the signal-clear count for each accepted methodology post in the run report (e.g., "`joshwcomeau.com` — 4 of 5 signals cleared, accepted") so the user can audit boundary cases.
- A candidate that clears all 5 signals AND is not yet allow-listed MUST be promoted to the [Trusted-Authors Allow-List](#trusted-authors-allow-list) at run end per the promotion rule there.
- These signals are heuristic and partly gameable. SHOULD revisit them when content farms start producing code-shaped output that clears the bar; the allow-list is the durable trust layer.

## Verify Publish Date

When discovery is via web search rather than a feed, search-result snippets routinely lie about dates — Google surfaces older articles with recent crawl timestamps. Always verify the date on the article page itself:

- MUST fetch the article HTML and read `<meta property="article:published_time">`, `<meta name="date">`, `<meta name="datePublished">`, or JSON-LD `Article.datePublished`. MUST NOT trust the search-result snippet's date for window gating.
- When the article exposes both `published_time` and `updated_time` (or `dateModified`), MUST use `published_time` for window gating — an old article that was lightly edited yesterday is still old.
- When no machine-readable publish date is exposed and only a human-rendered date appears in the page body, MAY parse the rendered date but MUST flag the entry as "publish-date inferred" in the run report so the user can audit it.
- An article with no recoverable publish date MUST be rejected — both window-gating and downstream sorting depend on this field.

## Off-Topic Articles

- MUST drop a candidate that does not fit any of the project's covered topics. MUST NOT squeeze a security-only / DevOps-only / business article into an adjacent covered topic — wrong tags pollute the by-topic view and the topic-based ranking downstream.
- A clearly on-scope article that does not fit any *specific* covered topic MAY be tagged `General` per [notion-storage.md → Topic Mapping](./notion-storage.md#topic-mapping); a tangentially-on-scope article MUST be dropped instead.
- The dropped count and the reason (off-topic vs low-signal vs missing date) SHOULD appear in the run report so the user can distinguish "no articles published" from "articles published but rejected".

## Search Query Patterns

- SHOULD bias date-restricted queries with the calendar date in the query itself ("React Server Components April 2026") in addition to the host's date filter (Google `tbs=qdr:d` etc.) — date filters silently miss publishers whose pages don't expose a parseable date.
- SHOULD chain `site:` filters on known primary publishers when running per-topic searches, rather than open queries that surface SEO farms first.
- SHOULD search Hacker News (`hn.algolia.com/?q=<keyword>&dateRange=...`) for the time window directly to find articles that already passed at least one round of community filtering — but always follow through to the primary publisher before storing.
- MUST NOT include filler-keyword padding ("best", "ultimate", "complete guide", "in 2026") in queries — these terms bias rankings toward content farms over primary engineering writing.
