# Covered Topics

The project's editorial scope. **The canonical data lives in the structured-data file [topics.jsonl](./topics.jsonl).** This document is the schema/procedure companion — it explains how to read and modify the data, but does not duplicate the list itself.

## Why JSONL

JSON Lines is chosen for the data file because:

- It matches the format already used by [publisher-blocklist.jsonl](./publisher-blocklist.jsonl) — one append-friendly format for the project's hand-curated reference data is easier to remember and easier to script against than a mix.
- Each topic is one line: trivial to append, trivial to diff, and `git blame` resolves to the line that introduced or last touched a single topic.
- Strict JSON parsing on each record — no ambiguity about quoting, multi-line strings, or whitespace, which is the typical failure mode of YAML when records grow.
- Forward-compatible: readers ignore unknown keys, so adding optional fields is non-breaking.

The trade-off vs. YAML is that JSONL forbids comments. Schema documentation therefore lives in this `.md` file rather than inline next to the data; the two files MUST be kept in sync.

## Schema (v1)

Each line in [topics.jsonl](./topics.jsonl) is a single JSON object representing one covered topic. Lines are ordered; order is preserved when rendering reports and indexes.

Required fields:

- `name` — the exact Notion `Topics` multi-select option string this topic maps to, AND the human-readable name used in run reports and documentation. The two roles are unified: the JSONL entry's `name` IS the Notion option name, byte-for-byte. MUST be present in the Notion-side vocabulary at write time (see [../../notion-article-database/SKILL.md → Topics Vocabulary](../../notion-article-database/SKILL.md#topics-vocabulary)); writes referencing an unknown option fail. Renaming a `name` requires (a) a repo-wide grep + sweep of every caller that references the topic by name, AND (b) the controlled-vocabulary rename procedure on the Notion side; treat it as a deliberate refactor, not an in-place edit.
- `keywords` — short JSON array (MUST have at least one entry) spelling out what falls under this topic. **This is the primary signal AI agents use to decide which Notion `Topics` tags to apply to a given article.** Each keyword is a subtopic, alternate phrasing, or core concept; the agent reads the article's subject and the keyword set and decides — by judgment, not literal substring matching — whether the article belongs under this topic. For self-evident topics, the keyword set MAY be as small as the topic's own constituent terms (e.g., `JavaScript/TypeScript` → `["JavaScript", "TypeScript"]`, `React` → `["React"]`). For umbrella topics, include the subdomains that aren't obvious from the name (e.g., `Agentic/AI Coding` → `["Context Engineering", "Harness Engineering", "Agent Skills"]`). Keep the array short (typically 1–4 entries); avoid exhaustive lists of every related word. MUST NOT be used as a literal substring-search gate — see [Application Rules](#application-rules).

Optional fields the schema is forward-compatible with (none defined yet, but reserved as a deliberate extensibility surface):

- `aliases` — JSON array of alternate `name`-equivalent strings that should resolve to this topic when other surfaces (CLI input, run reports, the user's prompt) refer to the topic by an alternate label.
- `description` — long-form prose for documentation and run-report tooltips.
- `seed_publishers` — JSON array of RSS/Atom feed URLs to seed discovery for this topic.
- `status` — one of `"active"` / `"paused"` / `"deprecated"`; lets the list document intent without removing entries.

Add any of the above by editing the JSONL; readers MUST ignore fields they don't recognize.

The schema is currently unversioned at the file level — JSONL has no clean place for top-level metadata, and adding fields is non-breaking. If a true breaking schema change ever becomes necessary (e.g., renaming a required field), introduce a discriminated meta record at the top of the file (`{"$type":"meta","schema_version":2}`) and migrate readers in the same change.

## Application Rules

When picking the `Topics` value for a Notion entry from the data in [topics.jsonl](./topics.jsonl):

- MUST pick the option(s) matching the *post itself*, not the publisher's primary topic — e.g., a Vercel Blog post about edge functions goes under `Serverless`, not `Next.js`.
- MAY assign multiple options when the post legitimately spans topics; SHOULD NOT exceed three options per entry.
- SHOULD use `General` only as a last resort, when the post is on-scope but does not fit any specific option. MUST NOT use `General` as a synonym for "I didn't classify this".

When deciding *which* topics apply to an article, agents MUST consult each topic's `keywords` array — that array is the canonical spec for what falls under the topic. The agent reads the article's subject and the keyword set, then decides by **judgment about subject overlap**, not by literal substring matching:

- For each topic, ask: does the article's actual subject fall under any of `[name, …keywords]`? If yes, apply the tag.
- A post about "context engineering for Claude agents" belongs under `Agentic/AI Coding` because `Context Engineering` is in that topic's `keywords` — the agent recognizes context engineering as a subdomain even though the post may not contain the literal word "agentic".
- A post about React Hooks belongs under `React` because Hooks is part of React (the keyword). The agent does not need every React API name listed in `keywords` — judgment fills the gap.
- Keywords MUST NOT be lowercased and substring-matched against the article body. That pattern produces false positives (a `React` keyword hit in an unrelated changelog footer is a typical failure). Topic assignment is always a judgment about the article's subject.
- When the user adjusts the keyword set on a topic, the agent's tag-application behavior changes accordingly — that's the intended feedback loop. Adding a keyword to a topic broadens what the agent treats as in scope; removing one narrows it.

The Notion `Topics` vocabulary may include cross-cutting options (`Software Architecture`, `General`) that don't have a 1:1 entry in the JSONL. Those exist for tagging *within* an in-scope post and are governed by [../../notion-article-database/SKILL.md → Topics Vocabulary](../../notion-article-database/SKILL.md#topics-vocabulary).

## Adjusting the List

Editing the editorial scope is a deliberate decision and MUST NOT happen as a side effect of routine fetch work. To make a change:

- **Add a topic.** Append a new line to [topics.jsonl](./topics.jsonl) with `name` and `keywords` (at least one entry). If `name` does not already exist in the Notion `Topics` vocabulary, add the new option per [../../notion-article-database/SKILL.md → Topics Vocabulary](../../notion-article-database/SKILL.md#topics-vocabulary) in the same change.
- **Remove a topic.** Delete the line. Existing Notion entries tagged with the corresponding option are NOT auto-archived; archival is a separate decision the user owns. Consider adding `"status":"deprecated"` to the line instead if the topic should remain audit-visible without being scoped in.
- **Split a topic.** Replace the single line with two new lines, each with its own `name` and `keywords`. The two new `name` values MUST exist in the Notion `Topics` vocabulary at the time of the next write — add them per the controlled-vocabulary procedure in the same change.
- **Rename a topic.** Renaming `name` requires (a) a repo-wide grep + sweep of every caller that references the topic by name, AND (b) the controlled-vocabulary rename procedure in the Notion-side skill so the Notion option moves in lockstep. Updating `keywords` is non-breaking — adjust freely; this is the intended feedback loop for tuning what the agent treats as in scope per topic.

After any change, MUST grep the repo for stale inline topic listings (a recurring failure mode); the only place topics should be enumerated is `topics.jsonl`. The `name` strings are good grep targets.
