---
description: >
  Scheduled workflow that polls Jira for new bugs and creates GitHub issues
  to trigger the automated verification and fix pipeline.

name: Poll Jira for New Bugs

on:
  schedule:
    - cron: '0 * * * *'  # every hour
  workflow_dispatch:       # allow manual trigger

permissions:
  contents: read
  issues: read

engine: copilot

tools:
  github:
    toolsets: [issues]
  bash:
    - curl:*

safe-outputs:
  create-issue:
    max: 20

timeout-minutes: 10
---

# Jira Bug Poller

Run the `jira-poller` agent to query Jira for new bugs and create GitHub issues for each one.

Use the JQL filter:
```
reporter = "Connector for Salesforce & Jira"
AND issuetype = Bug
AND priority in (P2, P3, P4)
AND labels != "sent-to-github"
AND status not in (Done, Closed, Resolved)
ORDER BY created ASC
```

For each result: create a GitHub issue with label `needs-verification` and stamp the Jira ticket with `sent-to-github`.
