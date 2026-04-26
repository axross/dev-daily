# Time-Range Handling

Apply these rules at the start of every fetch run (see [SKILL.md](../SKILL.md)) to convert the user's prompt into a closed UTC publish-date interval, or to ask the user when the prompt does not give one.

## Parsing User Input

Recognize these forms; anything else MUST trigger the *ask the user* path:

- **Trailing window** — phrases like "today", "yesterday", "in this 24 hours", "in the last 7 days", "in the past month", "this week". Resolve `end` to the project's current date, then resolve `start` so the closed inclusive `[start, end]` interval covers *exactly* the named span — never one day more:
  - "today" / "in this 24 hours" → `[end, end]` (one day inclusive).
  - "yesterday" → `[end - 1, end - 1]` (one day inclusive, shifted back).
  - "in the last N days" / "past N days" / "last N days" → `[end - (N - 1), end]`. So "last 7 days" covers 7 calendar days inclusive (`end - 6 .. end`), not 8.
  - "this week" → `[end - weekday_index, end]` where `weekday_index` is days since the user's week start (default Monday → Sunday = 6).
  - "this month" / "in the past month" — month-to-date: `[first_day_of_current_month, end]`.
- **Offset window** — "from 5 days ago to 2 days ago", "between 3 and 7 days ago". Resolve both endpoints relative to the current date.
- **Absolute range** — "2026-02-13 - 2026-03-28", "Feb 13 to Mar 28, 2026", "from 2026-02-13 through 2026-03-28". Parse both endpoints as ISO-8601 dates.
- **Single absolute date** — "on 2026-04-12", "for 2026-04-12". Treat as a one-day closed interval `[2026-04-12, 2026-04-12]`.
- **Open-ended absolute** — "since 2026-02-13", "after 2026-02-13". Resolve the missing end to the current date.

Application rules:

- MUST anchor relative phrases to the date already present in the conversation context (e.g., the harness-supplied `currentDate`). If no current date is known and the prompt is relative, MUST ask rather than assume the host clock.
- MUST NOT silently coerce ambiguous inputs — "recently", "lately", "the latest" carry no defined window and MUST trigger the ask path.
- SHOULD echo the parsed interval back to the user in the same message that begins the fetch ("Fetching articles published 2026-04-20 → 2026-04-26 inclusive — 7 calendar days…") so a misparse or off-by-one is caught before any Notion write.

## When to Ask the User

The skill MUST ask the user for a time range when any of the following hold:

- The prompt contains no temporal phrasing at all.
- The prompt contains a temporal phrase the rules above do not recognize.
- The prompt gives only one endpoint and the missing endpoint cannot be inferred (e.g., "until last Tuesday" with no start).
- The parsed interval is invalid — `start > end`, or `end` is past the current date.

Use this question template — phrased once, not chained across turns:

> What time range should I cover? You can give a window ("last 7 days", "this 24 hours"), an offset ("from 5 days ago to 2 days ago"), or absolute dates ("2026-02-13 - 2026-03-28").

- MUST NOT proceed to fetch with a guessed range. Block on the user's reply.
- MUST NOT default to a built-in window (e.g., "last 7 days") when the user is silent — defaults silently mis-target the digest and dedup logic.

## Normalization

- MUST emit a closed interval `[start, end]` in UTC, both endpoints inclusive at day granularity. A request for "today" resolves to `[YYYY-MM-DD, YYYY-MM-DD]`, a single calendar day inclusive on both ends.
- MUST treat day boundaries as `00:00:00Z` for `start` and `23:59:59Z` for `end` when applying the filter to per-entry timestamps from feeds (which usually carry full datetimes).
- SHOULD record the resolved interval alongside each entry in pipeline output so downstream stages can audit which run a row came from.
- MUST NOT widen the interval to make the result non-empty when the feed returns nothing — an empty result is a valid answer and MUST be reported as such.

## Edge Cases

- **Future endpoint** — the user gives `end` after the current date. SHOULD clamp `end` to the current date and note the clamp in the echo-back; MUST NOT silently clamp without telling the user.
- **Range > 90 days** — accept it, but SHOULD warn the user that long ranges are slow and high-volume, and offer to narrow before proceeding.
- **Time-zone-tagged inputs** — if the user gives a range in a non-UTC time zone (e.g., "from JST 2026-04-20"), MUST convert to UTC before applying the filter; MUST NOT mix time zones across endpoints.
- **DST and leap-second pedantry** — out of scope; day-granularity gating absorbs both.
- **Empty result** — report "no entries in range across [N] sources" and stop; MUST NOT widen, retry, or fall back to a different range without the user's say-so.
