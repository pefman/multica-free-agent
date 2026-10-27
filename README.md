# multica-free-agent

CLI that lists **free OpenRouter models** with your API key, then points chosen [Multica](https://multica.ai) agents at a **local harness** plus that model.

Multica agents live on **workspaces**, not projects. This tool lists every workspace you belong to and asks which agents to change.

## Install

Needs `bash`, `curl`, `jq`, and a logged-in `multica` CLI with the daemon running.

```bash
git clone <this-repo>
./bin/multica-free-agent
```

Put `bin/` on your `PATH` if you want the command as `multica-free-agent`.

## API key

The OpenRouter key is **not** the Multica token.

Resolution order:

1. `OPENROUTER_API_KEY` in the environment
2. `~/.config/multica-free-agent/api_key` (directory mode `0700`, file mode `0600`)
3. Interactive prompt, then save to that file

The cached key is only for this CLI (listing models). For daemon-launched runs to call OpenRouter, the script can also merge `OPENROUTER_API_KEY` into each agent's Multica `custom_env`. Existing env keys are preserved (`****` on set). The key is written through a `0600` temp file, never on the process command line.

## Interactive flow

```bash
./bin/multica-free-agent
```

1. Load or prompt for the OpenRouter key
2. Pick a free model (`:free` suffix or zero prompt/completion/request price, plus `openrouter/free`)
3. Pick an **online local** harness, grouped as `provider @ machine` (opencode / omp / pi / hermes listed first)
4. Select workspaces (`a` = all)
5. Select agents (`a` = all; Mika is labelled `[mika]`)
6. Confirm whether to write `OPENROUTER_API_KEY` into each agent's `custom_env`
7. Review the old → new table, then `Proceed?`

Nothing is written until you confirm.

## Flags

```bash
./bin/multica-free-agent --list-models

./bin/multica-free-agent --dry-run \
  --model nvidia/nemotron-3-ultra-550b-a55b:free \
  --runtime opencode \
  --workspace pelagic-open-waters

./bin/multica-free-agent --yes --no-env \
  --model openrouter/free \
  --runtime opencode \
  --machine omarchy-4090 \
  --workspace fiskmas \
  --agent "Oskar (Developer)"
```

| Flag | Meaning |
|---|---|
| `--list-models` | Print free models and exit |
| `--dry-run` | Print the apply table, do not update |
| `--yes` | Apply without a confirm prompt |
| `--model ID` | OpenRouter model id |
| `--runtime PROVIDER` | `opencode`, `omp`, `pi`, `hermes`, … |
| `--machine NAME` | Defaults to the only online machine for that provider |
| `--workspace SLUG` | Repeatable; slug, name, or id |
| `--agent NAME_OR_ID` | Repeatable |
| `--env` / `--no-env` | Merge or skip `OPENROUTER_API_KEY` in `custom_env`. `--yes` defaults to `--env` |

`--yes` and `--dry-run` require `--model` and `--runtime` when stdin is not a tty.

## Model ids

OpenRouter ids look like `nvidia/nemotron-3-ultra-550b-a55b:free`.

The value written to Multica is harness-prefixed:

- `opencode` + `nvidia/foo:free` → `openrouter/nvidia/foo:free`
- `opencode` + `openrouter/free` → `openrouter/openrouter/free` (Free Models Router)

Claude, Codex, Grok, Cursor, and Copilot do not take OpenRouter ids. You can still pick them; Multica will likely reject the model.

## Limits

OpenRouter free-tier traffic is capped (currently 20 requests/min and 50/day until the account has bought credits). A whole squad on one `:free` model will hit that quickly.

## Tests

```bash
bash tests/run
```

Uses stub `curl` and `multica` under `tests/stubs/`. Does not touch your live workspaces or your real API key.
