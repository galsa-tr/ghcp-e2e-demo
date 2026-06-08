# ghcp-e2e-demo — Repo Constitution

This file is read by AI agents (bug-solver, code review, etc.) before making any changes to this repo.

## What This Repo Does

An end-to-end demo of the GitHub Copilot automated bug lifecycle:
- A Flask app with an intentional `ZeroDivisionError`
- When triggered, automatically creates a GitHub issue
- An agentic workflow verifies the bug
- A bug-solver agent fixes it and opens a PR
- CI/CD deploys the fix to Azure Container Apps

## Tech Stack

- **Language**: Python 3.10+
- **Framework**: Flask
- **Testing**: pytest
- **Infrastructure**: Azure Container Apps, Azure Container Registry
- **CI/CD**: GitHub Actions
- **Container**: Docker (linux/amd64)

## Project Structure

```
app/
  main.py           # Flask app — main entry point
  error_reporter.py # Auto-creates GitHub issues on errors
  config.py         # Environment config
  requirements.txt  # Python dependencies
  templates/
    index.html      # Web UI
tests/
  test_app.py       # pytest tests
infra/
  main.bicep        # Azure infrastructure
scripts/
  setup-infra.sh    # Provision Azure resources
  deploy-app.sh     # Build & deploy Docker image
.github/
  agents/
    bug-solver.agent.md       # Bug solver agent
  workflows/
    verify-issue.md           # Verification agentic workflow
    deploy.yml                # CI/CD deploy on push to main
  copilot-instructions.md     # Code review guidelines
```

## Coding Standards

- Follow **PEP 8** for all Python code
- Use **type hints** on all function signatures
- No bare `except` clauses — always catch specific exceptions
- Use `with` statements for resource management
- No hardcoded secrets or credentials — always use environment variables
- All new functionality must have corresponding tests in `tests/`

## Running the App Locally

```bash
pip install -r app/requirements.txt
PORT=5000 python app/main.py
```

## Running Tests

```bash
pip install pytest
python -m pytest tests/ -v
```

## Environment Variables

| Variable | Purpose |
|---|---|
| `GITHUB_REPO` | Repository in `owner/repo` format |
| `GITHUB_TOKEN` | PAT with issues write permission |
| `COPILOT_GITHUB_TOKEN` | PAT for Copilot SDK |
| `PORT` | Port to run the Flask app on (default: 8080) |

## Deployment

The app is deployed to Azure Container Apps in the `ghcpe2e-rg` resource group (AI Lab subscription, West Europe).

Build always with `--platform linux/amd64` (required for Azure):
```bash
docker build --platform linux/amd64 -t <image> .
```

## PR Guidelines

- PR title must start with `fix:`, `feat:`, `chore:`, or `docs:`
- Always include `Fixes #<issue-number>` in PR body
- All PRs must have passing tests before merge
- Request review from `@galsa-tr`
