# Week 8 — Minimal AI Code Review with Microsoft Foundry

**Estimated time:** 60–90 minutes
**Goal:** Add a very simple AI code review workflow that sends your PR diff to a model deployed in Microsoft Foundry and returns the raw response.

**Prerequisites:** Complete [Week 5](week-05.md) (CI workflow), [Week 6](week-06.md) (branch protection), and [Week 7](week-07.md) (AI in GitHub Actions) before starting this week.

---

## What you will learn

- How to create a basic AI review workflow using `curl` in GitHub Actions
- How to pass a git diff into a model request payload safely
- How to call a model from Microsoft Foundry using a GitHub Secret
- Why this example is intentionally minimal and not production-ready

---

## Important context

This lesson intentionally shows the **simplest possible** AI code review setup.

- It does **not** format the model output into a polished PR comment
- It does **not** do advanced prompt engineering
- It does **not** include robustness features like retries, structured parsing, or quality gates

Its purpose is only to prove one key concept: **you can call any suitable model available through Microsoft Foundry from a GitHub Actions workflow.**

---

## Theory (10–15 min)

### Why use this minimal version?

Before building a full AI reviewer, it helps to verify the integration path first:

1. Workflow can read repository changes
2. Workflow can authenticate to your model endpoint
3. Model can return a response in CI

Once this works, you can improve formatting, reliability, and review quality in later iterations.

### Security note

Never hardcode API keys or endpoint credentials in YAML files. Store secrets in:

**Settings → Secrets and variables → Actions**

For this lesson, you will use:

| Secret name | Purpose |
|------------|---------|
| `AZURE_API_KEY` | Authenticates your request to the Foundry endpoint |

---

## Tasks

### 1. Create the workflow file

Create `.github/workflows/ai-review.yml`:

```yaml
name: ai-review

on:
  pull_request:
    branches:
      - 'main'
  push:
    branches:
      - 'main'

jobs:
  lint-html:
    runs-on: ubuntu-latest
    steps:
      - name: checkout repo
        uses: actions/checkout@v4

      - name: Use Node.js
        uses: actions/setup-node@v4

      - name: Install dependencies
        run: npm i -g htmlhint

      - name: Run html linter
        run: npx htmlhint index.html

  ai-code-review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Run code review
        env:
          AZURE_API_KEY: ${{ secrets.AZURE_API_KEY }}
        run: |
          DIFF=$(git diff origin/main)

          JSON_PAYLOAD=$(jq -n --arg diff "$DIFF" '{
            "messages": [
              {
                "role": "user",
                "content": "these are my code changes:\n\($diff)"
              },
              {
                "role": "system",
                "content": "You are a professional programmer with knowledge of clean code principles"
              },
              {
                "role": "user",
                "content": "refactor this code and fix bugs"
              }
            ],
            "max_completion_tokens": 13107,
            "temperature": 1,
            "top_p": 1,
            "stop": [],
            "frequency_penalty": 0,
            "presence_penalty": 0,
            "model": "gpt-4.1"
          }')

          curl -X POST "https://YOUR-RESOURCE-NAME.openai.azure.com/openai/v1/chat/completions" \
            -H "Content-Type: application/json" \
            -H "api-key: $AZURE_API_KEY" \
            -d "$JSON_PAYLOAD"
```

### 2. Abstract personal details

In your real workflow, keep these values generic in examples and docs:

- Use `https://YOUR-RESOURCE-NAME.openai.azure.com/...` instead of personal or tenant-specific resource names
- Keep secrets in GitHub Secrets (never in plain text)
- Avoid exposing organization/project identifiers in public documentation

### 3. Add the repository secret

In your repository:

1. Go to **Settings → Secrets and variables → Actions**
2. Add secret:

| Secret name | Value |
|------------|-------|
| `AZURE_API_KEY` | API key from your Foundry/OpenAI resource |

### 4. Commit and open a PR

```bash
git checkout -b feature/week-8-ai-review
git add .github/workflows/ai-review.yml
git commit -m "Add minimal Foundry AI code review workflow"
git push -u origin feature/week-8-ai-review
```

Open a PR and watch the workflow run.

### 5. Validate expected behavior

- `lint-html` passes when `index.html` is valid
- `ai-code-review` executes and returns a model response in logs
- Workflow succeeds when secret and endpoint are configured correctly

---

## Expected result

- [ ] `.github/workflows/ai-review.yml` created
- [ ] `AZURE_API_KEY` configured in repository secrets
- [ ] Minimal AI request runs in GitHub Actions
- [ ] You understand this is a baseline integration, not a production reviewer

---

## Troubleshooting

**`401` or `403` from API:**
- Verify `AZURE_API_KEY` is set correctly
- Confirm key belongs to the same resource as your endpoint

**`404` from API:**
- Check endpoint format and deployment/model availability
- Replace placeholder URL with your real resource URL

**`jq: command not found`:**
- Ensure your runner image includes `jq` (Ubuntu runners usually do)
- If needed, install `jq` in a step before request construction

**No useful review output:**
- This is expected in a minimal baseline example
- Improve prompt structure and output handling in later iterations

---

> Next week: finalize the course with secrets, security response, and full pipeline understanding.
