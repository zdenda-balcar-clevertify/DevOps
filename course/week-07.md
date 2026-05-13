# Week 7 — Claude AI in GitHub Actions (via Azure AI Foundry)

**Estimated time:** 90–120 minutes
**Goal:** Add Claude to your pipeline in two ways: automatic code review on every PR, and an interactive `@claude` assistant that responds to mentions in issues and PR comments — all powered by Azure AI Foundry.

**Prerequisites:** Complete [Week 5](week-05.md) (CI workflow in place) and [Week 6](week-06.md) (branch protection active, PR workflow understood) before starting this week.

---

## What you will learn

- How to deploy Claude Sonnet 4.6 on Azure AI Foundry
- How to use the official `anthropics/claude-code-action` with Azure as the API provider
- How to trigger Claude automatically on every PR (auto review)
- How to use `@claude` in any issue or PR comment (interactive assistant)
- How to store Azure credentials as GitHub Secrets

---

## Cost warning

This week requires an Azure account with a **credit card**. However:
- New Azure accounts get **$200 free credit** valid for 30 days — enough for this entire course
- Set a **$2/month budget alert** so nothing unexpected happens
- The actual cost for a handful of test PRs is a **few cents at most**

If you do not want to add a credit card, you can read through this week and skip the hands-on part — the concepts (API keys, Secrets, calling an AI API from CI) still apply.

> **Subscription requirement:** Azure subscriptions that rely only on credits (free trial, student, startup credits) **do not work** with Claude models. You need a pay-as-you-go billing method (credit card). The $200 free credit is applied automatically if you have a card on file.

---

## Theory (10 min)

### How it works

`anthropics/claude-code-action` is the official Anthropic GitHub Action. It runs Claude inside a GitHub Actions job and gives it access to your repository context — the PR diff, file contents, comments.

Azure AI Foundry is an **officially supported API provider** for this action. Instead of calling the Anthropic API directly, the action routes all requests through your Azure deployment. The workflow YAML is almost identical — the only difference is a few environment variables.

Two modes:

| Mode | Trigger | What it does |
|------|---------|-------------|
| **Auto review** | PR opened or updated | Claude reads the diff and posts a review comment automatically |
| **@claude mention** | Someone writes `@claude` in a comment | Claude reads the comment and replies |

### GitHub App

The action needs the official Claude GitHub App installed on your repository so it can post comments on your behalf. You install it once at `github.com/apps/claude`.

---

## Tasks

### 1. Create an Azure account

