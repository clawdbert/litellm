# Hermes Agent → LiteLLM Proxy

This documents how Hermes Agent is wired to the self-hosted LiteLLM proxy so that all inference flows through our gateway (PostHog tracing, budget controls, virtual keys, model aliases).

## Architecture

```
hermes chat  →  LiteLLM proxy (port 4004)  →  upstream provider (Moonshot AI / Kimi)
                       │
                       └─ PostHog trace (every request logged)
```

Hermes speaks OpenAI-compatible REST. LiteLLM's `/v1/chat/completions` is exactly that interface, so Hermes treats the proxy as a "custom provider" — no shims needed.

## Configuration

### `~/.hermes/config.yaml` (relevant sections)

```yaml
model:
  default: moonshotai/kimi-k2.6   # model alias as defined in litellm-config.yaml
  provider: custom:litellm-proxy
  api_mode: chat_completions

custom_providers:
  - name: litellm-proxy
    base_url: https://sandboxes-1.tailf02e75.ts.net:4004/v1
    api_key: <litellm-virtual-key>   # NOT an upstream provider key; see note below
```

> **Note — `key_env` vs `api_key` (Hermes v0.7.0 bug):**  
> The published docs say to use `key_env: SOME_ENV_VAR` to reference the API key from an environment variable. In Hermes v0.7.0 the `key_env` field is parsed but never actually read during key resolution — the code falls straight through to `OPENAI_API_KEY` / `OPENROUTER_API_KEY` env vars as fallbacks (see `runtime_provider.py:_resolve_named_custom_runtime`, lines 318–323). Use `api_key:` directly in the config entry until this is fixed upstream. Since `~/.hermes/config.yaml` is not tracked by git and lives in the user's home directory this is acceptable.

### What is the `api_key` value?

It is a LiteLLM **virtual key** created in the LiteLLM proxy admin UI (`/keys/`). The proxy resolves the actual upstream provider credentials server-side. The virtual key grants scoped access (spend limits, model allowlists, user tracking) without exposing upstream keys.

## Switching models mid-session

```
/model custom:litellm-proxy:moonshotai/kimi-k2.6
/model custom:litellm-proxy:openai/gpt-4o          # if configured in litellm
```

Or pick interactively: `hermes model`

## Smoke test

```bash
hermes chat -q "Hello" -Q
```

Expected output:
```
╭─ ⚕ Hermes ───────────────────────╮
Hello! How can I help you today?
session_id: ...
```

Then verify the trace appeared in PostHog: open the PostHog dashboard and look for a new event from LiteLLM within ~10 s of the request.

## Adding a second endpoint (e.g. local dev proxy)

Append another entry to `custom_providers` in `config.yaml`:

```yaml
custom_providers:
  - name: litellm-proxy
    base_url: https://sandboxes-1.tailf02e75.ts.net:4004/v1
    api_key: <virtual-key-for-sandbox>
  - name: litellm-local
    base_url: http://localhost:4000/v1
    api_key: <virtual-key-for-local>
```

Then switch with `/model custom:litellm-local:moonshotai/kimi-k2.6`.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `401 Unauthorized` | Wrong virtual key | Regenerate key in LiteLLM admin → update `api_key` in config.yaml |
| `404 Not Found` on model | Model alias not in `litellm-config.yaml` | Add the model to the proxy config and restart LiteLLM |
| Connection refused | Proxy not running or Tailscale down | SSH to the host and check `docker compose ps` in `self-host/` |
| No PostHog trace | PostHog not configured on proxy | Check `POSTHOG_API_KEY` and `POSTHOG_HOST` env vars on the proxy host |
| Hermes sends wrong key (e.g. OpenRouter key) | `key_env` not working in v0.7.0 | Use `api_key:` directly in the `custom_providers` entry |

## Related files

- `self-host/litellm-config.yaml` — LiteLLM model list and PostHog config
- `self-host/docker-compose.yml` — proxy service definition
