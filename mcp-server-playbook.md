# MCP Server Playbook

last-update: 2026-01-23T12:00:00-0500
Author: Harold Campbell (harold.campbell@gmail.com)

This playbook captures project rules, architecture decisions, and a repeatable workflow for building new MCP servers quickly and safely. It is based on the repository docs, specs, and ADRs, and is intentionally general (not vendor-specific).

## 1) Guardrails (Non-Negotiable)

- Work only inside the repo; avoid destructive changes.
- Use specs to drive development, and follow the roadmap phases.
- Use git for source control; commit after each spec is implemented.
- Never store or commit secrets; only use environment variables.
- Prioritize security in every change.
- Keep docs fresh after each spec or significant change.
- When updating this playbook, increment the `last-update` timestamp.

## 2) Architecture Decisions (ADRs)

- Record key design choices as ADRs (language, SDKs, auth strategy, hosting).
- Prefer an OSS SDK when feasible; fall back to raw HTTP for gaps.

## 3) Standard Repo Structure

- `spec/`: specs that define scope, requirements, and acceptance criteria.
- `docs/`: documentation, ADRs, and roadmap.
- `docs/adrs/`: architecture decision records.
- `src/`: MCP server and tools.
- `tests/`: unit and integration tests.
- `scripts/`: CLI wrappers for tools.
- `logs/`: runtime logs.

## 4) Tooling & Runtime

- MCP server framework: FastMCP.
- Python env: `uv`.
- Logging: write to `logs/` with audit logging and PII redaction.

## 4a) Packaging & Distribution

### Build a macOS CLI binary (no Python required for end users)

This project can be packaged into a standalone macOS CLI binary using PyInstaller via `uv`.

#### Project root

Use the repository root (the folder that contains `src/`, `tests/`, `README.md`, etc.) as the project
directory. Run commands either from that root or with `--project /absolute/path/to/repo-root`.

#### Packaging inputs (pyproject-only)

- The canonical dependency source is `pyproject.toml` (do not use `requirements.txt`).
- Sync environments from `pyproject.toml` with `uv sync` (and `--group dev` when building).

#### One-time setup (add PyInstaller to dev group)

```bash
uv --project /absolute/path/to/repo-root add --dev pyinstaller
uv --project /absolute/path/to/repo-root sync --group dev
```

#### Build the binary (onefile)

```bash
uv --project /absolute/path/to/repo-root sync --group dev
uv --project /absolute/path/to/repo-root run pyinstaller \
  --onefile \
  --name my_server \
  src/server.py
```

Output binary: dist/my_server (created in the project root by default).
If you want a different output directory, add --distpath /absolute/path/to/dist.

#### Build the binary (onedir, faster startup)

Use onedir to avoid onefile extraction overhead at startup. This produces a directory
with the executable and bundled dependencies.

```bash
uv --project /absolute/path/to/repo-root sync --group dev
uv --project /absolute/path/to/repo-root run pyinstaller \
  --onedir \
  --name my_server \
  src/server.py
```

Run it like:

```bash
./dist/my_server/my_server
```

#### Build with a spec

```bash
uv --project /absolute/path/to/repo-root run pyinstaller my_server.spec
```

#### Spec guidance (datas + hidden imports)

- Prefer a spec that auto-collects datas, binaries, and hidden imports from the PyInstaller analysis graph.
- Seed the analysis with any known required metadata (use distribution names; e.g., `copy_metadata("mcp")`,
  optionally `copy_metadata("fastmcp")` if present) and known hidden
  imports (e.g., `lupa` modules) before auto-collecting.
- Use a two-pass Analysis pattern: initial `Analysis` (seeded), then collect packages, then a final `Analysis`
  with the combined datas/binaries/hidden imports.
- Check `build/<name>/warn-<name>.txt` after each build for missing modules/data and add explicit fixes.

#### Build with the script

```bash
./scripts/build_dist.sh
```

The script installs dev dependencies, cleans `build/` and `dist/`, and then runs PyInstaller.
Create this script in each MCP server repo to standardize builds. Use a pattern that:
- Resolves the repo root reliably (`pwd -P`).
- Runs `uv sync --group dev` before building.
- Cleans `build/` and `dist/` under the repo root.
- Supports `DIST_PATH` to override the output directory via `--distpath`.
- Invokes PyInstaller with the repo's `.spec` file (spec stays authoritative).