Go to [azure.microsoft.com/free](https://azure.microsoft.com/free) and sign up.
- A credit card is required for identity verification
- You get **$200 free credit** valid for 30 days
- You will set a budget alert so you know if costs approach $2

### 2. Create a Foundry project

> **Region is critical:** Claude models are only available in **East US 2** and **Sweden Central**. You must choose one of these when creating the project — any other region will not show Claude in the model catalog.

1. Go to [ai.azure.com](https://ai.azure.com) and sign in with your Azure account
2. In the top-left corner of the portal, make sure the **New Foundry** toggle is switched **on**. If you see a toggle, flip it — the instructions below are for the new Foundry UI. If there is no toggle, you are already on the new UI.
3. Click the **project name** in the upper-left corner of the page, then click **Create new project**
4. Enter a project name: `devops-course`
5. Open **Advanced options** (click the arrow or link below the name field):
   - **Resource group:** leave as default (a new one will be created)
   - **Location: East US 2** (or Sweden Central)
6. Click **Create project** — wait ~2 minutes for the Foundry resource and project to be provisioned

> **Note:** Azure automatically creates a "Foundry resource" behind the scenes when you create a project. You do not need to create it separately. The resource name appears later under your project details.

### 3. Deploy Claude Sonnet 4.6

1. In the top navigation bar, click **Discover**
2. In the left sidebar, click **Models**
3. Search for `claude-sonnet-4-6` and click on the model card to see its details
4. Click **Deploy** → **Custom settings**
5. Claude is a partner model sold through Azure Marketplace. A terms dialog will appear — read the terms and click **Agree and Proceed**
6. On the deployment screen:
   - **Deployment name:** `claude-sonnet-4-6` (keep the default)
   - The deployment type is **Global Standard** (for Claude models this is the only option)
7. Click **Deploy** — wait ~1 minute
8. When the deployment completes, you land in the Playground. Verify the deployment status shows **Succeeded**.

### 4. Set a budget alert

1. In a separate tab, go to the Azure portal: [portal.azure.com](https://portal.azure.com)
2. Search for **Cost Management + Billing** in the top search bar
3. Go to **Budgets** → **+ Add**
4. Set:
   - Budget amount: `$2`
   - Period: Monthly
   - Alert threshold: 100% → your email
5. Save

### 5. Get your credentials

You need two values: the **Foundry resource name** and the **Project API key**.

**Get the resource name:**

1. Go back to [ai.azure.com](https://ai.azure.com)
2. In the upper-left corner, click your **project name** → **Project details**
3. Find the field **Parent resource** — copy this value (it is a short name like `devops-abc123`, not a full URL)

**Get the API key:**

1. Go to the **Home** page of your project (click the project name in the left nav)
2. Find the **Project API key** field on the Home page
3. Click **Copy Project API key**

Keep both values — you will add them to GitHub Secrets next.

> **Resource name vs URL:** The action constructs the full endpoint URL automatically. You only provide the short resource name (e.g. `devops-abc123`), not `https://devops-abc123.services.ai.azure.com`.

### 6. Install the Claude GitHub App

1. Go to [github.com/apps/claude](https://github.com/apps/claude)
2. Click **Install**
3. Choose **Only select repositories** → select your `DevOps` repository
4. Click **Install**

This gives Claude permission to read your repository and post comments. Without this step the action will fail to post review comments.

### 7. Add credentials as GitHub Secrets

1. Go to your GitHub repository → **Settings** → **Secrets and variables** → **Actions**
2. Click **New repository secret** and add both:

| Secret name | Value |
|------------|-------|
| `AZURE_FOUNDRY_RESOURCE` | The Parent resource name you copied (short name) |
| `AZURE_FOUNDRY_API_KEY` | The Project API key you copied |

### 8. Create the Claude workflow

Create `.github/workflows/claude.yml`:

```yaml
name: Claude AI

on:
  # Automatic PR review — triggers when a PR is opened or updated
  pull_request:
    types: [opened, synchronize]

  # Interactive @claude mentions — triggers when someone comments
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]

permissions:
  contents: read
  pull-requests: write
  issues: write

jobs:
  claude:
    name: Claude
    runs-on: ubuntu-latest

    steps:
      - uses: anthropics/claude-code-action@v1
        with:
          # Prompt used for automatic PR reviews.
          # When triggered by @claude mention, Claude uses the comment text instead.
          prompt: |
            You are a helpful code reviewer for a beginner DevOps course project.
            Review this pull request. The project is a plain HTML/CSS/JS landing page.
            Focus on:
            - HTML validity and accessibility
            - CSS best practices
            - JavaScript correctness
            - Any security issues
            Be concise, constructive, and beginner-friendly. Format in Markdown.
          trigger_phrase: "@claude"
        env:
          # Azure AI Foundry provider configuration
          CLAUDE_CODE_USE_FOUNDRY: "1"
          ANTHROPIC_FOUNDRY_RESOURCE: ${{ secrets.AZURE_FOUNDRY_RESOURCE }}
          ANTHROPIC_FOUNDRY_API_KEY: ${{ secrets.AZURE_FOUNDRY_API_KEY }}
          ANTHROPIC_DEFAULT_SONNET_MODEL: "claude-sonnet-4-6"
```

### 9. Commit and deploy via PR

Since `main` is protected, commit on a branch:

```bash
git checkout -b feature/add-claude-action
git add .github/workflows/claude.yml
git commit -m "Add Claude AI workflow via Azure AI Foundry"
git push -u origin feature/add-claude-action
```

Open a Pull Request. Wait for the CI lint check ✅, then merge.

Update your local `main`:

```bash
git checkout main
git pull
```

### 10. Test automatic PR review

Create a test branch with a small change:

```bash
git checkout -b feature/test-claude-review
# Edit index.html — change any text or add something
git add index.html
git commit -m "Test Claude automatic PR review"
git push -u origin feature/test-claude-review
```

Open a Pull Request. After about 30–60 seconds, Claude should post an automatic review comment.

### 11. Test @claude mention

On any open PR or issue, write a comment:

```
@claude Can you suggest how to improve the accessibility of this page?
```

Claude will read the repository context and reply in the comment thread.

Other things you can ask:
```
@claude What does the script.js file do?
@claude Are there any security issues in index.html?
@claude How could I add a dark/light mode toggle?
```

---

## Commit your work

`claude.yml` was committed on `feature/add-claude-action` and merged into `main` via PR. After merging, verify final state:

```bash
git checkout main
git pull
git status
```

Expected output:
```
On branch main
nothing to commit, working tree clean
```

New files added this week:
- `.github/workflows/claude.yml` — Claude AI workflow (auto review + @claude)

---

## Expected result

- [ ] Azure AI Foundry project created in East US 2 or Sweden Central
- [ ] `claude-sonnet-4-6` deployed (Marketplace terms accepted)
- [ ] Budget alert set to $2/month
- [ ] Claude GitHub App installed on your repository
- [ ] `AZURE_FOUNDRY_RESOURCE` and `AZURE_FOUNDRY_API_KEY` added as GitHub Secrets
- [ ] `claude.yml` workflow committed to `main` via PR
- [ ] Claude posts an automatic review comment on a test PR
- [ ] `@claude` mention in a comment gets a response

---

## How the Azure path differs from direct Anthropic API

The `anthropics/claude-code-action` works the same way — only the environment variables change:

| | Direct Anthropic API | Azure AI Foundry |
|-|----------------------|-----------------|
| **Action version** | `@v1` | `@v1` (same) |
| **API key input** | `anthropic_api_key:` | *(not used as input)* |
| **Enable Azure** | *(not needed)* | `CLAUDE_CODE_USE_FOUNDRY: "1"` |
| **Credentials** | `ANTHROPIC_API_KEY` env | `ANTHROPIC_FOUNDRY_RESOURCE` + `ANTHROPIC_FOUNDRY_API_KEY` envs |
| **Model** | `claude_args: --model claude-sonnet-4-6` | `ANTHROPIC_DEFAULT_SONNET_MODEL: claude-sonnet-4-6` |
| **Billing** | Anthropic console | Azure subscription |

---

## Troubleshooting

**Claude does not post a comment after opening a PR:**
- Check the **Actions** tab — is the workflow running at all?
- If the workflow does not appear: check that the `on: pull_request` trigger is correct YAML (indentation!)
- If it runs but fails with a 404 or auth error: re-copy `AZURE_FOUNDRY_RESOURCE` — it must be the **short resource name** (Parent resource value), not the full URL

**"The model `claude-sonnet-4-6` is not available on your foundry deployment":**
- The model is not deployed. Go to [ai.azure.com](https://ai.azure.com) → **Discover** → **Models** → search for `claude-sonnet-4-6` → **Deploy** → **Custom settings**
- If the model does not appear in the catalog, your project is in the wrong region. Claude is only available in **East US 2** and **Sweden Central**.

**"Model not found" or 404 error from Azure:**
- Verify the deployment succeeded: go to [ai.azure.com](https://ai.azure.com) → top nav **Build** → left pane **Models** → confirm the deployment status shows **Succeeded**
- The `ANTHROPIC_DEFAULT_SONNET_MODEL` value must exactly match the deployment name (default: `claude-sonnet-4-6`)

**Claude Code prompts for Anthropic login instead of using Azure:**
- `CLAUDE_CODE_USE_FOUNDRY=1` is missing or not set. Verify it is in the `env:` block of your workflow YAML.

**`@claude` mentions do not trigger a response:**
- The `on: issue_comment` trigger only fires on *new* comments, not edits
- Check the Actions tab for a workflow run after your comment
- Verify the `trigger_phrase: "@claude"` is in your workflow

**"Error: Resource not accessible by integration":**
The `permissions` block is missing or the Claude GitHub App is not installed.
- Verify your workflow has `pull-requests: write` and `issues: write` permissions
- Go to [github.com/apps/claude](https://github.com/apps/claude) and confirm the app is installed on your repo

**"Authorization failed" (HTTP 401/403):**
- Re-copy the **Project API key** from the Home page of your project at [ai.azure.com](https://ai.azure.com)
- If using a free trial or student subscription without a credit card: Claude models require a pay-as-you-go billing method

**Deployment creation fails in the Foundry portal:**
- Verify you have **Contributor** or **Owner** role on the resource group
- Verify your Azure subscription has Marketplace access enabled (required for partner models like Claude)

---

> Next week: build a minimal Foundry-based AI review workflow to prove model integration in CI. The final lesson after that covers secrets, leak response, and full-pipeline security understanding.
