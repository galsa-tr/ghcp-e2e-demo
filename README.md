# GitHub Copilot E2E Bug Lifecycle Demo

An end-to-end demo showing how GitHub Copilot agents can automatically **detect**, **verify**, **fix**, **review**, and **redeploy** bugs — with zero human intervention until the final merge.

## How It Works

```
User triggers bug ──> Flask error handler ──> Copilot SDK creates GitHub issue
                                                       │
                                           label: needs-verification
                                                       │
                                              Agentic Workflow runs
                                           (reproduces bug with Playwright)
                                                       │
                                       ┌───────────────┴───────────────┐
                                 Reproducible?                   Not reproducible?
                                       │                               │
                              label: verified-bug              label: not-reproducible
                                       │                         close issue
                                       │
                           Assign bug-solver agent
                          (via GitHub.com dropdown)
                                       │
                              Agent opens a PR
                           (fix + tests + screenshots)
                                       │
                           Assign code review agent
                                       │
                              Merge PR ──> Deploy workflow
                                       │
                              Bug is fixed in production
```

## Prerequisites

- **Azure CLI** (`az`) — logged in with a subscription
- **Docker** — running locally (for initial deploy)
- **GitHub CLI** (`gh`) — authenticated
- **Node.js** 20+
- **Python** 3.12+
- **gh-aw** CLI extension — `gh extension install github/gh-aw` (for compiling agentic workflows)
- A **GitHub Copilot** license with access to coding agent features

## Repository Structure

```
├── app/
│   ├── main.py              # Flask app with intentional ZeroDivisionError
│   ├── error_reporter.py    # Copilot SDK integration for auto-reporting
│   ├── config.py            # Environment config loader
│   ├── requirements.txt     # Python dependencies
│   └── templates/
│       └── index.html       # Web UI for the calculator
├── infra/
│   ├── main.bicep           # Azure infrastructure (ACR, Container App, Log Analytics)
│   └── main.bicepparam      # Bicep parameters
├── scripts/
│   ├── setup-infra.sh       # Provision Azure resources
│   └── deploy-app.sh        # Build & deploy container to Azure
├── tests/
│   └── test_app.py          # pytest tests
├── .github/
│   ├── agents/
│   │   └── bug-solver.agent.md        # Custom agent for fixing bugs
│   ├── workflows/
│   │   ├── deploy.yml                 # CI/CD deploy on push to main
│   │   ├── verify-issue.md            # Agentic Workflow source
│   │   └── verify-issue.lock.yml      # Compiled Agentic Workflow (auto-generated)
│   └── copilot-instructions.md        # Code review guidelines
└── Dockerfile
```

## Setup Guide

### Step 1: Fork the Repository

Fork this repo to your own GitHub account or organization.

```bash
gh repo fork <owner>/ghcp-e2e-demo --clone
cd ghcp-e2e-demo
```

### Step 2: Create GitHub Tokens (PATs)

You need **two** Personal Access Tokens:

#### Token 1: `GITHUB_TOKEN` (for issue creation)
- Go to **Settings > Developer settings > Personal access tokens > Fine-grained tokens**
- Repository access: your forked repo only
- Permissions:
  - **Issues**: Read and write
  - **Contents**: Read

#### Token 2: `COPILOT_GITHUB_TOKEN` (for Copilot SDK + agentic workflows)
- Go to **Settings > Developer settings > Personal access tokens > Fine-grained tokens**
- Repository access: your forked repo only
- Permissions:
  - **Copilot**: Read and write (required for the Copilot SDK inside the container)
  - **Contents**: Read

> **Note**: The Copilot SDK (`github-copilot-sdk`) running inside the container uses `COPILOT_GITHUB_TOKEN` to authenticate with the Copilot API. Without it, errors won't be auto-reported.

### Step 3: Create GitHub Labels

The workflow relies on these labels existing in your repo. Create them if they don't exist:

