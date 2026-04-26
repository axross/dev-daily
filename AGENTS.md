# AGENTS.md

This file is the master skill index for the **dev-daily** project. It routes topics and roles to the skills under [.agents/skills/](.agents/skills/). Update this file whenever a skill is added, renamed, moved, or removed — broken index links are a routine failure mode after a rename.

The authoring discipline that governs how the skills below are written lives in [agent-skills-best-practices](.agents/skills/agent-skills-best-practices/SKILL.md). Consult it before drafting a new skill or restructuring an existing one.

## Skill Authoring

Skills in this section govern how *other* skills are written. Apply them when creating, refining, splitting, consolidating, or auditing skills in this repo.

- [agent-skills-best-practices](.agents/skills/agent-skills-best-practices/SKILL.md) — apply when creating, refining, splitting, renaming, or auditing any skill under `.agents/skills/`. Covers agentskills.io frontmatter, kebab-case naming, MECE scoping, description writing for discovery, RFC-2119 keywords, decision-vs-implementation tone, progressive disclosure (~500-line ceiling, one-level-deep references), and relative-path cross-referencing.

## Knowledge Base

Skills in this section govern the Notion databases that hold the project's curated content — their identity, schema, lifecycle, and read/write contracts. Apply them whenever any agent (importer, summarizer, manual script) reads from or writes to one of these databases.

- [notion-article-database](.agents/skills/notion-article-database/SKILL.md) — apply whenever any agent reads from or writes to the *Knowledge Base › Articles* Notion database. Covers the database / data source IDs, the property schema (Title, URL, Published At, Status, Topics; Found At auto-set), the URL canonicalization rule used for both storage and dedup, the dedup contract (URL property is NOT enforced unique by the schema), the Status lifecycle (New → Summarized → Archived) and which transitions each caller owns, the 12-option Topics controlled vocabulary plus the rule for adding a new option without clobbering the existing set, and the available views (Default, Recent, Unread, By Topic, Calendar).

## News Pipeline

Skills in this section govern how tech news, articles, and blog posts are discovered, filtered to a publish-date range, and written to the project's knowledge base.

- [tech-news-fetch](.agents/skills/tech-news-fetch/SKILL.md) — apply when retrieving and storing recent tech articles within a publish-date range into the *Knowledge Base › Articles* Notion database. Covers time-range parsing (trailing windows, offset windows, absolute ranges, single dates) and the rule to ask the user when no range is given, discovery heuristics (format preference RSS/Atom > JSON Feed > sitemap > web search; rejection of SEO content farms; the trusted-authors allow-list at `trusted-authors.jsonl`; the 5-signal content-depth scorecard for personal methodology posts; publish-date verification when discovering via search), the topic-name mapping from the project's covered domains (TypeScript/JavaScript, React/Next.js, React Native/Expo, HTML/CSS, Flutter/Dart, web platform, software architecture, UI/UX design, serverless, agentic / context / harness engineering, SWE career growth) to the database's `Topics` options, the write-call atomicity rule, and per-topic run reporting. Database-level details (identity, schema, dedup contract, lifecycle, Topics vocabulary) defer to [notion-article-database](.agents/skills/notion-article-database/SKILL.md).

## Knowledge Synthesis

Skills in this section govern how queued articles are summarized into their database rows and synthesized into reusable wiki pages under the Knowledge Base.

- [knowledge-ingestion](.agents/skills/knowledge-ingestion/SKILL.md) — apply when processing the *Knowledge Base › Articles* queue end-to-end: reading every row with `Status = New`, writing a structured 1–12 section summary into the row's page content, transitioning the row to `Summarized`, then synthesizing wiki-style reference sub-pages under the Knowledge Base index page for summaries that warrant foundational treatment. Covers the summarize-then-synthesize procedure (queue read, source retrieval with WebFetch + WebSearch fallback, summary format and length, wiki-or-skip decision, wiki page structure and length targets, filing under the Knowledge Base index), and the Knowledge Base index-page conventions (H3 for topic sections, H4 for sub-topics, the canonical `<page url="...">Title</page>` link block format, dashed-UUID requirement). Database-level details (identity, schema, status lifecycle, dedup, Topics vocabulary) defer to [notion-article-database](.agents/skills/notion-article-database/SKILL.md); article discovery and import defer to [tech-news-fetch](.agents/skills/tech-news-fetch/SKILL.md).

## Adding a New Skill

When adding a skill to this repo:

1. Author the skill following [agent-skills-best-practices](.agents/skills/agent-skills-best-practices/SKILL.md).
2. Place its directory under `.agents/skills/<skill-name>/` with `SKILL.md` plus any reference files.
3. Add a bullet to the section above that best fits the skill's role. If no existing section fits, add a new H2 here in the same change — keep section names role/topic-shaped, not actor-shaped.
4. Verify every relative link in the new bullet resolves on disk before finalizing.
