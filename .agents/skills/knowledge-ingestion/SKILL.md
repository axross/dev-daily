---
name: knowledge-ingestion
description: Apply this skill to process the Knowledge Base › Articles queue end-to-end — read every row with Status "New", write a structured 1–12 section summary into the row's page content, transition the row to "Summarized", then synthesize wiki-style reference sub-pages under the Knowledge Base page for summaries that warrant foundational treatment. Trigger whenever the user asks to "process my Knowledge Base", "summarize new articles", "ingest my reading queue", "catch me up on my KB", "run the weekly KB refresh", or any unread/new-articles request against the user's Notion knowledge base — even when the user does not name "Notion" or "wiki" explicitly. Do not trigger for general Notion editing, single-article summarization unrelated to the queue, or wiki creation without a corresponding queued row.
---

# Knowledge Ingestion

Process the *Knowledge Base › Articles* queue end-to-end: every row with `Status = New` is summarized into its own page content and transitioned to `Summarized`; summaries that warrant foundational treatment then become wiki-style reference sub-pages filed under the Knowledge Base index page.

This skill governs **the summarize-then-synthesize procedure** and **the Knowledge Base index-page conventions** that wiki filing relies on. Articles database identity, property schema, status lifecycle, dedup contract, and Topics vocabulary defer to [../notion-article-database/SKILL.md](../notion-article-database/SKILL.md) — consult it before any read or write against the Articles data source.

## Scope

- IN scope: reading the queue (`Status = New`), writing summary content into the row's page, the `New → Summarized` transition, deciding which summaries warrant a wiki, authoring the wiki, and filing it under the Knowledge Base index page (creating a `###` section when no existing one fits).
- OUT of scope: Articles database identity / schema / dedup / Topics vocabulary (defer to [../notion-article-database/SKILL.md](../notion-article-database/SKILL.md)); article discovery and import (the [tech-news-fetch](../tech-news-fetch/SKILL.md) skill owns this); the `Summarized → Archived` transition (a separate cleanup concern).

## Knowledge Base Page Conventions

The Knowledge Base index page (`https://www.notion.so/34ba88827414805d8ef8ea0e95f49bd6`) is the parent of every wiki sub-page produced by this skill. The conventions below apply only when this skill edits the index or creates a child page under it.

- Top-level topic sections MUST use H3 (`###`); sub-topics MUST use H4 (`####`) directly under the parent H3.
- Wiki pages MUST be created as direct children of the Knowledge Base page; the parent relationship determines hierarchy, while the index-page link block is purely organizational.
- Page link blocks on the index page MUST use the open/close form `<page url="https://www.notion.so/<dashed-id>">Title</page>`; the self-closing form `<page url="..." />` fails block creation with a generic "Failed to create block" error.
- Page URLs MUST use the dashed UUID form (`34ea8882-7414-81c7-87b3-ea00ecbc3492`); the undashed form is unreliable for fresh pages.
- New section names on the index MUST match the Articles `Topics` vocabulary verbatim (see [../notion-article-database/SKILL.md](../notion-article-database/SKILL.md)) — do not invent variants.
- Heading-level promotion (e.g., `####` → `###`) MUST be a deliberate structural decision; routine ingestion runs MUST preserve existing heading levels unless the user has signaled otherwise.

## Workflow

The seven steps below are sequential at the queue level. Per-row work in steps 2–4 SHOULD parallelize across rows; per-wiki work in steps 6–7 SHOULD parallelize across wiki candidates.

### Step 1 — Read the queue

- MUST read the queue from the `Unread` view (`Status = New`) on the Articles data source — see [../notion-article-database/SKILL.md](../notion-article-database/SKILL.md) for the data source ID and view metadata.
- MUST capture each candidate row's `URL` (canonical, per the data source's storage rule), `Topics`, and page ID; these drive steps 2–7.

### Step 2 — Retrieve each source

- MUST fetch the source URL with `WebFetch`, requesting "main argument, key technical points, examples, conclusions, and direct quotes where notable" so the response is rich enough to summarize from.
- For HTTP 404 or wrong slug, SHOULD fetch the publication's date-indexed listing for the article's published date, find the correct slug, and retry.
- For responses that contain only navigation chrome (a recurring failure on `thenewstack.io`, `infoq.com`, and similar JS-heavy sites), MUST fall back to `WebSearch` for the article title plus author plus key terms; reconstruct from snippets plus background knowledge.
- MUST NOT fabricate content from the title alone; if both `WebFetch` and `WebSearch` fail to surface the article's actual claims, MUST leave the row at `Status = New` and report the row as skipped.

### Step 3 — Write the summary into the row

