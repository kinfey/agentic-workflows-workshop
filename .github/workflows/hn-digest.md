---
name: Hacker News Digest

# Trigger - daily on weekdays plus manual dispatch.
on:
  schedule: daily on weekdays
  workflow_dispatch:

# Permissions - read-only; writing the issue is done by the safe-outputs job.
permissions:
  contents: read
  issues: read

# Network allowlist - public Hacker News Firebase API only.
network:
  allowed:
    - hacker-news.firebaseio.com

tools:
  web-fetch:

safe-outputs:
  create-issue:
    max: 1
---

# Hacker News Daily Digest with Category Breakdown

Create a GitHub issue summarising the top stories on Hacker News today,
grouped by topic, with a clearly highlighted "Top Story of the Day".

## Instructions

1. **Fetch the top story IDs** from
   `https://hacker-news.firebaseio.com/v0/topstories.json`. Take the first **30**
   IDs in the order returned.
2. **Fetch each story** from
   `https://hacker-news.firebaseio.com/v0/item/<id>.json`. For each story collect:
   - `title`, `url` (fallback to `https://news.ycombinator.com/item?id=<id>` if missing),
   - `score`, `descendants` (comment count), `by` (author), `time` (unix seconds).
   Skip items where `type` is not `story`, where `deleted` or `dead` is true,
   or where there is no `title`.
3. **Classify every story** into exactly **one** of these categories based on
   the title (and URL host as a secondary signal). Use this ordered priority -
   the first matching category wins:
   - `🤖 AI / ML` — LLM, GPT, Claude, Gemini, agent, model, ML, neural, embedding, RAG, OpenAI, Anthropic, DeepMind
   - `☁️ Cloud / DevOps` — AWS, Azure, GCP, Kubernetes, Docker, Terraform, observability, SRE, serverless
   - `🦀 Programming Languages & Tools` — Rust, Go, Python, TypeScript, JavaScript, Zig, compilers, package managers, IDEs
   - `🐧 Open Source` — Linux, Git, Apache, GNU, FOSS, foundation releases, license changes
   - `🔒 Security` — CVE, exploit, vulnerability, breach, ransomware, encryption, authentication, zero-day
   - `🎨 Web / Frontend` — React, Vue, Svelte, CSS, browser, WebAssembly, design system
   - `🔧 Hardware & Systems` — chip, CPU, GPU, ARM, RISC-V, kernel, embedded, firmware
   - `💼 Business & Startups` — funding, IPO, acquisition, layoffs, Show HN companies, hiring
   - `🧪 Science & Research` — paper, study, physics, biology, math, space, climate
   - `📰 Other` — anything that does not fit cleanly above.
4. **Identify the "Top Story of the Day"** as the single story with the
   highest `score`. Break ties by preferring the one with more `descendants`.
5. **Build the Markdown report** with this exact structure (substitute real
   values; omit empty category sections):

   ```markdown
   # 📰 Hacker News Digest – <YYYY-MM-DD>

   Summary of the top 30 stories on Hacker News today, grouped by category.

   ## 🏆 Top Story of the Day

   **[<title>](<url>)** — <score> points · <descendants> comments · by <by>
   > One short sentence (≤ 200 chars) about why this story is notable, written by you.

   ---

   ## <category emoji + name> (<count> stories)

   - **[<title>](<url>)** — <score> pts · <descendants> comments · by <by>
   - …
   ```

   - List categories in the priority order from step 3, skipping any that
     have zero stories.
   - Within a category, sort stories by `score` descending.
   - Render every title as a Markdown link to the story's `url`.
   - End the issue body with a footer:
     `_Generated from https://news.ycombinator.com/ · 30 stories analysed_`.
6. **Create exactly one issue** via the `create-issue` safe output with title
   `HN Digest – <YYYY-MM-DD>` (today's date, UTC, `YYYY-MM-DD` format) and
   the body produced in step 5.

## Guardrails

- Only fetch URLs on `hacker-news.firebaseio.com`. Do not follow story URLs.
- Quote titles verbatim - do not translate or rewrite them.
- If fewer than 30 stories are available, work with what you have and note
  the actual count in the footer.
- Do not emit any safe output other than the single `create-issue`.

## Notes

- Run `gh aw compile` after editing this file.
- Inspired by the workshop's "Customise the HN digest" extension idea.
