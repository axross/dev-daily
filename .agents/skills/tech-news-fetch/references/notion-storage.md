# Notion Storage

Apply these rules during step 4 of the fetch pipeline (see [SKILL.md](../SKILL.md)) when writing each surviving entry into the Articles database in Notion.

The database's identity, property schema, URL canonicalization, dedup contract, Status lifecycle, and Topics vocabulary are governed by [../../notion-article-database/SKILL.md](../../notion-article-database/SKILL.md). Load that skill before any read or write to the Articles database. The covered-topic-to-Notion-option mapping data lives in [topics.jsonl](./topics.jsonl); the schema and application rules are documented in the companion [covered-topics.md](./covered-topics.md). The rules below are the **fetch-pipeline overlays** — which fields the fetch path is responsible for setting, and the per-run reporting requirements specific to this pipeline.

## Topic Mapping

The list of in-scope topics is data, not prose: parse [topics.jsonl](./topics.jsonl) and use each entry's `name` directly as the Notion `Topics` option string (the two are unified — there is no separate mapping field). The application rules (match the post not the publisher, max three options, `General` only as a last resort) are documented in [covered-topics.md → Application Rules](./covered-topics.md#application-rules). MUST consult those files when picking the `Topics` value for a new entry; MUST NOT duplicate, paraphrase, or hard-code the topic list here.

When a new Notion `Topics` option is needed, add it per [../../notion-article-database/SKILL.md → Topics Vocabulary](../../notion-article-database/SKILL.md#topics-vocabulary) AND add a corresponding entry to [topics.jsonl](./topics.jsonl) in the same change, so the fetch pipeline's controlled vocabulary stays aligned with the database.

## Writing the Entry

For each surviving entry that passes [discovery-heuristics.md](./discovery-heuristics.md) and the database's [dedup contract](../../notion-article-database/SKILL.md#dedup-contract):

- MUST create one Notion page per entry, populating Title, URL, Published At, Status, and Topics in the same call. Multi-call writes leave half-populated rows visible in the `Recent` and `Unread` views during the gap.
- MUST set `Status="New"` per the lifecycle ownership rule in [../../notion-article-database/SKILL.md → Status Lifecycle](../../notion-article-database/SKILL.md#status-lifecycle); the fetch pipeline does not own any other Status transition.
- SHOULD leave the page body empty — downstream summarization owns the body, and writing it from the fetch path creates merge conflicts when the summarizer later edits the page.

## Run Reporting

- SHOULD report at run end: counts of entries written, skipped (duplicate), and rejected (bad metadata, off-topic, or low-signal per [discovery-heuristics.md](./discovery-heuristics.md)), broken down by topic — including topics that returned zero in-window hits, so the user can distinguish "thin window" from "no sources surveyed for that topic".
- SHOULD also report a one-line **language breakdown** (count of English vs Japanese entries written). Per the bilingual coverage mandate in [discovery-heuristics.md → Language Coverage](./discovery-heuristics.md#language-coverage), a run that lands zero entries in either language is a partial run; the report MUST flag this so the user can spot a discovery-surface regression.
- SHOULD flag a **discovery anomaly** when no candidate from a previously-unseen host was surfaced this run, per [discovery-heuristics.md → Per-Run Exploration Mandate](./discovery-heuristics.md#per-run-exploration-mandate).
- SHOULD include the signal-clear count for each accepted methodology post (e.g., "`joshwcomeau.com` — 4 of 5 signals cleared, accepted") per [discovery-heuristics.md → Content Depth Signals](./discovery-heuristics.md#content-depth-signals).
- SHOULD list any blocklist appends made during the run (host + one-line reason) per [discovery-heuristics.md → Publisher Blocklist](./discovery-heuristics.md#publisher-blocklist).
- MUST NOT report success when any write fails partway through a batch — list the failures *and* the entries that did land, so the user can decide whether to retry.

## Verification

- SHOULD spot-check the first written entry by querying the `Recent` view (sorted by `Published At` desc) after the run; if the entry is not visible, the data source ID was likely wrong.
- MUST verify a newly added `Topics` option exists on the data source before the first write that references it, and confirm it appears on the written page after — silent fallback to "no Topics" is otherwise easy to miss.
