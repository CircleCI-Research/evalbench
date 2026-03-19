# AGENTS.md

This file provides guidance to Claude Code (claude.ai/code) or any AI agent when working with code in this repository.

## Remote Repository

The canonical remote for this repo is `github.com/CircleCI-Research/evalbench`. Never target or push to `petmal/MindTrial` (the upstream fork origin). Always confirm PRs, pushes, and branch operations target `CircleCI-Research/evalbench`.

## Commands

```bash
# Build
go build -v ./...
go build -o evalbench ./cmd/evalbench/

# Test (all tests require the `test` build tag)
go test -tags=test -race -v ./...

# Run a single test package
go test -tags=test -race -v ./runners/...

# Run a specific test
go test -tags=test -race -v -run TestName ./runners/...
```

## Architecture

EvalBench is a Go CLI that runs AI model evaluations across 9+ providers (OpenAI, Anthropic, Google Gemini, DeepSeek, Mistral, xAI, Alibaba, Moonshot, OpenRouter) in parallel and compares results.

**Execution flow:** `cmd/evalbench/main.go` → parses CLI flags and config files → `runners/default_runner.go` orchestrates task execution → each task is sent to each provider via `providers/provider.go` → responses are validated by `validators/` → results are written by `formatters/`.

**Key packages:**
- `config/` — YAML-driven config structs. `config.go` holds `AppConfig` (providers, judges, tools); `tasks.go` holds task definitions with validation rules and response schemas.
- `providers/` — One file per AI provider, all implementing a shared `Provider` interface. Provider-specific quirks (streaming, reasoning effort, tool use) are encapsulated here. `providers/tools/` runs sandboxed Docker tool execution.
- `runners/` — `default_runner.go` is the main orchestrator: parallel across providers, sequential within a provider.
- `validators/` — Pluggable validation: `value_validator.go` for exact/fuzzy matching, `judge_validator.go` for LLM-based semantic evaluation.
- `formatters/` — CSV, HTML, JSONL, and log output formats.
- `cmd/evalbench/tui/` — Bubble Tea terminal UI for interactive config selection and real-time task monitoring.
- `pkg/mistralai/` and `pkg/xai/` — Auto-generated OpenAPI clients (do not edit manually).

**Configuration:** Two YAML files drive everything:
- `config.yaml` — providers (API keys, models, rate limits, parameters), judge configs, tool definitions, output paths.
- `tasks.yaml` — task prompts, expected outputs, response format (plain text or JSON schema), validation rules, file attachments.

**Claude Code slash commands** in `.claude/commands/` implement a "model race" UX: `/run-model-comparison`, `/simulate-model-comparison`, `/announce-model-comparison` (live Ken Squier-style voice commentary via Kokoro TTS), and `/stop-model-comparison`.

## Testing Notes

- All tests use build tag `test` — always pass `-tags=test`.
- Tests use `stretchr/testify` for assertions.
- `pkg/testutils/` contains shared test helpers.
- Provider tests (`providers/provider_test.go`, ~35KB) and validator tests (`validators/validator_test.go`, ~60KB) are the heaviest test files.