- MUST update the row via `notion-update-page` with `command: replace_content` and the summary as `new_str`.
- The summary MUST follow this format:
  - Lead with a callout linking back to the source: `> 🔗 Source: [Site name — Article title](URL)`.
  - 1–12 H2 (`##`) sections, each with 2–5 paragraphs of prose. Section count scales with article substance — short notes use 1–2; deep technical posts use 8–12.
  - SHOULD use the source's own framing where it aids comprehension; quote sparingly with `**bold**` or block quotes for genuinely notable lines.
  - MUST NOT be bullet-only; bullets MAY appear inside sections for enumerations.

### Step 4 — Transition the row to Summarized

- MUST set the row's `Status` to `Summarized` via a separate `notion-update-page` call with `command: update_properties` and `{"Status": "Summarized"}`.
- Status-transition ownership rules (only `New → Summarized` is permitted from this skill) defer to [../notion-article-database/SKILL.md](../notion-article-database/SKILL.md).

### Step 5 — Decide whether each summary warrants a wiki page

- SHOULD build a wiki page when (a) a reader who has not seen the source could benefit from a foundational reference on the underlying concept, (b) the article exemplifies a general pattern or methodology worth capturing independently, or (c) future articles are likely to touch the same topic.
- SHOULD NOT build a wiki for joke/quirk notes, single-paragraph observations, anecdotes, or pure news items with no reusable concept.
- When the call is genuinely ambiguous, SHOULD lean toward creating the wiki — wikis are inexpensive to delete later.

### Step 6 — Construct each wiki page

- The page title MUST name the foundational concept (e.g., `Distributed Tracing for AI Agents`), not the source article (`Jaeger v2 article summary`); the page MUST remain useful even if the source disappears.
- SHOULD research foundational context with `WebSearch` when the source assumes background the reader may lack; the wiki MUST explain what the concept is, why it matters, the key trade-offs, and concrete guidance — not just rephrase the source.
- The wiki SHOULD follow this structure:
  - Opening callout (`> 💡 …`) — one-line statement of what the page covers.
  - Foundational context — define jargon on first use.
  - Substance — trade-offs, layers, components, principles, decision rules; tables for comparisons.
  - Practical guidance — when to use, when not to use, checklists, common objections answered.
  - References — original source link plus 2–4 authoritative background links.
- SHOULD aim for 600–1,500 words; prefer clear, declarative prose over corporate hedging; use code blocks for snippets and emoji-prefixed callouts (`> 💡`, `> ⚠️`) sparingly.
- MUST pass an emoji `icon` to `notion-create-pages` that fits the topic (🧠 prompting, 🔭 observability, ⚡ performance, 🏆 testing, ☁️ cloud).

### Step 7 — File each wiki under the Knowledge Base

- MUST create the wiki via `notion-create-pages` with `parent.type: "page_id"` and the Knowledge Base page ID; this anchors the parent relationship described in [Knowledge Base Page Conventions](#knowledge-base-page-conventions).
- MUST match the article's `Topics` value (or its closest fit) to an existing `###` heading on the index. When no existing section fits, MUST add a new `###` heading whose name matches the Articles `Topics` vocabulary verbatim.
- MUST insert a `<page>` link block under the chosen section via `notion-update-page` `command: update_content`, anchoring `old_str` on the section heading:
  ```
  old_str: "### <Section Name>"
  new_str: "### <Section Name>\n\n<page url=\"https://www.notion.so/<dashed-id>\">Title</page>"
  ```
- Page link block format and URL form MUST follow [Knowledge Base Page Conventions](#knowledge-base-page-conventions); link block creation will silently fail otherwise.

## Final Report

After all rows have been processed, MUST post a concise summary the user can scan in seconds:

- **Summarized** — bulleted list of every row touched, each link clickable.
- **Wikis created** — bulleted list grouped by topic section, each link clickable, with `(NEW)` flag on any sections added.
- **Skipped** — bulleted list with the reason for each (e.g., source 404, fetch returned only chrome, deliberately too thin for a wiki).
- **Structural notes** — any heading-level changes or new sections added to the Knowledge Base index page.

## Gotchas

- The Knowledge Base index page MUST NOT be edited via `replace_content` without first inventorying every existing `<database>` and `<page>` reference into `new_str`; otherwise the call silently proposes to delete those child blocks. Use `update_content` with targeted `old_str → new_str` operations instead.
- WebFetch frequently returns site chrome instead of article body for major tech-news sites; SHOULD inspect the response body and fall back to `WebSearch` rather than summarizing from chrome.
- Per-row work (steps 2–4) parallelizes cleanly across rows because the Notion MCP and WebFetch tolerate concurrent calls; spawning all rows in parallel keeps queues of 10+ articles tractable.
