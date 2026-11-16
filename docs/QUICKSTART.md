# Docker quick start

**English** | [한국어](ko/QUICKSTART.md)

[README](../README.md) → Installation → [LiteLLM registration](LITELLM-NATIVE-RESPONSES.md) → [Client setup](CLIENTS.md)

This guide is for a new installation. For an existing deployment, follow the [upgrade procedure](../integration/deployment/README-lifecycle.md) instead of replacing its Compose project or volumes. Commands below use a Bash-compatible shell on Linux, macOS, or WSL.

## Preparation

```sh
git clone https://github.com/binary1215/claudemock.git
cd claudemock
```

Docker Compose v2 is required. The proxy and LiteLLM may run on separate hosts. Choose **one** authentication path below; they use different service and data-volume names.

<a id="container-login"></a>
## A. Official CLI login inside the container

### 1. Environment variables

```sh
cp .env.example .env
openssl rand -base64 32
openssl rand -base64 32
```

Generate two independent random values and put them in `ADMIN_API_SECRET` and `RESPONSES_STATE_KEY`, respectively. Never commit the actual values. Configure the following in `.env`:

```dotenv
ADMIN_API_SECRET=<first-random-value>
CLAUDE_INFERENCE_BACKEND=cli
PROXY_BUILD_TARGET=default
CLAUDE_AUTH_AUTO_REFRESH=true
RESPONSES_ENABLED=true
RESPONSES_STATE_KEY=<second-random-value>
RESPONSES_DEVELOPER_MESSAGE_MODE=hoist
RESPONSES_APPLY_PATCH_MODE=validated
```

The final two settings support the qualified Codex coding workflow. `hoist` moves developer instructions into the leading system context; `validated` checks completed output against the supported apply_patch grammar. Both default to `reject`. Neither enables constrained decoding nor grants file-editing permissions. Messages-only deployments do not need the Responses settings enabled.

Optionally set `CLAUDE_AUTH_EMAIL=you@example.com` to prefill the login email. Do not put a code or token in this variable.

### 2. Publish a port for LiteLLM on another host

The base `docker-compose.yml` exposes `app` port 3456 only inside Docker. Create this **additional file**, and set `PROXY_BIND_IP` to the proxy host's actual LAN address:

```sh
cat > compose.lan.yml <<'YAML'
services:
  app:
    environment:
      RESPONSES_DEVELOPER_MESSAGE_MODE: ${RESPONSES_DEVELOPER_MESSAGE_MODE:-reject}
    ports:
      - "${PROXY_BIND_IP:?Set the proxy host LAN IP}:${PROXY_PORT:-13456}:3456"
YAML
```

Add to `.env`:

```dotenv
PROXY_BIND_IP=<proxy-host-LAN-IP>
PROXY_PORT=13456
```

This file contains no credentials and does not configure TLS for public Internet exposure. On a shared Docker network, you can omit `ports` and use the network address. Keep the `environment` entry when using Codex's `hoist` setting: it supplies developer-mode forwarding missing from the current base Compose file.

### 3. Start and log in

The shell function below reuses the same Compose file combination:

```sh
dc() { docker compose -f docker-compose.yml -f docker-compose.cli-auth.yml -f compose.lan.yml "$@"; }
dc up -d --build app
dc exec app claude-auth login
dc exec app claude-auth status
```

1. Open the official login URL printed in the terminal.
2. Complete website sign-in. Any email sign-in code belongs on that website.
3. Paste the final OAuth callback code into the **same waiting terminal**.
4. The official CLI stores credentials in the `claude-auth` volume.

The proxy may start before login, but requests can fail until credentials are ready. After login, it reads the saved credentials. Automatic refresh uses the official CLI in `claude_cli` authentication mode. Run `dc exec app claude-auth refresh` for a manual due check, or `dc exec app claude-auth logout` to log out. [Detailed lifecycle](CONTAINER-AUTH.md)

These commands start only `app`, not the admin dashboard, nginx, or Ollama. Application data persists in `app-data`; authentication persists in `claude-auth`.

<a id="configured-credentials"></a>
## B. Existing credentials through environment variables

```sh
cp .env.adapter.example .env
```

Set `PROXY_BIND_IP` and `ADMIN_API_SECRET` in `.env`. Supply the official `claude setup-token` result through `CLAUDE_CODE_OAUTH_TOKEN`, or configure `ANTHROPIC_API_KEY`. For Responses, also add the Responses settings from section A and an independent state key.

```sh
docker compose -f compose.adapter.yml up -d --build
docker compose -f compose.adapter.yml ps
```

The service is `claude-proxy`, the default published port is 13456, and the data volume is `claude-proxy-data`. The default inference backend is still CLI. `UPSTREAM_AUTH_SOURCE=configured` selects credentials in this order: stored database credential → OAuth environment variable → API-key environment variable.

Setting a setup-token environment variable does not create a login volume with a refresh token. Use path A for managed refresh. Do not apply A's `app` overlay directly to this `claude-proxy` service.

To explicitly select a CLI-free HTTP relay, set both `CLAUDE_INFERENCE_BACKEND=http` and `PROXY_BUILD_TARGET=http`, then rebuild. Standalone Responses compact is unavailable in that mode.

## Create a proxy API key and verify the connection

These steps apply to both A and B. Privately prepare the following shell environment variables:

- `PROXY_URL`: `http://<PROXY_HOST>:13456`, without a trailing `/v1`.
- `ADMIN_API_SECRET`: the administration secret configured in the container.

```sh
curl "$PROXY_URL/health"
curl "$PROXY_URL/api/admin/keys" \
  -H "Authorization: Bearer $ADMIN_API_SECRET" \
  -H 'Content-Type: application/json' \
  -d '{"name":"litellm"}'
```

Run key creation **once**. Privately save the successful response's `key` value. The full key is returned when created; the database retains its hash and display prefix. Set `CLAUDE_PROXY_API_KEY` to this key for subsequent requests.

```sh
curl "$PROXY_URL/v1/messages" \
  -H "Authorization: Bearer $CLAUDE_PROXY_API_KEY" \
  -H 'anthropic-version: 2023-06-01' \
  -H 'Content-Type: application/json' \
  -d '{"model":"claude-opus-5-5","max_tokens":2048,"messages":[{"role":"user","content":"Reply with OK."}]}'
```

This is a **direct proxy request**, so the model name has no `-claude`/`-codex` suffix. It makes a real model request. HTTP 200 verifies a single connection, not tools, compaction, or long-session behavior.

Next, apply the [eight-model LiteLLM configuration](../integration/litellm/native-responses.example.yaml) and proceed to [client setup](CLIENTS.md). If an existing gateway uses named credentials instead of environment-based keys, you can retain that administration method.

## Values to preserve

Preserve the database volume, authentication volume (A), Responses state key, and existing Compose project name. Generating a new state key or logging into another account can invalidate opaque state from earlier conversations. `/health` and `claude-auth status` report local readiness, not remote model entitlement.