For faster startup times, add a companion onedir script (example name: `build_dist_onedir.sh`)
that uses a dedicated onedir spec and produces `dist/<name>/<name>`.

Template:

```bash
#!/usr/bin/env bash
set -euo pipefail

ROOT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd -P)"
DIST_PATH="${DIST_PATH:-}"

echo "Building <server-name> from: ${ROOT_DIR}"

(
  cd "${ROOT_DIR}"

  uv sync --group dev

  rm -rf "${ROOT_DIR}/build" "${ROOT_DIR}/dist"

  if [[ -n "${DIST_PATH}" ]]; then
    uv run pyinstaller --distpath "${DIST_PATH}" <server-name>.spec
  else
    uv run pyinstaller <server-name>.spec
  fi
)
```

#### Run the binary

```bash
./dist/my_server
```

#### Environment variables still work

PyInstaller binaries read env vars normally:

```bash
MY_API_TOKEN_FILE=/path/to/token ./dist/my_server
```

#### Notes

- Build on the same CPU architecture you want to support (arm64 vs x86_64).
- For a universal binary, build for each architecture and combine using a universal build workflow.
- If you hit `ModuleNotFoundError: No module named 'lupa.lua51'` when running the binary, ensure `lupa` is
  installed at build time and rebuild using `my_server.spec` (the spec includes the required
  hidden imports).
- If you hit `FileNotFoundError` for `fakeredis` `commands.json`, rebuild using `my_server.spec`
  (it includes the `fakeredis` data files required at runtime).
- The spec auto-discovers package data and hidden imports from the PyInstaller analysis graph to reduce
  one-off packaging fixes and keep the bundle smaller.

## 5) Configuration & Secrets

- Use environment variables or a secrets manager; never store secrets in repo files.
- Never log raw secrets or tokens; redact secrets in errors and logs.
- Document required and optional environment variables in `docs/`.

### Passwords & Secrets Handling (Clear Rules)

- Do not commit `.env` files or credentials; add to `.gitignore` if needed.
- Do not echo secrets in CLI output or logs; redact or omit them.
- Do not copy secrets into issues, docs, or chat logs.
- Avoid passing secrets via command line args when possible (shell history risk).
- Use least-privileged tokens/scopes for the tasks at hand.
- If a secret is accidentally exposed, rotate it immediately and document the incident.

## 6) Canonical Workflow (Spec-Driven)

1. **Pick or write a spec** in `spec/`.
    - Include Goal, Requirements, Non-Goals, Interfaces, Security, Tests, Acceptance Criteria.
2. **Implement** the spec in `src/`.
    - Use SDK first, raw HTTP only for gaps.
    - Validate inputs and normalize errors.
    - Add audit logging with redaction.
3. **Add/Update CLI scripts** in `scripts/` for each tool.
4. **Update docs** in `docs/README.md` and `docs/Roadmap.md`.
5. **Add tests** (unit + integration scaffolding) where applicable.
6. **Commit** after the spec is complete.

## 7) Core Baseline Specs (Template Order)

Use this order when bootstrapping a new MCP server:

1. **Spec 001**: MCP server baseline (FastMCP + logging).
2. **Spec 002**: Auth + config (env-only secrets).
3. **Spec 003**: Core CRUD + search tools.
4. **Spec 004**: Associations + engagements.
5. **Spec 005**: Property management.
6. **Spec 006**: Security hardening.
7. **Spec 007**: CLI scripts.
8. **Spec 008**: Tests.
9. **Spec 009+**: Domain-specific utilities (e.g., portal info, pipelines).

## 8) Implementation Checklists

### Server Baseline

- FastMCP server entrypoint in `src/server.py`.
- Logging configured (file + console) to `logs/`.
- Tools registry in `src/tools.py`.
- No secrets in repo.

### Auth & Config

- Env validation with clear errors.
- No token leakage in logs or errors.
- Document configuration in `docs/`.

### Tools (Generic)

- Input validation.
- Normalize response shapes.
- Normalize errors.
- Use SDK when possible, raw HTTP for gaps.
- Audit log requests without PII/secrets.

