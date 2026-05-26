---
name: Dev.to Daily Digest

# Trigger - daily on weekdays plus manual dispatch.
on:
  schedule: daily on weekdays
  workflow_dispatch:

permissions:
  contents: read
  issues: read

# Network allowlist - public Dev.to REST API.
network:
  allowed:
    - dev.to

tools:
  web-fetch:

safe-outputs:
  create-issue:
    max: 1
---

# Dev.to Daily Digest

Create a GitHub issue summarising today's top articles on dev.to, grouped by
their primary tag, with a "Top Article of the Day" highlighted.

## Instructions

1. **Fetch the top articles** from
   `https://dev.to/api/articles?top=1&per_page=30`. The `top=1` parameter
   returns the most popular articles from the past day, ordered by
   reaction count. Take **all** items returned (up to 30).
2. For each article extract:
   - `title`, `url`, `description` (often the subtitle),
   - `public_reactions_count`, `comments_count`,
   - `user.name`, `published_at`,
   - `tag_list` (an array of strings).
   Skip articles where `title` or `url` is missing.
3. **Assign a primary tag** to each article: pick the **first** tag in
   `tag_list`. If `tag_list` is empty, use the literal tag `general`.
4. **Identify the "Top Article of the Day"** as the article with the highest
   `public_reactions_count`. Break ties by preferring the one with more
   `comments_count`.
5. **Build the Markdown report** with this exact structure (substitute real
   values; omit empty tag sections):

   ```markdown
   # 💻 Dev.to Daily Digest – <YYYY-MM-DD>

   Top developer articles from https://dev.to over the past 24 hours, grouped by tag.

   ## 🏆 Top Article of the Day

   **[<title>](<url>)** — ❤️ <public_reactions_count> · 💬 <comments_count> · by <user.name>
   > <description, trimmed to ≤ 240 chars on a word boundary>

   ---

   ## #<primary_tag> (<count> articles)

   - **[<title>](<url>)** — ❤️ <reactions> · 💬 <comments> · by <user.name>
   - …
   ```

   - Sort tag sections by article count descending; within a section, sort by
     `public_reactions_count` descending.
   - Render every title as a Markdown link to the article URL.
   - End the issue body with:
     `_Generated from https://dev.to · <N> articles analysed_` where `<N>` is
     the actual count.
6. **Create exactly one issue** via the `create-issue` safe output with title
   `Dev.to Digest – <YYYY-MM-DD>` (today's date, UTC) and the body from step 5.

## Guardrails

- Only fetch URLs on `dev.to`. Do not follow article URLs themselves.
- Quote article titles and descriptions verbatim - no translation or rewriting.
- If the API returns fewer articles than expected, work with what you have.
- Do not emit any safe output other than the single `create-issue`.

## Notes

- Run `gh aw compile` after editing.
- Demonstrates the workshop's "Connect to other APIs" extension idea by
  swapping Hacker News for dev.to.
