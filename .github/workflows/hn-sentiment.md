---
name: HN Sentiment Analysis

# Trigger - run when a user posts "/hn-sentiment <url>" as an issue or PR comment.
# The `slash_command:` trigger is gh-aw's built-in slash-command idiom; it
# automatically filters to comments that start with the command name.
on:
  slash_command:
    name: hn-sentiment

# Permissions - the agent only reads context; writing the reply is handled by
# the safe-outputs job (add-comment) with its own scoped permissions.
permissions:
  contents: read
  issues: read
  pull-requests: read

# Network allowlist - the agent must reach the public Hacker News API.
network:
  allowed:
    - hacker-news.firebaseio.com

# Tools - allow the agent to fetch HTTP(S) URLs (the HN Firebase API).
tools:
  web-fetch:

# Safe outputs - post exactly one comment back into the triggering thread.
safe-outputs:
  add-comment:
    max: 1
  # actions:
  # activation-comments:
  # add-comment:
  # add-labels:
  # add-reviewer:
  # allowed-github-references:
  # assign-milestone:
  # assign-to-agent:
  # assign-to-user:
  # autofix-code-scanning-alert:
  # call-workflow:
  # close-discussion:
  # close-issue:
  # close-pull-request:
  # concurrency-group:
  # create-agent-session:
  # create-agent-task:
  # create-code-scanning-alert:
  # create-discussion:
  # create-project:
  # create-project-status-update:
  # create-pull-request:
  # create-pull-request-review-comment:
  # dispatch-workflow:
  # dispatch_repository:
  # environment:
  # failure-issue-repo:
  # footer:
  # group-reports:
  # hide-comment:
  # id-token:
  # link-sub-issue:
  # mark-pull-request-as-ready-for-review:
  # max-bot-mentions:
  # max-patch-files:
  # mentions:
  # merge-pull-request:
  # missing-data:
  # missing-tool:
  # noop:
  # push-to-pull-request-branch:
  # remove-labels:
  # reply-to-pull-request-review-comment:
  # report-failure-as-issue:
  # report-incomplete:
  # resolve-pull-request-review-thread:
  # scripts:
  # set-issue-field:
  # set-issue-type:
  # steps:
  # submit-pull-request-review:
  # threat-detection:
  # unassign-from-user:
  # update-discussion:
  # update-issue:
  # update-project:
  # update-pull-request:
  # update-release:
  # upload-artifact:
  # upload-asset:

---

# HN Sentiment Analysis Slash Command

You are responding to a slash command posted as an issue or pull-request comment
in this repository. The triggering comment is of one of these forms:

**Single thread:**

```
/hn-sentiment <hacker-news-story-url>
```

**Compare two threads:**

```
/hn-sentiment <hacker-news-story-url-A> <hacker-news-story-url-B>
```

For example:

```
/hn-sentiment https://news.ycombinator.com/item?id=40942509
/hn-sentiment https://news.ycombinator.com/item?id=40942509 https://news.ycombinator.com/item?id=40000000
```

Use the comment body that this workflow was activated with (already provided to
you in the activation context) as the source of the URL argument(s).

## Instructions

1. **Parse the comment.** Extract **all** Hacker News story URLs that follow
   the `/hn-sentiment` command (separated by whitespace). For each URL extract
   the numeric item ID from the `id=` query parameter.
   - If **zero** URLs are present, or any URL is not a valid
     `https://news.ycombinator.com/item?id=<number>` URL, skip to step 7 and
     reply with a helpful error message explaining both single and two-URL
     usage.
   - If **one** URL is provided, run the **Single-thread analysis** flow
     (steps 2–6 below).
   - If **two** URLs are provided, run the **Comparison analysis** flow
     (step 6B below) instead of the single-thread report.
   - If **three or more** URLs are provided, reply with an error explaining
     that at most two URLs are supported.

### Single-thread analysis (one URL)

2. **Fetch the story metadata** from the Hacker News Firebase API:
   `https://hacker-news.firebaseio.com/v0/item/<ID>.json`. Capture the story
   `title`, `by` (author), `url`, and the array of top-level comment IDs in `kids`.
   If the response is empty / null, reply with `Item not found`.
3. **Fetch up to 50 top-level comments.** For each ID in `kids` (cap at 50),
   GET `https://hacker-news.firebaseio.com/v0/item/<comment_id>.json`. Skip
   entries that are `deleted` or `dead`, or that have no `text`. Strip basic HTML
   tags (`<p>`, `<i>`, `<a …>`, `&#x27;`, `&quot;`, `&gt;`, `&lt;`, `&amp;`) from
   the comment body before analysis.
