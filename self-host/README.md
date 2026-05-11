# Self-hosted LiteLLM gateway (auth2api + PostHog + Postgres)

Three Docker services on an internal bridge network:

- **auth2api** — translates a Claude / Codex subscription OAuth session into an OpenAI-compatible endpoint. Internal only.
- **db** — Postgres 16. Backs LiteLLM's virtual keys, spend tracking, and DB-managed model list.
- **litellm** — gateway on host port `4000`. Routes to `auth2api` over the internal Docker network. Streams traces to PostHog.

```
client → 127.0.0.1:4000 (litellm) ──→ auth2api:8317  ──→ Anthropic OAuth
                                  └─→ db:5432
                                  └─→ PostHog (traces)
```

LiteLLM never talks to Anthropic or OpenAI directly. It talks to `auth2api`, which holds the OAuth credentials.

## Prerequisites

- Docker Desktop, OrbStack, Docker Engine, **or Podman** with Compose v2 / `docker compatibility` mode.
- A PostHog account (free tier).
- An active Claude (Anthropic) or ChatGPT subscription.
- `openssl` and `curl` on the host.

## Step 1 — PostHog project key

1. Sign up or log in at <https://posthog.com>.
2. Create or select a project.
3. **Project Settings → Project API Key**. Copy the `phc_…` key.
4. Note your region:
   - US cloud → `https://us.i.posthog.com`
   - EU cloud → `https://eu.i.posthog.com`
   - Self-hosted → your URL.

## Step 2 — Generate keys

```bash
openssl rand -hex 32 | sed 's/^/sk-/'    # LITELLM_MASTER_KEY
openssl rand -hex 32 | sed 's/^/a2a-/'   # AUTH2API_PASSTHROUGH_KEY
openssl rand -base64 24                  # POSTGRES_PASSWORD (avoid @ : / & to skip URL encoding)
```

## Step 3 — Create `.env`

```bash
cd self-host
cp .env.example .env
```

Edit `.env` and fill in every value. **All five secrets must be set before the first `podman compose up`.**

| Var | Source |
|---|---|
| `POSTHOG_API_KEY` | Step 1 |
| `POSTHOG_API_URL` | Step 1 |
| `LITELLM_MASTER_KEY` | Step 2 |
| `AUTH2API_PASSTHROUGH_KEY` | Step 2 |
| `POSTGRES_PASSWORD` | Step 2 |
| `DATABASE_URL` | derived — see below |

`DATABASE_URL` must percent-encode reserved characters in the password. Common substitutions:

| Char | Encoded |
|---|---|
| `@` | `%40` |
| `:` | `%3A` |
| `/` | `%2F` |
| `&` | `%26` |
| `?` | `%3F` |
| ` ` | `%20` |

Example: `POSTGRES_PASSWORD=hello@world` → `DATABASE_URL=postgresql://llmproxy:hello%40world@db:5432/litellm`.

> **Postgres password is immutable after first init.** The `db` service runs `initdb` only when its data volume is empty. After that, `POSTGRES_PASSWORD` changes are ignored. To rotate, see "Reset DB" below.

Confirm `.env` is gitignored:

```bash
git check-ignore .env   # should print: .env
```

## Step 4 — Build images

```bash
podman compose build
```

First run pulls the LiteLLM image (pinned by digest), clones auth2api at the pinned commit, and `npm install`+`tsc`. Subsequent builds are cached.

## Step 5 — One-time OAuth logins

auth2api needs OAuth credentials before the gateway can serve requests. Each subscription provider has its own login. The login flow opens a browser, you authorize, and Anthropic/OpenAI redirects to `http://localhost:54545/callback`.

Because auth2api binds the callback listener to `127.0.0.1:54545` (not `0.0.0.0`), standard port-publishing won't reach it. We use a profile-gated `auth2api-login` service that runs with `network_mode: host`, so the container shares the host's loopback. The daemon-mode `auth2api` continues to use the bridge network.

> **Port 54545 must be free on the host.** The OAuth callback briefly binds it during each login. Verify with:
> ```bash
> ss -tlnp 2>/dev/null | grep ':54545'   # must be empty
> ```

### 5a — Anthropic (Claude)

```bash
podman compose --profile login run --rm -it auth2api-login --login
```

What you'll see:

1. The CLI prints an `https://claude.ai/oauth/authorize?...` URL.
2. Open it in your browser. Sign in with the Anthropic account that holds your Claude subscription.
3. Approve the requested scopes.
4. Browser redirects to `http://localhost:54545/callback`. The page shows "Login Successful". Close the tab.
5. Terminal prints account email and the path to the saved token.

### 5b — OpenAI (Codex / ChatGPT subscription)

```bash
podman compose --profile login run --rm -it auth2api-login --login --provider=codex
```

