# Snitch for GitHub

License-gated security audit on every pull request. Runs on your own GitHub Actions runner; your code never leaves your environment.

- Sticky PR comment with findings by severity
- Pass/fail check run, configurable severity threshold
- SARIF uploaded to GitHub Code Scanning
- Bring your own AI key (OpenRouter, Anthropic, OpenAI, Google, Copilot)
- Smart re-scan skips harmless pushes after a clean PR

Customer-facing docs and license-key signup: [snitchplugin.com](https://snitchplugin.com)

## Quick start

Drop this file at `.github/workflows/snitch.yml`:

```yaml
name: Snitch Security Scan
on:
  pull_request:
    types: [opened, synchronize, ready_for_review]

permissions:
  contents: read
  pull-requests: write
  checks: write
  statuses: write
  security-events: write

jobs:
  snitch:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: naieum/snitch-github-action@v1
        with:
          snitch-license-key: ${{ secrets.SNITCH_LICENSE_KEY }}
          openrouter-api-key: ${{ secrets.OPENROUTER_API_KEY }}
```

Add two repo secrets before the first PR:

1. `SNITCH_LICENSE_KEY` — get it at [snitchplugin.com/dashboard/keys](https://snitchplugin.com/dashboard/keys).
2. One AI provider key. Use whichever you already pay for:
   - `OPENROUTER_API_KEY` (recommended; works with any model)
   - `ANTHROPIC_API_KEY`
   - `OPENAI_API_KEY`
   - `GOOGLE_API_KEY` (Gemini via AI Studio)
   - `COPILOT_TOKEN` (requires an active GitHub Copilot subscription)

Open a PR. The Action scans changed files, comments once, writes a check run, uploads SARIF. Re-runs on every push until the check is green.

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `snitch-license-key` | yes | — | License key from `snitchplugin.com/dashboard/keys`. |
| `openrouter-api-key` | one provider | — | OpenRouter key. Works with any supported model. |
| `anthropic-api-key` | one provider | — | Anthropic (Claude) API key. |
| `openai-api-key` | one provider | — | OpenAI API key. |
| `google-api-key` | one provider | — | Google AI Studio key (Gemini). |
| `copilot-token` | one provider | — | GitHub Copilot OAuth token. |
| `trigger-mode` | no | `smart` | `smart` / `always` / `manual`. See [Triggers](#triggers). |
| `provider` | no | auto | Force a provider (`openrouter`, `anthropic`, `openai`, `google`, `copilot`). Default picks the first key present. |
| `model` | no | provider default | Model ID for the chosen provider (e.g. `claude-sonnet-4-6`, `gpt-4o-mini`, `gemini-2.5-pro`). |
| `fail-on` | no | `high` | Minimum severity that fails the check run: `critical` / `high` / `medium` / `low` / `none`. |
| `categories` | no | auto | Comma-separated category numbers, e.g. `1,2,5`. Omit to auto-detect from changed file types. |
| `paths` | no | all | Comma-separated glob filters; only files matching these paths are scanned. |

## Outputs

Use these in downstream workflow steps:

| Output | Description |
|---|---|
| `findings-count` | Total findings across all severities. |
| `critical-count` | Number of critical-severity findings. |
| `high-count` | Number of high-severity findings. |
| `report-url` | URL of the sticky PR comment. |
| `scan-id` | Snitch scan ID for dashboard correlation. |
| `provider-used` | Provider selected for this scan. |
| `model-used` | Model selected for this scan. |

## Triggers

`trigger-mode` controls when the Action runs. Override per-repo with a `.snitch.yml` at the repo root.

- **`smart`** (default) — Scans on `opened` and `ready_for_review`. On `synchronize` (push-to-PR), re-scans only if the prior scan found criticals or highs. If the last scan was clean, the push is skipped to save quota and CI minutes.
- **`always`** — Scans on every `opened` / `synchronize` / `ready_for_review`.
- **`manual`** — Ignores normal PR events. Only scans when a PR has `[snitch]` in its title/body, or when someone comments `/snitch` on the PR.

## Per-repo configuration

Drop a `.snitch.yml` at your repo root to override defaults without editing the workflow YAML. Schema at [snitchplugin.com/docs](https://snitchplugin.com/docs).

## Choosing a provider

| Provider | Strength | Notes |
|---|---|---|
| OpenRouter | flexibility | Single key, pick any supported model via the `model` input. Typically the cheapest path. |
| Anthropic | accuracy | Claude models do well on security-specific reasoning. Good default for high-signal scans. |
| OpenAI | cost | `gpt-4o-mini` is fast and cheap for high-volume repos; step up to `gpt-4o` for critical paths. |
| Google | long context | Gemini handles very large diffs well. |
| Copilot | no extra spend | Free if you already pay for GitHub Copilot. Limited model selection. |

## Quotas

Your active Snitch subscription tier determines how many scans you can run per month. Quota is enforced server-side; when you run out, the Action fails the job with HTTP 402 and a link to your billing page.

| Tier | Scans per month |
|---|---|
| Free | 50 |
| Pro | 100,000 |
| Team | 1,000,000 |
| Enterprise | 10,000,000 |

Upgrade or view your current usage at [snitchplugin.com/dashboard/billing](https://snitchplugin.com/dashboard/billing).

## Privacy

- The Action runs on your runner. Source code, diff content, and finding text stay in your environment.
- Scan metadata reported back to Snitch: user ID, repo owner, repo name, PR number, scan mode, provider, model, file count, token counts, finding counts by severity, duration. No code content, no finding text.
- AI provider calls are direct from your runner to the provider you configured. Snitch never proxies AI traffic.

Full privacy policy: [snitchplugin.com/privacy](https://snitchplugin.com/privacy).

## AI disclaimer

Findings are generated by an AI model and may be incorrect, incomplete, or fabricated. Treat them as a starting point for human review, not a definitive list. Full disclaimer: [snitchplugin.com/ai-disclaimer](https://snitchplugin.com/ai-disclaimer).

## Troubleshooting

**The Action fails with "license key invalid".**
Verify the secret is set: `gh secret list --repo OWNER/REPO | grep SNITCH_LICENSE_KEY`. Verify the key matches the one at [snitchplugin.com/dashboard/keys](https://snitchplugin.com/dashboard/keys). Keys are revocable; make sure yours hasn't been rotated.

**The Action fails with "quota exceeded" (HTTP 402).**
Your monthly scan quota has run out. Upgrade at [snitchplugin.com/dashboard/billing](https://snitchplugin.com/dashboard/billing), or wait for the monthly reset. You can also narrow the `paths` input to scan fewer files per PR.

**No PR comment appears after the job completes.**
Check the job's `permissions:` block. The Action needs `pull-requests: write` to post comments and `checks: write` to write the check run. `security-events: write` is needed for SARIF upload.

**Wrong provider or model selected.**
Pass `provider` and `model` explicitly. For example:
```yaml
with:
  provider: anthropic
  model: claude-sonnet-4-6
```

**Action re-scans on every push and I only want it on PR open.**
Set `trigger-mode: manual`. Only scans when you comment `/snitch` or add `[snitch]` to the PR title/body.

**I need to turn it off temporarily.**
Delete or comment out `.github/workflows/snitch.yml`, or remove the `uses:` step.

## FAQ

**Does Snitch see my source code?**
No. The Action runs on your runner; source code never reaches Snitch servers. Only scan metadata is sent back.

**Can I self-host the scan engine?**
Not in v1. Enterprise customers who need full on-prem can contact us.

**Which languages are supported?**
Any language the model can read. Best signal today is on JavaScript/TypeScript, Python, Go, Rust, Java, Ruby, PHP, C#, and shell. The category catalog is language-agnostic; the AI adapts.

**How do I give feedback during the beta?**
Email [eric.waters@snitch.live](mailto:eric.waters@snitch.live?subject=GitHub%20Action%20feedback) or open an issue on this repo.

## Support

- General: [support@snitchplugin.com](mailto:support@snitchplugin.com)
- Security disclosure: [security@snitchplugin.com](mailto:security@snitchplugin.com)
- Legal: [legal@snitchplugin.com](mailto:legal@snitchplugin.com)

## License

Proprietary. See [LICENSE](./LICENSE). Use requires a valid Snitch license key.
