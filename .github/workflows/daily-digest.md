---
name: Daily Digest

# Trigger - run every weekday, plus manual dispatch
on:
  schedule: daily on weekdays
  workflow_dispatch:

# Permissions - read access to issues and PRs in the repository
permissions:
  contents: read
  issues: read
  pull-requests: read

# Network access
network: defaults

# Outputs - the agent can create a single digest issue per run
safe-outputs:
  create-issue:
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

# Daily Digest

Every weekday, create a GitHub issue that summarises all open issues and pull
requests in this repository. Group them by label. Include the total count, the
title, the author, and how long each item has been open. Title the issue
`Daily Digest – <date>`, where `<date>` is the current date in `YYYY-MM-DD`
format.

## Instructions

1. List all **open** issues in the current repository (excluding pull requests).
2. List all **open** pull requests in the current repository.
3. Group the items by label. Items with multiple labels appear under each
   relevant label; items with no labels go under an `unlabeled` group.
4. For each item include:
   - Title (as a Markdown link to the item)
   - Author
   - How long it has been open (e.g. `3 days`, `2 weeks`)
5. At the top of the issue body, include the **total count** of open issues
   and open pull requests.
6. Create a single new issue with title `Daily Digest – <YYYY-MM-DD>` and the
   summary above as the body. Use the `create-issue` safe output to do this.
7. If there are no open issues or pull requests, still create the digest
   issue and state that the repository has none open today.

## Notes

- Run `gh aw compile` to generate the GitHub Actions workflow.
- See https://github.com/github/gh-aw/blob/v0.74.8/.github/aw/github-agentic-workflows.md
  for complete configuration options and tools documentation.
