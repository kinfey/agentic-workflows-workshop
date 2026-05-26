---
name: Summarise PR Slash Command

# Trigger - `/summarise-pr <url>` posted as an issue or PR comment.
on:
  slash_command:
    name: summarise-pr

permissions:
  contents: read
  pull-requests: read
  issues: read

network: defaults

# Read-only GitHub access for fetching the target PR's metadata, files, and diff.
tools:
  github:
    toolsets: [pull_requests, repos]

safe-outputs:
  add-comment:
    max: 1
---

# /summarise-pr — AI Pull Request Summary

A user posts a comment of the form:

```
/summarise-pr https://github.com/<owner>/<repo>/pull/<number>
```

Fetch the target pull request and reply in the same thread with a concise,
review-ready summary in Markdown.

## Instructions

1. **Parse the argument.** Extract the pull request URL from the triggering
   comment. The URL must match
   `https://github.com/<owner>/<repo>/pull/<number>` (any
   trailing path or query string is ignored). Capture `owner`, `repo`, and the
   integer `number`.
   - If no URL is present, or it does not match the pattern above, skip to
     step 5 and reply with a helpful error message that shows the expected
     syntax and a usage example.
2. **Fetch PR metadata** using the available GitHub tools (`get_pull_request`
   on `<owner>/<repo>` and PR number `<number>`). Collect:
   - `title`, `state` (open / closed / merged), `draft`, `merged`,
   - `user.login` (author), `created_at`, `updated_at`, `merged_at` if any,
   - `additions`, `deletions`, `changed_files`,
   - `head.ref` and `base.ref`,
   - `labels[*].name`,
   - the first ~2000 characters of `body` (truncate on a word boundary).
   If the PR cannot be fetched (404, private, etc.), reply with an error
   message stating the PR was not accessible.
3. **Fetch the list of changed files** (`list_pull_request_files`). Keep the
   top 10 files by `changes` (= `additions` + `deletions`). For each kept file
   capture: `filename`, `status` (added / modified / removed / renamed),
   `additions`, `deletions`.
4. **Build the Markdown report** with this exact structure:

   ```markdown
   ## 🔎 Pull Request Summary

   **[<title>](<original PR url>)** · <owner>/<repo>#<number>

   | Field        | Value                                         |
   |--------------|-----------------------------------------------|
   | State        | <open / closed / merged / draft>              |
   | Author       | @<user.login>                                  |
   | Branch       | `<head.ref>` → `<base.ref>`                   |
   | Files        | <changed_files>                                |
   | Lines        | +<additions> / -<deletions>                    |
   | Labels       | <comma-separated labels, or `none`>            |
   | Created      | <created_at, YYYY-MM-DD>                        |

   ### What this PR does
   2–4 sentences in your own words, grounded in the PR title, body, and the
   top changed files. No speculation about behaviour you cannot see.

   ### Top changed files
   | File | Status | +Lines | -Lines |
   |------|--------|--------|--------|
   | `<filename>` | <status> | +<additions> | -<deletions> |
   …

   ### Suggested review focus
   - Bullet 1 (≤ 1 sentence) — what a reviewer should look at first.
   - Bullet 2 — second-priority concern (tests, security, docs, perf, etc.).
   - Bullet 3 — optional; omit if you have nothing meaningful to add.

   _Summary generated from the PR metadata and file list; not a substitute
   for reading the diff._
   ```

   - If a section would be empty (e.g. no labels), keep the row but show
     `none`.
   - Do not invent file names, line counts, or behaviour that is not present
     in the data you fetched.
5. **Reply in the same thread** by emitting exactly one `add-comment` safe
   output whose body is the Markdown from step 4 (or the error message from
   step 1 / step 2). Do not create issues, PRs, or any other safe output.

## Guardrails

- Use only the read-only GitHub tools listed in the frontmatter. Do not call
  any write APIs.
- Quote PR titles, branch names, and file paths verbatim.
- Be conservative in "What this PR does" — if information is insufficient,
  say so instead of guessing.
- The summary must fit comfortably in a single GitHub comment (< 6000 chars).

## Notes

- Run `gh aw compile` after editing.
- Demonstrates the workshop's "Add more slash commands" extension idea.