```bash
gh label create "needs-verification" --color "d876e3" --description "Issue needs automated verification" --repo <your-fork>
gh label create "auto-bug" --color "e11d48" --description "Automatically reported bug" --repo <your-fork>
gh label create "verified-bug" --color "b60205" --description "Bug verified by automated workflow" --repo <your-fork>
gh label create "not-reproducible" --color "0e8a16" --description "Bug could not be reproduced" --repo <your-fork>
```

### Step 4: Configure Repository Settings

#### Enable Copilot Coding Agent
1. Go to your repo **Settings > Copilot > Coding agent**
2. Enable the coding agent
3. Make sure the `bug-solver` custom agent appears in the agents list

#### Enable Agentic Workflows
1. Go to your repo **Settings > Actions > General**
2. Ensure "Allow all actions and reusable workflows" is selected
3. The `verify-issue.lock.yml` workflow should appear under **Actions**

### Step 5: Provision Azure Infrastructure

Update the Bicep parameters for your environment:

```bash
# Edit infra/main.bicepparam to set your preferred Azure region
# Default is 'uksouth' — change if needed
```

Then provision:

```bash
# Login to Azure if not already
az login

# Create resource group and deploy infrastructure
./scripts/setup-infra.sh
```

This creates:
- Azure Container Registry (ACR)
- Log Analytics Workspace
- Container App Environment

### Step 6: Initial Deployment (Local)

Set environment variables and deploy:

```bash
export GITHUB_REPO="<your-username>/<your-fork>"
export GITHUB_TOKEN="<token-from-step-2>"
export COPILOT_GITHUB_TOKEN="<copilot-token-from-step-2>"

./scripts/deploy-app.sh
```

The script will output the app URL. Verify it works:

```bash
curl https://<your-app-url>/health
# Should return: {"status":"healthy"}
```

### Step 7: Configure GitHub Secrets (for CI/CD)

Add these secrets to your repo for the deploy workflow:

```bash
# Azure credentials (JSON output from: az ad sp create-for-rbac --name ghcpe2e-sp --role contributor --scopes /subscriptions/<sub-id>/resourceGroups/ghcpe2e-rg)
gh secret set AZURE_CREDENTIALS --repo <your-fork> < azure-creds.json

# GitHub PAT for issue creation (same token from Step 2)
gh secret set GH_PAT --repo <your-fork>

# Copilot token for Copilot SDK and agentic workflows
gh secret set COPILOT_GITHUB_TOKEN --repo <your-fork>
```

> **Important**: The deploy workflow uses `secrets.GH_PAT` (not `secrets.GITHUB_TOKEN`) because the built-in `GITHUB_TOKEN` doesn't have permission to create issues via the API.

### Step 8: Compile the Agentic Workflow

If you make changes to `.github/workflows/verify-issue.md`, you need to recompile:

```bash
gh aw compile verify-issue
```

This regenerates `verify-issue.lock.yml`. Commit and push both files.

> **Gotcha**: The compiled lock file is auto-generated. Never edit it directly. Always edit the `.md` source and recompile.

## Running the Demo

### 1. Trigger the Bug

Navigate to the app URL and divide by zero:

```
https://<your-app-url>/calculate?a=10&b=0
```

Or use the web UI and enter `0` as the divisor.

### 2. Wait for the Issue (~15-30 seconds)

The error handler catches the `ZeroDivisionError`, sends it to the Copilot SDK, which creates a GitHub issue with labels `auto-bug` and `needs-verification`.

### 3. Verify Workflow Runs Automatically

The `needs-verification` label triggers the **Verify Auto-Reported Issue** agentic workflow. It:
- Installs dependencies and starts the app
- Reproduces the bug using Playwright and curl
- Takes screenshots as evidence
- Classifies the issue as `verified-bug` or `not-reproducible`

Check progress: **Actions tab > Verify Auto-Reported Issue**

> **Note**: Two workflow runs appear per issue — one for each label (`needs-verification` runs, `auto-bug` is skipped). This is expected GitHub behavior.

### 4. Assign the Bug-Solver Agent

Once the issue is labeled `verified-bug`:
1. Open the issue on github.com
2. In the right sidebar, click **Assignees**
3. Select **Copilot**
4. In the agent dropdown, select **bug-solver**