### Security Hardening

- PII redaction in audit logs.
- Rate limiting / backoff on API calls.
- Safe error handling (no PII/secrets).

### CLI Scripts

- One script per tool.
- Simple JSON payload input.
- No secrets in scripts.

### Tests

- Unit tests for config, validation, and routing.
- Integration tests mocked by default.
- Live tests require explicit opt-in.

## 9) Minimal Tool Template (Pattern)

Use the following pattern for new tools:

- Validate inputs.
- Call SDK or raw HTTP.
- Normalize output.
- Audit log the action.
- Return `{"status": "ok", "data": ...}` or standardized error.

## 10) Example Tool Additions (from this repo)

- Metadata tool (e.g., account/portal info).
- Pipelines list tool (for label to ID mapping).

## 11) Doc Update Checklist

- Update `docs/README.md` tool list.
- Update `docs/Roadmap.md` with new spec.
- Add or update ADRs if architecture decisions change.

## 12) Git Practices (Mandatory)

- Use `git-control` for all git operations (status, add, commit, push).
- Commit after each spec is implemented; keep commits small and scoped.
- Do not amend commits unless explicitly requested.
- Never reset hard or discard changes unless explicitly requested.
- Keep the working tree clean between specs when possible.

## 13) Testing & Validation

- Prefer fast unit tests for validation and routing logic.
- Mock external APIs by default; gate live tests behind explicit flags.
- Add smoke tests for new CLI scripts if reasonable.
- Document how to run tests in `docs/DEV.md` or equivalent.

## 14) Operational Hygiene

- Keep logs in `logs/` with rotation.
- Use structured logging for audit events.
- Redact PII and secrets in all logs.
- Keep CLI scripts minimal and safe.

## 15) Quickstart Commands

- Create venv: `uv venv`
- Install deps: `uv pip install -r requirements.txt`
- Run server: `uv run python -m src.server`
- Run tool: `python -m src.cli --tool <tool_name> --payload '<json>'`

## 16) Codex CLI MCP Server Install (Config)

Add an entry to `~/.codex/config.toml` for each MCP server. General pattern:

```toml
[mcp_servers.my_server]
command = "uv"
args = ["--project", "/path/to/repo", "run", "python", "-m", "src.server"]

[mcp_servers.my_server.env]
PYTHONPATH = "/path/to/repo"
MY_API_TOKEN_FILE = "${HOME}/path/to/repo/.secrets/my_api.token"
```

Guidelines:

- Keep secrets in files under a local `.secrets/` directory that is gitignored.
- Use `*_TOKEN_FILE` env vars when supported so tokens are not exposed in process lists.
- Keep server names short and descriptive (e.g., `crm`, `billing`, `support`).

## 17) Quality Bar

- Spec-driven changes only.
- Secure by default (redaction, no secrets, rate limits).
- Stable interfaces, consistent naming, clear docs.
- Commit after each spec is complete.

## 18) Optional Considerations (Nice-to-Have)

### Spec Template (Copy/Paste)

```
# Spec XXX - Title

## Goal

## Requirements

## Non-Goals

## Interfaces

## Security

## Tests

## Acceptance Criteria
```

### Error & Retry Guidance

- Define retryable error classes and backoff strategy.
- Ensure idempotency for retried writes (idempotency keys where supported).
- Surface actionable error messages without leaking sensitive data.

### Pagination & Limits

- Document pagination parameters and default limits.
- Ensure list/find tools accept `limit` + `after` and return paging info.
- Guard against unbounded requests or large payloads.

### MCP Tool Contract

- Use a consistent response envelope for all tools.
- Standardize error shape and error types.
- Document any non-standard fields or behaviors.

### Code Style & Linting

- Define formatting and linting standards (and where they live).
- Keep code style consistent across tools and scripts.

### Security Review Checklist

- Least-privilege tokens/scopes.
- Secrets redaction verified in logs.
- Input validation across all tools.
- Audit logging for sensitive operations.

### Test Matrix

- Unit tests for validation, formatting, routing.
- Integration tests with mocked APIs.
- Live tests gated by explicit opt-in.

### Release Hygiene

- Update docs + roadmap.
- Consider a changelog entry.
- Tag releases or version the server.
