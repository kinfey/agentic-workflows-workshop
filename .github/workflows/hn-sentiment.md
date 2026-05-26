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
in this repository. The triggering comment was a message of the form:

```
/hn-sentiment <hacker-news-story-url>
```

For example:

```
/hn-sentiment https://news.ycombinator.com/item?id=40942509
```

Use the comment body that this workflow was activated with (already provided to
you in the activation context) as the source of the URL argument.

## Instructions

1. **Parse the comment.** Extract the Hacker News story URL that follows the
   `/hn-sentiment` command. Then extract the numeric item ID from the URL's
   `id=` query parameter (e.g. `40942509`). If no URL is provided, or it is not a
   valid `https://news.ycombinator.com/item?id=<number>` URL, skip to step 6 and
   reply with a helpful error message explaining the expected format.
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
5. **Build the Markdown report** using exactly this structure:

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
   ```

   Round percentages to whole numbers so they sum to 100. If a section has fewer
   than 3 qualifying comments, include only what you have. If no comments could
   be fetched, state that explicitly instead of fabricating excerpts.
6. **Reply in the same thread** by emitting exactly one `add-comment` safe
   output whose body is the Markdown report from step 5 (or the error message
   from step 1 / 2). Do not create issues, PRs, or any other safe outputs.

## Guardrails

- Only call the `hacker-news.firebaseio.com` host. Do not fetch other URLs.
- Do not include any user secrets, tokens, or workflow internals in the reply.
- Quote comment excerpts verbatim — do not paraphrase, summarise, or translate.
- Be concise: the entire reply must fit within a single GitHub comment.

## Notes

- Run `gh aw compile` after editing this file to regenerate the `.lock.yml`.
- The `command:` trigger only fires once the lock file is on the repository's
  default branch (usually `main`).