Same flow as 5a but with your OpenAI account that holds ChatGPT Plus/Pro.

### Verify both logins persisted

```bash
ls auth2api-data/
# expect: claude-<email>.json AND codex-<email>.json
```

If a file is missing, re-run that provider's login. Tokens live in `./auth2api-data/` on the host (gitignored), bind-mounted into the container at `/data`.

## Step 6 — Start the stack

```bash
podman compose up -d
podman compose ps
```

Healthchecks gate the startup order: `db` must report ready (Postgres accepting queries), `auth2api` must be listening, and only then does `litellm` start. Expect ~30–60 seconds before `litellm` reaches `healthy`.

Watch progress:

```bash
podman compose ps        # status column should reach "healthy" for all three
podman compose logs litellm  | tail -50
podman compose logs auth2api | tail -50
podman compose logs db       | tail -20
```

If `litellm` stays in `starting` after 2 minutes, check its logs for Prisma migration errors against the DB.

## Step 7 — Verify the gateway

Load the master key:

```bash
export $(grep '^LITELLM_MASTER_KEY=' .env | xargs)
```

Send a test request:

```bash
curl -sS http://127.0.0.1:4000/v1/chat/completions \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "claude-sonnet-4-6",
    "messages": [{"role": "user", "content": "Reply with the single word: ok"}]
  }'
```

Expected: a JSON response containing `"ok"`.

## Step 8 — Verify PostHog ingestion

1. PostHog → **Activity → Live events**.
2. Within ~30 s of the curl above, an event should appear with the model name, latency, input, and output.
3. If nothing arrives within 2 minutes:
   - `podman compose logs litellm | grep -i posthog`
   - Check the region URL matches your project (US vs. EU).
   - Confirm the container has outbound HTTPS.

## Models exposed

| `model` value | Routes via auth2api to | Configured where |
|---|---|---|
| `claude-opus-4-7` | Claude OAuth (Anthropic) | `litellm-config.yaml` |
| `claude-sonnet-4-6` | Claude OAuth (Anthropic) | `litellm-config.yaml` |
| `gpt-5.5` | Codex OAuth (OpenAI ChatGPT) | `litellm-config.yaml` |
| `gpt-5.4` | Codex OAuth (OpenAI ChatGPT) | `litellm-config.yaml` |
| `claude-haiku-4-5-20251001` | (unconfigured) | Add via Admin UI after first run — see below |

**Codex model selection.** Plain `gpt-5` is rejected by Codex for ChatGPT-account users with "model not supported when using Codex". The accepted models advertised by auth2api's `/v1/models` are: `gpt-5.5`, `gpt-5.4`, `gpt-5.4-mini`, `gpt-5.3-codex`, `gpt-5.2`. We expose `gpt-5.5` (latest). Edit `litellm-config.yaml` if you need a different one.

### Adding the haiku slot via UI

`STORE_MODEL_IN_DB=True` is set, so models can be added at runtime via the LiteLLM Admin UI:

1. Browse to <http://127.0.0.1:4000/ui>.
2. Sign in with `LITELLM_MASTER_KEY`.
3. **Models → Add Model** with `model_name=claude-haiku-4-5-20251001`, point `litellm_params.model` and `api_base` at your open-weight backend (Ollama, vLLM, etc.).

Models added via UI persist in Postgres and survive restarts.

## Updating

```bash
podman compose pull              # pulls a refreshed litellm digest if you bump the Dockerfile
podman compose build --no-cache  # rebuilds auth2api at its pinned SHA
podman compose up -d
```

To bump:
- LiteLLM: `docker buildx imagetools inspect ghcr.io/berriai/litellm:main-stable` → copy the new digest into `Dockerfile`.
- auth2api: edit `AUTH2API_SHA` in `Dockerfile.auth2api`. Latest at <https://github.com/amazingang/auth2api/commits/main>.

## Reset DB (rotate Postgres password)

The `POSTGRES_PASSWORD` is baked into the data volume at first init. To rotate, you must destroy the volume and lose all virtual keys, spend logs, and DB-stored models.

```bash
podman compose down                                # stops services
docker volume rm litellm_self_host_postgres_data   # destroys data
# Edit .env: new POSTGRES_PASSWORD + matching DATABASE_URL
podman compose up -d                               # reinitializes
```

## Teardown

```bash
podman compose down              # stop + remove containers (keeps Postgres volume + OAuth files)
podman compose down -v --rmi local   # also delete volume (DESTROYS DB) and locally-built images
```

OAuth credentials in `./auth2api-data/` survive teardown. `rm -rf auth2api-data/*.json` to revoke locally; revoke at the provider for full removal.

## Remote access

If you need this gateway reachable from another machine, front it with [Tailscale](https://tailscale.com). Do **not** change the `127.0.0.1:4000` bind to `0.0.0.0`.
