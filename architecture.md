### Architecture Overview

```
User triggers bug ──> Flask error handler ──> Copilot SDK creates GitHub issue
                                                       │
                                           label: needs-verification
                                                       │
                                              Agentic Workflow runs
                                           (reproduces bug)
                                                       │
                                       ┌───────────────┴───────────────┐
                                 Reproducible?                   Not reproducible?
                                       │                               │
                                  verified-bug                  not-reproducible
                                       │                         close issue
                                       │
                           Assign bug-solver agent
                                       │
                              Agent opens a PR
                           (fix + tests + screenshots)
                                       │
                              Code review agent
                                       │
                              Merge PR ──> Deploy workflow
                                       │
                              Bug is fixed in production
```