The agent will:
- Read the issue and traceback
- Find the root cause in `app/main.py`
- Add a `b == 0` check before the division
- Write regression tests
- Take before/after screenshots
- Open a PR with `Fixes #<issue-number>`

### 5. Review the PR

Once the agent opens the PR:
1. Open the PR
2. Request a review from **Copilot** (the Code Review agent)
3. Copilot reviews using the guidelines in `.github/copilot-instructions.md`

> **Note**: Copilot automatically adds the issue assigner as a reviewer — this is by design and cannot be disabled.

### 6. Merge and Redeploy

Merge the PR to `main`. The deploy workflow automatically:
- Builds a new Docker image
- Pushes to ACR
- Redeploys the Container App
- Runs a health check

## Troubleshooting

### "Invalid custom agent config: mcp-servers.\<server\>.tools is required"

The `bug-solver.agent.md` MCP server config must include a `tools` field. Even to allow all tools, you need:

```yaml
mcp-servers:
  playwright:
    type: stdio
    command: npx
    args: ["@playwright/mcp@latest"]
    tools: [""]    # <-- Required! Empty string = all tools
```

### Verify workflow fails with secret validation error

Make sure `COPILOT_GITHUB_TOKEN` is set as a repository secret. The agentic workflow engine needs it to authenticate with the Copilot API.

### No issue gets created after triggering the error

- Check that `GITHUB_TOKEN` and `GITHUB_REPO` environment variables are set in the Container App
- Check container logs: `az containerapp logs show -n ghcpe2e-app -g ghcpe2e-rg`
- The Copilot SDK needs `COPILOT_GITHUB_TOKEN` to call the LLM that composes the issue
- It can take 15-30 seconds for the issue to appear (the SDK calls an LLM to analyze the error)

### Two workflow runs per issue

This is expected. The issue is created with both `auto-bug` and `needs-verification` labels. GitHub fires a `labeled` event for each, creating two workflow runs. Only the `needs-verification` one actually executes; the `auto-bug` one is skipped by the `if` condition.

### Bug-solver agent doesn't appear in the dropdown

- Ensure `.github/agents/bug-solver.agent.md` is on the default branch
- The agent file must have valid YAML frontmatter with `name`, `description`, and `tools`
- Any MCP server defined must include the `tools` field (see first troubleshooting item)
- Refresh the agents tab at `https://github.com/<owner>/<repo>/agents`

### Deploy workflow fails

- Verify `AZURE_CREDENTIALS` secret contains valid service principal JSON
- The service principal needs Contributor role on the resource group
- Check that the ACR and Container App Environment exist (run `./scripts/setup-infra.sh` first)

### Agentic workflow compilation fails

Common errors when running `gh aw compile`:
- `'files' not a valid toolset` — use `repos` instead of `files` in the `toolsets` list
- `'contents: write' not allowed` — agentic workflows can only have `contents: read`; use `safe-outputs` for write operations

## Environment Variables Reference

| Variable | Where | Purpose |
|---|---|---|
| `GITHUB_REPO` | Container App env | Repository in `owner/repo` format for issue creation |
| `GITHUB_TOKEN` | Container App secret | PAT with issues write permission |
| `COPILOT_GITHUB_TOKEN` | Container App secret + GitHub repo secret | PAT for Copilot SDK and agentic workflows |
| `AZURE_CREDENTIALS` | GitHub repo secret | Azure service principal JSON for deploy workflow |
| `GH_PAT` | GitHub repo secret | Same as GITHUB_TOKEN, used by deploy workflow |

## Key Configuration Files

| File | Purpose | When to Edit |
|---|---|---|
| `.github/agents/bug-solver.agent.md` | Custom agent profile | To change fix strategy, tools, or MCP servers |
| `.github/workflows/verify-issue.md` | Verification workflow source | To change how bugs are verified |
| `.github/copilot-instructions.md` | Code review guidelines | To change review criteria |
| `infra/main.bicepparam` | Azure deployment params | To change region, naming, or resource config |
| `app/error_reporter.py` | Error-to-issue pipeline | To change how errors are reported |
