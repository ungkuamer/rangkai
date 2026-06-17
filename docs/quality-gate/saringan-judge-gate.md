# Saringan Judge Gate Integration

This document explains how Rangkai invokes Saringan's deterministic validation and advisory Contextual Judge Gate before creating an Integration Pull Request.

## Architecture

Rangkai does **not** import Saringan as a Python dependency. Per ADR 0009, Rangkai invokes quality gates as external CLI commands.

Current flow:

```text
Rangkai Agent Run succeeds
→ Rangkai calls quality_gate.command
→ wrapper prepares judge inputs
→ wrapper runs saringan validate
→ if validation passes, wrapper runs saringan judge
→ wrapper prints Validation Result JSON to stdout
→ Rangkai parses status
→ Rangkai creates or blocks Integration Pull Request
```

The wrapper script lives at:

```text
scripts/saringan-quality-gate.sh
```

## Wrapper contract

The wrapper is designed for Rangkai's `quality_gate` command contract:

- it is called as an external executable
- it prints one parseable Saringan-style Validation Result JSON object to stdout
- it exits `0` if it successfully produced a result, even when that result is `failed` or `error`
- it only exits non-zero for wrapper-level crashes before it can emit JSON

Rangkai then interprets the JSON `status`:

| Status | Rangkai behavior |
| --- | --- |
| `passed` | Continue to Integration Pull Request creation |
| `failed` | Block Integration Pull Request, label issue `needs-triage`, persist diagnostics |
| `error` | Block Integration Pull Request, label issue `needs-triage`, persist diagnostics |

## Configure Rangkai

In `rangkai.yaml`, configure the repo quality gate to call the wrapper:

```yaml
quality_gate:
  enabled: true
  command: /home/ungku/programming/rangkai/scripts/saringan-quality-gate.sh
  args_template:
    - "{worktree_path}"
    - "{issue_number}"
    - "{prd_branch}"
    - "{implementation_branch}"
  timeout_seconds: 900
```

The wrapper receives:

```bash
scripts/saringan-quality-gate.sh \
  <worktree_path> \
  <issue_number> \
  <prd_branch> \
  <implementation_branch>
```

## Judge Harness Configuration (recommended)

Saringan supports a dedicated **Judge Harness Configuration** file that declares
named harnesses with provider, model, command, and timeout per-harness.
This replaces the single `SARINGAN_JUDGE_MODEL` env var with a richer setup.

Rangkai's pre-built config lives at:

```text
docs/quality-gate/judge-harness-config.toml
```

It defines three harnesses:

| Harness | Provider | Model | Notes |
|---------|----------|-------|-------|
| `legacy-litellm` | openai | openai/kimi-k2.6 | Built-in LiteLLM path (planned) |
| `pi` | anthropic | claude-sonnet-4 | Pi headless agent |
| `codex-headless` | openai | gpt-5 | Codex CLI |

Set it via environment:

```bash
export SARINGAN_JUDGE_CONFIG=/home/ungku/programming/rangkai/docs/quality-gate/judge-harness-config.toml
export SARINGAN_JUDGE_HARNESS=legacy-litellm
export SARINGAN_JUDGE_PROVIDER=openai
export SARINGAN_JUDGE_MODEL=openai/kimi-k2.6
```

Or pass it on the CLI to the wrapper (requires updating the wrapper to forward
`--judge-config`).

### How harness selection works

1. `--harness <name>` CLI flag → named harness from config
2. `SARINGAN_JUDGE_HARNESS` env var → named harness from config
3. `default_harness` from the config TOML
4. If no config at all → built-in LiteLLM path (backward compatible)

Provider and model can be overridden at runtime via `--provider` / `--model`
CLI flags or `SARINGAN_JUDGE_PROVIDER` / `SARINGAN_JUDGE_MODEL` env vars.
These overrides take precedence over the harness definition.

> **Note:** The `legacy-litellm` harness requires the
> `saringan.legacy_adapter_runner` entry point which is planned but not yet
> implemented. Until then, omit `--judge-config` to fall back to the
> built-in LLM path, or set `SARINGAN_SKIP_JUDGE=1` to skip the judge.

## Required environment (legacy path)

Before starting Rangkai, export the Saringan executable and judge model:

```bash
export SARINGAN_BIN=/home/ungku/programming/saringan/.venv/bin/saringan
export SARINGAN_JUDGE_MODEL=gpt-4o-mini
```

Install Saringan with judge dependencies:

```bash
cd /home/ungku/programming/saringan
uv pip install -e ".[judge]"
```

The Rangkai process must inherit any API keys needed by LiteLLM. For OpenAI:

```bash
export OPENAI_API_KEY=...
```

