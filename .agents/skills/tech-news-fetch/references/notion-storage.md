# Notion Storage

Apply these rules during step 4 of the fetch pipeline (see [SKILL.md](../SKILL.md)) when writing each surviving entry into the Articles database in Notion.

The database's identity, property schema, URL canonicalization, dedup contract, Status lifecycle, and Topics vocabulary are governed by [../../notion-article-database/SKILL.md](../../notion-article-database/SKILL.md). Load that skill before any read or write to the Articles database. The rules below are the **fetch-pipeline overlays** — which fields the fetch path is responsible for setting, the topic mapping from this project's covered domains to the database's controlled vocabulary, and the per-run reporting requirements specific to this pipeline.

## Topic Mapping

Map each fetched article's covered domain (per [SKILL.md](../SKILL.md)) to one or more options from the database's [Topics vocabulary](../../notion-article-database/SKILL.md#topics-vocabulary):

| Covered domain | Notion `Topics` option |
| --- | --- |
| TypeScript and JavaScript | `JavaScript/TypeScript` |
| React and Next.js | `React & Next.js` |
| React Native and Expo | `React Native & Expo` |
| HTML and CSS | `HTML & CSS` |
| Flutter and Dart | `Flutter & Dart` |
| Web Technology / web platform & standards | `Web Technology` |
| UI/UX Design | `UI/UX Design` |
| Serverless Architecture | `Serverless` |
| Agentic Coding / Context Engineering / Harness Engineering | `Agentic/AI Coding` |
| Career Enhancement in Software Engineering | `Career` |
| Software / Application Architecture | `Software Architecture` |

Application rules:

- MUST pick the option(s) matching the *post itself*, not the publisher's primary topic — e.g., a Vercel Blog post about edge functions goes under `Serverless`, not `React & Next.js`.
- MAY assign multiple options when the post legitimately spans topics; SHOULD NOT exceed three options per entry.
- SHOULD use `General` only as a last resort, when the post is on-scope but does not fit any specific option. MUST NOT use `General` as a synonym for "I didn't classify this".
- When no option fits and a new one is needed, add it per [../../notion-article-database/SKILL.md → Topics Vocabulary](../../notion-article-database/SKILL.md#topics-vocabulary), and update this mapping table in the same change so the fetch pipeline's controlled vocabulary stays aligned with the database.

## Writing the Entry

For each surviving entry that passes [discovery-heuristics.md](./discovery-heuristics.md) and the database's [dedup contract](../../notion-article-database/SKILL.md#dedup-contract):

- MUST create one Notion page per entry, populating Title, URL, Published At, Status, and Topics in the same call. Multi-call writes leave half-populated rows visible in the `Recent` and `Unread` views during the gap.
- MUST set `Status="New"` per the lifecycle ownership rule in [../../notion-article-database/SKILL.md → Status Lifecycle](../../notion-article-database/SKILL.md#status-lifecycle); the fetch pipeline does not own any other Status transition.
- SHOULD leave the page body empty — downstream summarization owns the body, and writing it from the fetch path creates merge conflicts when the summarizer later edits the page.

## Run Reporting

- SHOULD report at run end: counts of entries written, skipped (duplicate), and rejected (bad metadata, off-topic, or low-signal per [discovery-heuristics.md](./discovery-heuristics.md)), broken down by topic — including topics that returned zero in-window hits, so the user can distinguish "thin window" from "no sources surveyed for that topic".
- SHOULD include the signal-clear count for each accepted methodology post (e.g., "`joshwcomeau.com` — 4 of 5 signals cleared, accepted") per [discovery-heuristics.md → Content Depth Signals](./discovery-heuristics.md#content-depth-signals).
- MUST NOT report success when any write fails partway through a batch — list the failures *and* the entries that did land, so the user can decide whether to retry.

## Verification

- SHOULD spot-check the first written entry by querying the `Recent` view (sorted by `Published At` desc) after the run; if the entry is not visible, the data source ID was likely wrong.
- MUST verify a newly added `Topics` option exists on the data source before the first write that references it, and confirm it appears on the written page after — silent fallback to "no Topics" is otherwise easy to miss.
