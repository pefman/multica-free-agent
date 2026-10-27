# AGENTS.md

Single-file bash CLI: `bin/multica-free-agent`. Lists free models (OpenCode `opencode models | grep free` + OpenRouter API) and points Multica agents at a local harness plus that model. Needs bash, curl, jq. No build step, no linter, no CI — `bash tests/run` is the only verification gate.

## Tests

```bash
bash tests/run
```

- Run it after every script change; it must fully pass. Network/CLI stubs live in `tests/stubs/` (`curl` serves `tests/fixtures/models.json`, fake `multica`/`opencode` serve canned JSON) and are prepended to `PATH`, so tests never touch live workspaces, the real `multica` daemon, or your API key.
- The runner **sources** `bin/multica-free-agent` with `MULTICA_FREE_AGENT_LIB=1` to unit-test internal functions (`load_api_key`, `free_models_json`, `map_model_id`, `resolve_runtime_id`, …). The lib-mode guard (`if [[ "${MULTICA_FREE_AGENT_LIB:-}" == "1" ]] … return`) must stay below all function definitions and above all interactive-only code; `main "$@"` stays last. New functions go above the guard to stay testable; top-level side effects above the guard break sourcing tests.
- Test-only switches:
  - `MULTICA_FREE_AGENT_NONINTERACTIVE=1` — forces non-tty behavior.
  - `MULTICA_FREE_AGENT_TEST_NO_KEY=1` — `cli()` skips exporting a fake `OPENROUTER_API_KEY` (needed for no-key cases; must also `unset OPENROUTER_API_KEY`).
  - `MULTICA_FREE_AGENT_STUB_LOG=<file>` — stubs log every invocation; CLI tests grep the log (`agent update`, `ENVFILE {…}`) to assert side effects.
  - `MULTICA_STUB_FAIL_UPDATE=1` — exercises the failed-update path.
- Add CLI-level tests via the `cli()` helper in `tests/run` (sets PATH, fake `HOME`/`XDG_CONFIG_HOME`, key env); add unit tests in `( … )` subshells, which inherit `set -euo pipefail` from the sourced script.

## Behavior invariants (tests assert these — change script and tests together)

- Model id mapping (`map_model_id`): `opencode/…` ids pass through unchanged; `openrouter/free` → `openrouter/openrouter/free`; anything else gets the `openrouter/` prefix.
- Harness filtering: only `status == "online"` and `runtime_mode == "local"`; machine name comes from `device_info` (text before `·`) with a parenthesized-name fallback; cloud runtimes (e.g. `claude`) must never leak; OpenRouter-capable providers (`opencode omp pi hermes`) sort first.
- Runtime ids are per-workspace: always resolve with the agent's own workspace id (`resolve_runtime_id ws_id provider machine`).
- `OPENROUTER_API_KEY` is merged into `custom_env` via a `0600` temp file (`multica agent env set --custom-env-file`), never on the command line; the outgoing JSON masks existing `custom_env` values as `****`; env set only happens after a successful `agent update`.
- `--yes`/`--dry-run` without `--model` and `--runtime` fail on a non-tty. `opencode/…` (Zen) ids need no key and force `--no-env`.
- Colors are stripped unless stdout is a tty (`NO_COLOR` respected); tests grep plain text.

## Places that must stay in sync

- Flags appear in three spots: the `usage()` heredoc, the arg parser in `main`, and the README flag table — update all.
- The OpenRouter-capable provider list is written three times: `OR_CAPABLE` (top of script), the jq sort in `list_runtime_groups`, and the warning text in `main`.

## Live runs

Against a real `multica` daemon, an agent should stick to `--list-models`, `--list-agents`, `--list-runtimes`, or `--dry-run`; `--yes` mutates real agents across real workspaces.