## Custom model endpoints

Saringan calls the judge model through LiteLLM. Rangkai does not call the model directly.

```text
Rangkai
→ scripts/saringan-quality-gate.sh
→ saringan judge
→ LiteLLM
→ model provider API or custom endpoint
```

For an OpenAI-compatible custom endpoint:

```bash
export OPENAI_API_KEY="your-key-or-dummy-if-not-required"
export OPENAI_API_BASE="https://your-endpoint.example.com/v1"
export SARINGAN_JUDGE_MODEL="openai/your-model-name"
```

Some environments may require `OPENAI_BASE_URL` instead of, or in addition to, `OPENAI_API_BASE`:

```bash
export OPENAI_BASE_URL="https://your-endpoint.example.com/v1"
```

For Azure OpenAI:

```bash
export AZURE_API_KEY="..."
export AZURE_API_BASE="https://your-resource.openai.azure.com"
export AZURE_API_VERSION="2024-02-15-preview"
export SARINGAN_JUDGE_MODEL="azure/your-deployment-name"
```

For Ollama-style local models:

```bash
export OLLAMA_API_BASE="http://localhost:11434"
export SARINGAN_JUDGE_MODEL="ollama/llama3.1"
```

Caveat: Saringan currently requests structured JSON-schema output. The endpoint/model should reliably return JSON compatible with Saringan's schema, or the judge command will return `status: error`.

## Optional environment

Run only deterministic validation and skip the judge:

```bash
export SARINGAN_SKIP_JUDGE=1
```

Use a specific Saringan config file:

```bash
export SARINGAN_CONFIG=/path/to/saringan.toml
```

Add extra convention/context files for the judge. Paths may be absolute or relative to the worktree. Separate multiple files with `:`:

```bash
export SARINGAN_CONVENTIONS="docs/adr/0009-quality-gate-before-integration-pull-request.md:ECOSYSTEM.md"
```

Override the diff base ref:

```bash
export SARINGAN_BASE_REF="origin/prd/some-branch"
```

## Generated files

For issue `#123`, the wrapper writes input and intermediate files under:

```text
<worktree>/quality-gate-inputs/
```

Files include:

| File | Purpose |
| --- | --- |
| `diff.patch` | Diff sent to `saringan judge` |
| `issue.md` | Issue context sent to `saringan judge` |
| `conventions.md` | Repo conventions/context sent to `saringan judge` |
| `validate.result.json` | Raw stdout from `saringan validate` |
| `validate.stderr` | Human-readable stderr from `saringan validate` |
| `judge.result.json` | Raw stdout from `saringan judge` |
| `judge.stderr` | Human-readable stderr from `saringan judge` |

Rangkai also persists quality-gate diagnostics under:

```text
<worktree>/quality-gate/
```

## Blocking policy

Current policy:

- `saringan validate` is blocking.
- `saringan judge` is advisory-first.

That means judge findings are preserved in the JSON evidence, but a completed judge run usually returns `status: passed` and allows Rangkai to continue.

If we later want judge findings to block Integration Pull Request creation, the wrapper can convert judge evidence into `status: failed`. Example possible blocking conditions:

- `scope_guard.verdict == "no"`
- `completion_score < 1.0`
- any acceptance criterion verdict is `no`
- any debug-artifact advisory is present

Changing the judge from advisory to blocking is an orchestration policy decision and should be documented in a new ADR or an ADR 0009 update.

## Smoke test

Minimal local smoke test using the real wrapper with a fake Saringan executable:

```bash
tmp=$(mktemp -d)
cd "$tmp"
git init
git config user.email test@example.com
git config user.name Test
printf 'a\n' > file.txt
git add file.txt
git commit -m base
git branch prd-test
printf 'b\n' >> file.txt
git commit -am change

cat > "$tmp/fake-saringan" <<'SH'
#!/usr/bin/env bash
if [ "$1" = "validate" ]; then
  echo '{"status":"passed","check_outcomes":[]}'
elif [ "$1" = "judge" ]; then
  echo '{"status":"passed","check_outcomes":[{"id":"contextual_judge","evidence":{"completion_score":1.0}}]}'
else
  echo '{"status":"error","message":"bad command"}'
fi
SH
chmod +x "$tmp/fake-saringan"

SARINGAN_BIN="$tmp/fake-saringan" \
SARINGAN_JUDGE_MODEL=fake \
SARINGAN_BASE_REF=prd-test \
/home/ungku/programming/rangkai/scripts/saringan-quality-gate.sh \
  "$tmp" 123 prd-test impl/test
```

Expected stdout:

```json
{"status":"passed","check_outcomes":[{"id":"contextual_judge","evidence":{"completion_score":1.0}}]}
```
