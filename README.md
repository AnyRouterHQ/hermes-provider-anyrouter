# hermes-provider-anyrouter

[![lint-test](https://github.com/anyrouter-dev/hermes-provider-anyrouter/actions/workflows/lint-test.yaml/badge.svg)](https://github.com/anyrouter-dev/hermes-provider-anyrouter/actions/workflows/lint-test.yaml)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

Use [AnyRouter](https://anyrouter.dev) as a model provider in
[Hermes Agent](https://github.com/NousResearch/hermes-agent). One API key, any
model, with routing and fallback handled by AnyRouter.

The plugin registers `anyrouter` over AnyRouter's OpenAI-compatible endpoint
(`https://anyrouter.dev/api/v1`). It touches no Hermes core files: discovery,
credentials, `hermes doctor` and `--provider` all wire up from the provider
registry.

## Install

```bash
# From the Hermes plugin catalog (recommended)
hermes plugins install anyrouter

# From git
hermes plugins install https://github.com/anyrouter-dev/hermes-provider-anyrouter --subdir anyrouter

# As a pip package (found through the hermes_agent.plugins entry point)
pip install git+https://github.com/anyrouter-dev/hermes-provider-anyrouter.git
```

Check it:

```bash
hermes plugins list    # `anyrouter` should be listed and enabled
hermes doctor          # provider health check
```

## Quick start

1. Get an API key at <https://anyrouter.dev>.
2. Add it to `~/.hermes/.env`:

   ```
   ANYROUTER_API_KEY=sk-ar-...
   ```

3. Pick a model:

   ```bash
   hermes model                                    # interactive picker, choose anyrouter
   hermes --provider anyrouter -m anyrouter/hermes
   ```

   Or make it the default in `~/.hermes/config.yaml`:

   ```yaml
   model:
     provider: anyrouter
     default: anyrouter/hermes
   ```

## Configuration

| Variable | Required | Purpose |
|---|---|---|
| `ANYROUTER_API_KEY` | yes | Your AnyRouter API key |
| `ANYROUTER_BASE_URL` | no | Override the endpoint (proxy or self-hosted relay) |
| `ANYROUTER_APP_ATTRIBUTION` | no | Set to `1` to send app-attribution headers, so Hermes traffic is credited in AnyRouter's public app rankings. Off by default. |

## Models

| Model | Notes |
|---|---|
| `anyrouter/hermes` | Auto-routes across top tool-calling models, with fallback. Best default for agents. |
| `anyrouter/auto` | Auto-routes across coding and vision-capable models. |
| `x-ai/grok-4.7` | Direct vendor route. |
| `moonshotai/kimi-k3` | 1M context, agentic. |
| `deepseek/deepseek-v4-flash` | Fast, reasoning-capable. |

`hermes model` always shows the live list, read from the AnyRouter catalog.

## Features

- **Tool-capable picker.** Only routes that advertise `function-calling` or
  `tools` in AnyRouter's `/api/v1/models` catalog are offered. If the catalog
  is unreachable, or `ANYROUTER_BASE_URL` points at a relay, the plugin falls
  back to the generic `/v1/models` list.
- **Request shaping** to AnyRouter's API contract:
  - `extra_body.session_id`: sticky sessions in the Request Logs dashboard.
  - `extra_body.provider`: routing preferences (`only`, `ignore`, `order`,
    `sort`, `allow_fallbacks`, `max_price`).
  - `extra_body.reasoning`: reasoning effort, translated to each upstream's
    native thinking control.
- **Cost estimates** for `anyrouter/*` meta-models, from catalog pricing
  (`input_per_1m` / `output_per_1m`). The catalog is cached per API key when
  the model list is fetched, so cost lookups never hit the network. Other
  models use Hermes' generic pricing. Meta-models bill each hop at the
  underlying model's rate, so the figure is an estimate, not an invoice.
- **No tagging unless you opt in.** Attribution headers are off by default.

## Requirements

- Hermes Agent with the model-provider plugin system
  (`providers.register_provider` and `providers.base.ProviderProfile`).
  Verified on 0.21.5.
- Python 3.11+ when installed with pip.

There is no Hermes version pin. Every contact point with core
(`_supported_kwargs`, `_inherited_fetch`, `hermes_cli.urllib_security`,
`agent.usage_pricing.CostResult`) is resolved by introspection: newer fields
are used when present and skipped on older builds.

## Troubleshooting

| Symptom | Check |
|---|---|
| `unknown provider: anyrouter` | Run `hermes plugins list` and make sure the plugin is enabled. New sessions pick it up. |
| `Model 'x' not found` | Run `hermes model`. The picker lists only routes AnyRouter serves with tool-calling. |
| Catalog unreachable | The picker uses a built-in model list. Check your network and `ANYROUTER_BASE_URL`. |
| Upstream 429 or 402 | The upstream is rate-limited or out of credits. Retry, or pick another model. |

Logs are in `~/.hermes/logs/`; `hermes logs --follow` shows requests live.

## Development

```bash
pip install -e . ruff==0.17.0
ruff check anyrouter tests && ruff format --check anyrouter tests
python -m unittest discover -s tests -v
```

Tests run offline. `tests/conftest_stub.py` stubs the Hermes `providers`
runtime, including a legacy profile base, so modern and older builds are both
covered.

## Links

- Docs: <https://docs.anyrouter.dev/guides/hermes-agent>
- AnyRouter MCP server (`list_models`, `get_credits`, key management from your
  agent): `https://anyrouter.dev/api/v1/mcp`. Add it under `mcp_servers` in
  `~/.hermes/config.yaml`.
- Issues: <https://github.com/anyrouter-dev/hermes-provider-anyrouter/issues>

## License

MIT. See [LICENSE](LICENSE).
