# Aident Business Central AI Code Reviewer

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

Run an AI-powered, Business Central-specific code review on pull requests. This repository provides a GitHub composite action that:

- Collects the PR unified diff and optional context files (app.json, permission sets, markdowns, linked issues).
- Sends a structured prompt to your chosen LLM provider (Azure OpenAI, OpenAI, OpenRouter.ai).
- Posts a summary review and fine-grained inline comments on the PR.

**Demo**: https://github.com/skeck/bc-reviewer/pull/11

## Why this is Business Central–specific

This action is not a generic code reviewer. It is tailored for Business Central AL projects:

- It only reviews AL and BC app artifacts (`**/*.al`, `app.json`, permission sets, markdown docs) and builds deterministic BC object metadata from the HEAD version of your files.
- The review prompt is grounded in AL Guidelines [ALGuidelines.dev](https://alguidelines.dev/) and standard BC design patterns (event-driven extensibility, AL-Go structure, upgrade codeunits, etc.).
- Feedback is BC domain-aware: posting routines, journals, ledgers, dimensions, VAT/currencies, inventory costing, approvals, permissions, data classification, AppSource vs. Per-Tenant rules, and SaaS/multi-tenant safety are all first-class concerns.
- The reviewer prefers safe, upgrade-friendly, tenant-aware suggestions over "clever" but risky shortcuts, and explicitly calls out high-risk changes (posting, schema/upgrade, security/permissions, public APIs/events).

Combined with the [`.continue/prompts/review-pr.md`](.continue/prompts/review-pr.md) prompt and [`PR Reviewer Rules`](.continue/rules/pr-reviewer.md), this makes the review behavior strongly aligned with real-world BC AL practices rather than a generic LLM code review.

## Highlights
- Opinionated defaults and rules are bundled in the action (default-config.yaml) for sensible behaviour out of the box.
- MODELS_BLOCK enables flexible model/provider selection (OpenAI, Azure Foundry, OpenRouter, ...).

## Quick concepts
- MODELS_BLOCK (required): a multiline YAML string containing a `models:` array. It fully replaces the embedded `models:` section in default-config.yaml.
- Merged config: the action writes a merged YAML to `$RUNNER_TEMP/continue-config.yaml` and sets `CONTINUE_CONFIG` to that path before invoking the review scripts.
- Secrets: supply via `secrets.*` (interpolate directly in MODELS_BLOCK) or use placeholders like `apiKey: "{{AZURE_OPENAI_KEY}}"` and set corresponding env vars in the workflow.

## Example workflow

```yaml
name: "BC Code Reviewer"
on:
  pull_request:
    branches: [ main ]
    types: [ opened, reopened, ready_for_review ]
  issue_comment:
    types: [created]
  workflow_dispatch:

permissions:
  contents: read
  pull-requests: write

jobs:
  review:
    # Only run if this is a pull_request event or an issue_comment on a PR that includes '/review'
    if: >
      (github.event_name == 'pull_request') ||
      (github.event_name == 'issue_comment' &&
       github.event.issue.pull_request != null &&
       contains(fromJSON('["OWNER","MEMBER","COLLABORATOR"]'), github.event.comment.author_association) &&
       startsWith(github.event.comment.body, '/review'))
    runs-on: ubuntu-latest

    # Optional but recommended to avoid overlapping runs on the same PR
    concurrency:
      group: ai-review-${{ github.event.pull_request.number || github.event.issue.number }}
      cancel-in-progress: true

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Run AI Code Review
        # Replace with the immutable commit SHA approved by your organization.
        uses: your-company/bc-ai-reviewer@0123456789abcdef0123456789abcdef01234567
        with:
          GITHUB_TOKEN: ${{ github.token }}
          MODELS_BLOCK: |
            models:
              - name: GPT-5 @Azure OpenAI w Responses API
                provider: azure
                model: gpt-5
                apiBase: https://your-azure-resource.openai.azure.com/openai/v1 # use a secret here
                apiKey: ${{ secrets.AZURE_OPENAI_KEY }}
                roles: [chat, edit, apply]
                requestOptions:
                  extraBodyProperties:
                    reasoning:
                      effort: high

          ALLOWED_PROVIDER_HOSTS: "your-azure-resource.openai.azure.com"

          # Review configuration
          MAX_COMMENTS: 20
          AUTO_DETECT_APPS: true
          INCLUDE_APP_PERMISSIONS: true
          INCLUDE_APP_MARKDOWN: true
          INCLUDE_PATTERNS: "**/*.al,**/*.json"
          EXCLUDE_PATTERNS: ""
          CONTEXT_FILES: ""
          ISSUE_COUNT: 0
          FETCH_CLOSED_ISSUES: true
          BASE_PROMPT_EXTRA: ""
          DEBUG_PAYLOAD: false
          APPROVE_REVIEWS: false
          MAX_FILE_BYTES: 200000
          MAX_PAYLOAD_BYTES: 4000000
          MAX_CONTEXT_FILES: 30
          TRUSTED_ASSOCIATIONS: "OWNER,MEMBER,COLLABORATOR"
          PROJECT_CONTEXT: |
            Repository contains Business Central AppSource Apps, related documentation based on docfx, and a slightly extended version of AL-Go for GitHub.
```

Note: The review can be re-triggered by commenting `/review` on the pull request (issue_comment event).

## Composite Action Inputs

These inputs are defined in action.yml (defaults in parentheses).

| Input | Type | Default | Description |
|---|:---:|---:|---|
| GITHUB_TOKEN | string | required | GitHub token with repo scope. Defaults to `github.token` when not passed explicitly. |
| MODELS_BLOCK | string (YAML) | required | Multiline YAML that fully replaces the default `models:` section. |
| APPROVE_REVIEWS | boolean | false | Must remain false. The hardened action rejects AI approval or request-change authority. |
| MAX_COMMENTS | number | 10 | Maximum inline comments to post (0 = unlimited; GitHub caps at 1000 per PR). |
| PROJECT_CONTEXT | string | "" | Optional architecture / guidelines text injected into the prompt. |
| CONTEXT_FILES | string | "" | Comma-separated globs for extra context files (read from HEAD). |
| INCLUDE_PATTERNS | string | "**/*.al" | Comma-separated globs to include in review. |
| EXCLUDE_PATTERNS | string | "" | Comma-separated globs to exclude from review. |
| ISSUE_COUNT | number | 0 | Max linked issues to fetch (0 = all). |
| FETCH_CLOSED_ISSUES | boolean | true | Include closed issues as context. |
| AUTO_DETECT_APPS | boolean | true | Discover app.json for changed files. |
| INCLUDE_APP_PERMISSIONS | boolean | true | Include `*.PermissionSet.al` and `*.Entitlement.al` for each relevant app. |
| INCLUDE_APP_MARKDOWN | boolean | true | Include `*.md` for each relevant app. |
| BASE_PROMPT_EXTRA | string | "" | Free-form text appended to the base review prompt. |
| DEBUG_PAYLOAD | boolean | false | When true, prints the JSON payload to logs (use sparingly; can be large). |
| SNIPPET_CONTEXT_LINES | number | 12 | Lines of context around each changed line in numbered snippets. |
| MAX_FILE_BYTES | number | 200000 | Maximum UTF-8 bytes for an individual file or changed-file context. |
| MAX_PAYLOAD_BYTES | number | 4000000 | Maximum UTF-8 bytes for the complete model prompt. |
| MAX_CONTEXT_FILES | number | 30 | Maximum repository context files sent to the model. |
| TRUSTED_ASSOCIATIONS | string | OWNER,MEMBER,COLLABORATOR | Author associations allowed to invoke `/review`. |
| ALLOWED_PROVIDER_HOSTS | string | required | Exact hostname allowlist for explicit HTTPS model endpoints. The action fails closed when the endpoint is missing or unapproved. |


## Runtime behavior (what the action does)

- Installs Node.js v24 and lockfile-pinned runtime dependencies with `npm ci`.
- Merges continue [MODELS_BLOCK](https://docs.continue.dev/reference#models) into the bundled default-config.yaml at runtime via `.github/actions/continue/merge-config.ps1`, writes the merged file to `$RUNNER_TEMP`, and sets `CONTINUE_CONFIG`.
- Executes `scripts/continue-review.ps1` which:
  - Fetches the PR unified diff and parses it with `scripts/parse-diff.js`.
  - Filters changed files by INCLUDE_PATTERNS/EXCLUDE_PATTERNS.
  - Builds numbered snippets with SNIPPET_CONTEXT_LINES of surrounding context.
  - Optionally auto-discovers app.json and includes permissions and markdown files.
  - Assembles a structured prompt and invokes Continue CLI with `.continue/prompts/review-pr.md`.
  - Posts a summary review and up to MAX_COMMENTS inline comments. If APPROVE_REVIEWS is true, uses the model’s `suggestedAction` for the summary event.

## Troubleshooting & tips

- If the reviewer prints errors about `CONTINUE_CONFIG` being invalid, ensure the action sets it to the merged config path (the composite action does this automatically).
- If the model returns non-JSON or malformed JSON, the script fails without printing raw model output. Keep `DEBUG_PAYLOAD` disabled for customer repositories.
- Never commit real API keys or secrets. Use GitHub Secrets. The merged config is written to `$RUNNER_TEMP` and is not committed.
- Pin this action and all third-party actions to approved full commit SHAs.
- Use a regional Azure deployment and private networking or an egress proxy where required by company policy. The action validates configured endpoint hosts but cannot enforce runner-level networking.
- Pilot with synthetic or non-customer AL code and retain mandatory human review before enabling customer repositories.

## Where things live in this repo

- `action.yml` — composite action entry point (wires steps and inputs).
- `scripts/continue-review.ps1` — runner script that builds the prompt, calls the Continue CLI and posts GitHub reviews.
- `.github/actions/continue/default-config.yaml` — bundled template used when merging models.
- `.github/actions/continue/merge-config.ps1` — runtime merge helper (replaces `models:` and substitutes placeholders).
- `scripts/parse-diff.js` — diff parsing helper used by the runner.

## Contributing

- PRs & issues welcome. For major feature requests please open a discussion first.

## License & privacy

- Licensed under Apache 2.0. See LICENSE for details.
- This action sends diffs and selected context to third-party LLM providers. Do not include secrets or sensitive PII in your PR diffs or context files.
