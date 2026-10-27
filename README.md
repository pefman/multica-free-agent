# multica-free-agent

Point your [Multica](https://multica.ai) agents at a **free LLM** in under a minute.

This CLI lists free models (from the local OpenCode catalog, plus OpenRouter if you have a key), shows which of your Multica agents are online, and re-points any agents you choose onto your local harness + that model. No accounts to configure, no paid tiers — just a free model id your machine can run.

```
── Free models (22) ───────────────────────────────────────────────────────────
   1) opencode/beam-free
   2) opencode/deepseek-v4.1-flash-free
   3) opencode/glm-5.3-flash-free
   ...
Choose model [1-22]: 2

── Online local harnesses (3) ─────────
   1) omp @ omarchy-4090
   2) opencode @ omarchy-4090
   ...
```

## 60-second quick start

You need `bash`, `curl`, `jq`, a running `multica` daemon (with the CLI logged in), and ideally `opencode` on `PATH` (for OpenCode Zen free models).

```bash
git clone <this-repo>
cd multica-free-agent

# optional but recommended: shows every agent's current setup first
./bin/multica-free-agent --list-agents

# now just run it and answer the prompts
./bin/multica-free-agent
```

The interactive flow asks, in order:

0. Running with **no flags** opens the numbered menu first:
   `1) setup agents` (Enter = 1), list commands, then
   **install to your bin** (always second-to-last) and **exit** (always last).
1. **Pick a free model** (`opencode/…-free`, OpenRouter `…:free` / zero-priced, and the `openrouter/free` router)
2. **Pick an online local harness** (`provider @ machine`; `opencode` / `omp` / `pi` / `hermes` listed first)
3. **Which machines**, if the provider runs on more than one
4. **Which workspaces** (`a` = all, or comma-separated numbers)
5. **Which agents** (`a` = all; Mika agents are marked `[mika]`)
6. **Write `OPENROUTER_API_KEY`** into each agent's `custom_env`? (needed only for OpenRouter models)
7. **Review** the `old → new` table and confirm

**Nothing is changed until you answer `y` at the final `Proceed?` prompt.** Answer `n` (the default) and your agents are untouched.

## Inspect without changing anything

Every command here is read-only — safe to run anytime:

```bash
./bin/multica-free-agent --list-agents     # all agents: workspace, runtime, model, env keys, [mika] badge
./bin/multica-free-agent --list-models     # free models as TSV (id<TAB>name<TAB>context)
./bin/multica-free-agent --list-workspaces # slug / name / id
./bin/multica-free-agent --list-runtimes   # online local harnesses

# preview exactly what would change, on real data, without updating:
./bin/multica-free-agent --dry-run \
  --model opencode/deepseek-v4.1-flash-free \
  --runtime opencode \
  --workspace fiskmas --workspace nordicmcp-eu
```

`--list-agents` accepts the same filters as the setup flow, so you can look at one slice:

```bash
./bin/multica-free-agent --list-agents --workspace fiskmas --agent Mika
```

## One-shot (non-interactive) recipes

```bash
# apply to a specific agent without any prompts
./bin/multica-free-agent --yes --no-env \
  --model openrouter/free \
  --runtime opencode \
  --machine omarchy-4090 \
  --workspace fiskmas \
  --agent "Oskar (Developer)"

# preview first, then apply: swap --dry-run for --yes

# deploy the CLI into your user bin so it runs from any directory
./bin/multica-free-agent --install   # copies into ~/.local/bin (or ~/bin)
```

| Flag | Meaning |
|---|---|
| `--install` | Install the script into your user bin (prefers a dir already on `PATH`: `~/.local/bin`, `~/bin`), so `multica-free-agent` works from anywhere |
| `--list-agents` | Show each agent's config: workspace, runtime, model, `custom_env` key names (values are never printed) |
| `--list-models` | Print free models (TSV) and exit |
| `--list-workspaces` | Print workspaces (slug, name, id) and exit |
| `--list-runtimes` | Print online local runtimes and exit |
| `--dry-run` | Print the apply table, do not update |
| `--yes` | Apply without a confirm prompt |
| `--model ID` | Model id (`opencode/deepseek-v4.1-flash-free` or `nvidia/foo:free`) |
| `--runtime PROVIDER` | `opencode`, `omp`, `pi`, `hermes`, … |
| `--machine NAME` | Defaults to the only online machine for that provider |
| `--workspace SLUG` | Repeatable; slug, name, or id |
| `--agent NAME_OR_ID` | Repeatable |
| `--env` / `--no-env` | Merge or skip `OPENROUTER_API_KEY` in `custom_env`. `--yes` defaults to `--env` for OpenRouter ids and `--no-env` for `opencode/…` Zen ids |

`--yes` and `--dry-run` require `--model` and `--runtime` when stdin is not a tty (e.g. in pipes or scripts).

## API key (only needed for OpenRouter models)

The OpenRouter key is **not** the Multica token — it's only for listing OpenRouter models and for agents that fetch models from OpenRouter at run time.

Resolution order:

1. `OPENROUTER_API_KEY` in the environment
2. `~/.config/multica-free-agent/api_key` (directory mode `0700`, file mode `0600`)
3. Interactive prompt, then it's saved to that file for next time

No key at all? You can still use the whole tool — you just get the OpenCode catalog models only (all free, no key needed). When the key *is* set, listing shows real OpenRouter display names and context sizes (e.g. `131k`), and the injected `openrouter/free` router alias appears.

## About model ids

- OpenCode Zen free ids look like `opencode/deepseek-v4.1-flash-free`. They are written unchanged and need **no** OpenRouter key. Prefer `--runtime opencode` for those.
- OpenRouter ids look like `nvidia/nemotron-3-ultra-550b-a55b:free`. The value written to Multica gets an OpenRouter prefix:
  - `opencode` + `nvidia/foo:free` → `openrouter/nvidia/foo:free`
  - `opencode` + `openrouter/free` → `openrouter/openrouter/free` (Free Models Router)

Cloud harnesses (Claude, Codex, Grok, Cursor, Copilot) do not take OpenRouter ids. You can still pick them from the harness menu; Multica will likely reject the model.

## Troubleshooting

- **"No free models. Install opencode or set OPENROUTER_API_KEY."** — Install `opencode` (adds ~20 catalog models with no key needed) or provide a OpenRouter key.
- **"No online local runtimes."** — Start a local harness (e.g. run `opencode`'s server) and make sure it shows as online in the Multica enjoin CLI.
- **`--model is required in non-interactive mode`** — Running from a pipe/script: add `--model`, `--runtime`, and (for `--yes`) `--workspace`/`--agent` selectors or `--yes` answers.
- **"Refusing to apply without --yes on a non-tty."** — Safety break for scripts; add `--yes` (`--env` defaults on) or run interactively.
- **`multica-free-agent: command not found` after install** — The target bin dir isn't on your `PATH`. Add the line the installer prints, e.g. `export PATH="$HOME/.local/bin:$PATH"`.
- **"a newer version is on git: please run git pull"** — This repo copy is behind origin. `git pull`, then re-run `--install` if you use an installed copy. Rechecks are cached ~24 h; disable entirely with `MULTICA_FREE_AGENT_NO_UPDATE_CHECK=1`.

## Limits

OpenRouter's free tier is rate-capped (currently 20 requests/min and 50/day until the account has bought credits). A whole squad of agents on one `:free` model will hit that quickly — spread agents across models after their initial setup, or stick with OpenCode Zen ids that don't route through OpenRouter.

## Tests

```bash
bash tests/run
```

Uses stub `curl`, `multica`, and `opencode` under `tests/stubs/`. Does not touch your live workspaces or your real API key.
