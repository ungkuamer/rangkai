# Rangkai Packaging and Single-Command UX Proposal

This proposal describes how to turn Rangkai into a single installable command that can be run from a target Git repository, serve the packaged dashboard UI, and optionally control the continuous orchestrator from the dashboard.

The goal is a low-friction local operator workflow:

```bash
cd /path/to/project
rangkai start
```

`rangkai start` should start the FastAPI backend and packaged React dashboard for the active repository. The continuous orchestrator must remain stopped until the user starts it explicitly from the dashboard.

---

## 1. Current Shape

Rangkai already has:

- Python package source under `src/rangkai`.
- CLI entrypoint `rangkai = "rangkai.cli:main"` in `pyproject.toml`.
- Existing `rangkai dashboard` command that starts a FastAPI backend.
- React/Vite dashboard source under `dashboard/`.
- Built dashboard assets currently emitted to `dashboard/dist`.
- Repo-keyed dashboard APIs and frontend state.
- A synchronous `Orchestrator.run(repo_name, config, continuous=True)` loop.

Important implication: packaging the React assets is not enough by itself. The FastAPI app also needs to serve the packaged `index.html`, assets, and SPA fallback routes.

---

## 2. Installation and Packaging Strategy

Embed the built Vite dashboard in the Python wheel under the existing package layout:

```text
src/
  rangkai/
    __init__.py
    cli.py
    dashboard.py
    dashboard/
      dist/
        index.html
        assets/
```

### Build and Package Assets

Add a packaging step that:

1. Runs the dashboard build from `dashboard/`.
2. Copies `dashboard/dist/` to `src/rangkai/dashboard/dist/`.
3. Includes the copied files as Python package data.

`pyproject.toml` should include package data similar to:

```toml
[tool.setuptools.package-data]
rangkai = ["dashboard/dist/**/*"]
```

The build mechanism can be a small release script first, then a custom build hook only if needed. Keeping the wheel build predictable is more important than hiding all frontend build complexity inside setuptools immediately.

### Runtime Static Serving

Update the dashboard backend to serve packaged frontend assets:

- Resolve assets using `importlib.resources.files("rangkai").joinpath("dashboard/dist")`.
- Mount static assets from `dashboard/dist/assets`.
- Serve `dashboard/dist/index.html` for `/`.
- Add an SPA fallback for non-API, non-dashboard backend paths.
- Keep existing JSON/SSE routes unchanged.

This avoids requiring a separate Vite dev server for installed use.

---

## 3. Zero-Config Repository Discovery

Rangkai should support config-free startup when run inside a Git working tree. Existing explicit YAML config should continue to work and should take precedence.

### Discovery Rules

When no explicit usable config is present:

- `repo_path`: resolve with `git rev-parse --show-toplevel`, not raw CWD.
- `name`: derive from the Git repository root basename.
- `main_branch`: resolve in this order:
  1. `git symbolic-ref refs/remotes/origin/HEAD`
  2. remote default branch from `gh repo view --json defaultBranchRef` if available
  3. existing local `main`
  4. existing local `master`
  5. current branch as last resort
- `worktree_root`: default to `<repo_root>/.rangkai/worktrees`.
- `global_concurrency`: default to `1`.
- `per_prd_concurrency`: default to `1`.
- `default_harness`: use a named built-in harness.

### Built-In Harness

The plan must define the exact built-in harness before implementation. Proposed initial default:

```yaml
harnesses:
  codex:
    command: codex
    args_template:
      - exec
      - --cd
      - "{worktree_path}"
      - "--"
      - "Implement issue #{issue_number} following repository instructions."
```

If that is not the intended local agent interface, replace it before implementation. The fallback harness cannot remain abstract because readiness checks and execution errors need concrete behavior.

### Config Loading API

Add a new config loading path rather than loosening the current strict YAML parser:

- `load_config(path)` remains strict and continues to error when required YAML fields are missing.
- Add `load_config_or_discover(path, cwd)` or similar for `rangkai start`.
- If `rangkai.yaml` exists, load it and use the first or configured repo.
- If it does not exist, discover the current Git repository and synthesize an `AppConfig`.

This preserves existing CLI and test behavior while enabling zero-config startup.

### Gitignore Handling

Do not silently mutate `.gitignore` on every startup.

Safer behavior:

- Create `.rangkai/` when needed.
- Check whether `.rangkai/` is ignored.
- If not ignored, show a warning in CLI output and dashboard readiness.
- Add an explicit flag such as `--write-gitignore` to append `.rangkai/`.

Silent repository mutation should not be part of a dashboard startup command.

---

## 4. Unified Command

Add `rangkai start` as the installable, end-user command.

```bash
rangkai start [--config rangkai.yaml] [--host 127.0.0.1] [--port 8000] [--write-gitignore]
```

Behavior:

1. Load explicit config if present, otherwise discover the current Git repo.
2. Optionally write `.rangkai/` to `.gitignore` only when `--write-gitignore` is passed.
3. Create the FastAPI app with the active config.
4. Serve API routes, SSE, and packaged dashboard assets from one uvicorn process.
5. Print the local URL.

`rangkai dashboard` can remain as a compatibility alias or developer-oriented command. Long term, `rangkai start` should be the documented default.

---

## 5. Orchestrator Daemon Control

The dashboard should control the continuous orchestrator, but the current orchestrator is synchronous and not cancellable. A daemon manager is required before adding UI controls.

### Daemon Manager Requirements

Add a backend manager, likely in `src/rangkai/daemon.py`, with:

- Single active orchestrator per FastAPI process.
- Thread-safe status state: `idle`, `starting`, `running`, `stopping`, `stopped`, `failed`.
- A `threading.Event` cancellation signal.
- A lock around start/stop/status transitions.
- Captured failure details from the orchestrator thread.
- Start rejection when already running or stopping.
- Graceful shutdown from FastAPI lifespan cleanup.

### Orchestrator Cancellation

Update continuous orchestration so it can be stopped cleanly:

- Accept an optional cancellation callback/event.
- Check cancellation before each planning pass.
- Check cancellation before dispatching new work.
- Check cancellation before and after reconciliation.
- Replace long `time.sleep(15)` with cancellable waiting, for example `stop_event.wait(15)`.
- Do not kill already-running harness subprocesses in the first iteration unless that behavior is explicitly designed.

Initial stop semantics should be:

- Stop accepting new scheduling work.
- Let the current pass finish unless it reaches a cancellation checkpoint.
- Return to `idle` or `stopped` when the orchestrator exits.
- Report `stopping` while waiting.

### REST API

Add:

- `POST /api/orchestrator/start`
- `POST /api/orchestrator/stop`
- `GET /api/orchestrator/status`

Example status response:

```json
{
  "status": "running",
  "repo": "rangkai",
  "started_at": "2026-06-07T12:00:00Z",
  "stopping": false,
  "last_error": null
}
```

The manager should emit SSE events when status changes so the UI can update immediately. Polling can remain as a fallback.

---

## 6. Dashboard UI Streamlining

The dashboard should feel single-project when launched from a repository, while keeping the existing repo-keyed API shape initially.

### Keep Repo Data, Remove Repo Choice

Preserve `/api/repos` and existing repo query parameters for compatibility, but expect exactly one active repo in zero-config mode.

Frontend changes:

- Keep `effectiveSelectedRepo` internally.
- Remove the visible repository `<select>` from the sidebar.
- Show a read-only workspace summary:
  - project name
  - absolute repository path
  - main/default branch
  - worktree root
- Show a warning state when discovery created a config and `.rangkai/` is not ignored.

Avoid introducing repo-less frontend routes in the first implementation. The current API and hook structure are repo-keyed throughout, and preserving that shape reduces risk.

### Daemon Controls

Add a compact control in the sidebar or header:

- Start button when status is `idle`, `stopped`, or `failed`.
- Stop button when status is `running`.
- Disabled button or spinner when status is `starting` or `stopping`.
- Status badge with the current daemon state.
- Last error text only when status is `failed`.

The UI should use:

- `GET /api/orchestrator/status` on initial load.
- SSE updates for state changes.
- Polling fallback if SSE disconnects.

---

## 7. Scope

### In Scope

- Packaged React assets inside the Python wheel.
- FastAPI static serving for packaged dashboard assets.
- `rangkai start` command.
- Zero-config discovery for a single Git repository.
- Safe `.rangkai/` ignore warning and optional `--write-gitignore`.
- Built-in harness definition.
- Backend daemon manager.
- Cancellable continuous orchestrator loop.
- Orchestrator start/stop/status API.
- Single-project dashboard presentation.
- Focused backend and frontend tests for the new behavior.

### Out of Scope

- Multi-project dashboard sessions.
- Multiple concurrent orchestrator loops.
- Authentication, user accounts, or multi-tenancy.
- Database-backed state.
- Redesigning the scheduler algorithm.
- Killing in-flight harness subprocesses on stop.
- UI for editing harness configuration.
- Publishing automation to PyPI beyond producing a correct wheel.

---

## 8. Implementation Plan

### Phase 1: Config Discovery

1. Add discovery helpers for Git root, repo name, default branch, and worktree root.
2. Add `load_config_or_discover()`.
3. Add tests for:
   - explicit YAML still wins
   - missing YAML discovers repo
   - subdirectory startup resolves repo root
   - non-Git directory returns a clear configuration error

### Phase 2: Packaged Dashboard Serving

1. Copy built assets to `src/rangkai/dashboard/dist`.
2. Add package data config.
3. Serve packaged assets from FastAPI.
4. Add tests for:
   - `/` returns `index.html`
   - asset paths resolve
   - API routes still work
   - SPA fallback does not intercept API 404s

### Phase 3: Unified CLI

1. Add `rangkai start`.
2. Use discovered config when appropriate.
3. Keep `rangkai dashboard` for compatibility.
4. Add CLI tests for startup wiring without actually running uvicorn.

### Phase 4: Daemon Manager

1. Add a daemon manager abstraction.
2. Add cancellation support to continuous orchestrator execution.
3. Add lifecycle cleanup in FastAPI lifespan.
4. Add API endpoints.
5. Add tests for:
   - start from idle
   - duplicate start rejected
   - stop transitions to stopping
   - cancellation breaks sleep/wait promptly
   - worker exception surfaces as failed status

### Phase 5: UI Update

1. Replace sidebar repo dropdown with read-only workspace summary.
2. Add daemon status query/hook.
3. Add start/stop controls.
4. Wire SSE status updates with polling fallback.
5. Update frontend tests around single-project display and daemon controls.

### Phase 6: Release Verification

1. Build dashboard.
2. Build wheel.
3. Install in a clean environment.
4. Run `rangkai start` from a sample Git repository.
5. Verify dashboard loads without Vite.
6. Verify daemon can start, report status, and stop.

---

## 9. Main Risks

- The daemon API can lie about stoppability unless the orchestrator loop is actually cancellable.
- The installed app can still feel broken if FastAPI does not serve the packaged SPA.
- Default harness behavior can fail noisily if the expected local agent command is not installed.
- Auto-discovery from CWD can target the wrong directory unless Git root resolution is used.
- Silent `.gitignore` edits can surprise users and should stay opt-in.

Address these risks before polishing the dashboard UI.
