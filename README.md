# dev-daily

> A personal AI-agentic pipeline for staying current in software engineering.

`dev-daily` turns a stream of recent tech articles into a curated, queryable knowledge base in Notion. It is operated entirely through AI agents — there is no application code in this repository, only the [skills](https://agentskills.io/) that instruct agents how to run each stage of the pipeline.

## Pipeline

```
fetch ──▶ summarize ──▶ synthesize
  │           │              │
  │           │              └─ promote summaries that warrant foundational
  │           │                 treatment into wiki-style reference pages
  │           │                 under the Knowledge Base index
  │           │
  │           └─ write a structured per-article summary into the
  │              article's row and mark it Summarized
  │
  └─ discover and import recent articles within a publish-date range
     into Knowledge Base › Articles (Status = New)
```

Each stage is owned by one skill and routes work through a single Notion database (`Knowledge Base › Articles`) plus a monthly run-log page (`Knowledge Base › Updates`).

## Coverage

- **Topics** — JavaScript/TypeScript, React/Next.js, React Native/Expo, HTML/CSS, Flutter/Dart, web technology, UI/UX design, serverless, agentic / context / harness engineering, SWE career growth. The canonical list lives in [.agents/skills/tech-news-fetch/references/topics.jsonl](.agents/skills/tech-news-fetch/references/topics.jsonl); rules in [covered-topics.md](.agents/skills/tech-news-fetch/references/covered-topics.md).
- **Languages** — English and Japanese (日本語). Every fetch run targets both unless the user explicitly scopes one.

## Usage

The pipeline is driven by prompts to an agent that has access to this repository's skills and to the project's Notion workspace. Each prompt below routes deterministically to one skill via the index in [AGENTS.md](AGENTS.md).

**Stage 1 — fetch** (skill: [tech-news-fetch](.agents/skills/tech-news-fetch/SKILL.md)). Requires an explicit publish-date range; the skill asks the user when none is given.

```text
> Fetch articles published in the last 7 days.
> Fetch articles published between 2026-02-13 and 2026-03-28.
> Fetch articles published yesterday, Japanese sources only.
```

**Stages 2 + 3 — summarize and synthesize** (skill: [knowledge-ingestion](.agents/skills/knowledge-ingestion/SKILL.md)). Processes every row with `Status = New` end-to-end and appends a run log to the current month's *Updates* page.

```text
> Process the Knowledge Base › Articles queue.
```

The agent loads the matching skill, follows the procedure and reference files it points to, and executes the stage end-to-end.