4. **Classify sentiment.** For every fetched comment, classify it as exactly one
   of `Positive`, `Negative`, or `Neutral` based on the comment's tone. Track the
   counts for each category and keep an excerpt (≤ 240 characters, ending on a
   word boundary) for the top 3 most clearly positive and the top 3 most clearly
   negative comments. Prefer substantive comments; avoid one-word reactions.
5. **Compute a word-frequency table** over the cleaned comment text:
   - Lowercase the text and tokenise on non-letter characters.
   - Drop tokens shorter than 4 characters.
   - Drop common English stop words: `the, and, that, this, with, from, your,
     they, them, have, will, would, could, should, what, when, where, which,
     about, just, like, into, more, some, than, then, also, even, much, very,
     because, there, their, these, those, were, been, being, only, over, into,
     such, only, here, still, every, while, between, after, before, again`.
   - Count occurrences and keep the **top 10** by frequency. Break ties
     alphabetically.
6. **Build the Markdown report** using exactly this structure:

   ```markdown
   ## 🔍 Hacker News Sentiment Analysis

   **Story:** <story title> ([HN thread](<original HN URL>))

   | Sentiment   | Count | Percentage |
   |-------------|-------|------------|
   | 😊 Positive | <n>   | <p>%       |
   | 😐 Neutral  | <n>   | <p>%       |
   | 😠 Negative | <n>   | <p>%       |

   **Overall: <Mostly Positive | Mixed | Mostly Negative>** (based on <N> comments analysed)

   ### Top Positive Comments
   > "<excerpt 1>"
   > "<excerpt 2>"
   > "<excerpt 3>"

   ### Top Negative Comments
   > "<excerpt 1>"
   > "<excerpt 2>"
   > "<excerpt 3>"

   ### Top Terms in the Discussion
   | Term | Count |
   |------|-------|
   | <term 1> | <n> |
   | <term 2> | <n> |
   …
   ```

   Round percentages to whole numbers so they sum to 100. If a section has fewer
   than 3 qualifying comments, include only what you have. If no comments could
   be fetched, state that explicitly instead of fabricating excerpts. If
   word-frequency data is unavailable, omit the "Top Terms" section.

### Comparison analysis (two URLs)

6B. **Run steps 2–5 for both stories independently**, then build a single
    comparison report using exactly this structure:

   ```markdown
   ## 🆚 Hacker News Sentiment Comparison

   | | Thread A | Thread B |
   |---|---|---|
   | **Story** | [<title A>](<url A>) | [<title B>](<url B>) |
   | **Comments analysed** | <N_A> | <N_B> |
   | 😊 Positive | <n_A> (<p_A>%) | <n_B> (<p_B>%) |
   | 😐 Neutral  | <n_A> (<p_A>%) | <n_B> (<p_B>%) |
   | 😠 Negative | <n_A> (<p_A>%) | <n_B> (<p_B>%) |
   | **Overall** | <Mostly Positive / Mixed / Mostly Negative> | <…> |

   ### Headline difference
   One sentence (≤ 200 chars) stating the most striking difference between
   the two threads (e.g. "Thread A is much more positive (62 % vs 31 %).").

   ### Top Terms Compared
   | Rank | Thread A | Thread B |
   |------|----------|----------|
   | 1 | <term A1> (<n>) | <term B1> (<n>) |
   | 2 | <term A2> (<n>) | <term B2> (<n>) |
   …
   ```

   Show the top 5 terms per thread. Omit the Top Terms table entirely if both
   sides have no usable word data.

### Reply

7. **Reply in the same thread** by emitting exactly one `add-comment` safe
   output whose body is the Markdown report from step 6 or 6B (or the error
   message from step 1 / 2). Do not create issues, PRs, or any other safe
   outputs.

## Guardrails

- Only call the `hacker-news.firebaseio.com` host. Do not fetch other URLs.
- Do not include any user secrets, tokens, or workflow internals in the reply.
- Quote comment excerpts verbatim — do not paraphrase, summarise, or translate.
- Be concise: the entire reply must fit within a single GitHub comment.

## Notes

- Run `gh aw compile` after editing this file to regenerate the `.lock.yml`.
- The `slash_command:` trigger only fires once the lock file is on the
  repository's default branch (usually `main`).
