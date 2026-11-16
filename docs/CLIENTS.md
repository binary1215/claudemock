# Client setup

**English** | [한국어](ko/CLIENTS.md)

[README](../README.md) · [Installation](QUICKSTART.md) · [LiteLLM registration](LITELLM-NATIVE-RESPONSES.md)

## Check these three settings first

1. **Request API:** select `-claude` aliases for Messages, or `-codex` aliases for Responses/compact.
2. **Endpoint:** clients connect to LiteLLM; LiteLLM connects to claudemock.
3. **Credential:** give the client a LiteLLM key.

| Credential | Used for |
| --- | --- |
| Claude OAuth/API credential | claudemock → Claude |
| `ADMIN_API_SECRET` | claudemock administration API |
| Proxy API key | LiteLLM → claudemock |
| LiteLLM user key | Coding client → LiteLLM |

`-claude`/`-codex` suffixes do not change the actual model or restrict which client can connect. They identify endpoint/provider configurations. LiteLLM does not convert every request to Chat Completions: this setup forwards Messages and Responses to the proxy in their respective formats.

## Claude Code — Messages

Set these variables in a separate terminal, using your actual gateway address and a key issued by that gateway:

```sh
export ANTHROPIC_BASE_URL='http://<GATEWAY_HOST>:4000'
export ANTHROPIC_AUTH_TOKEN='<your-LiteLLM-key>'
export ANTHROPIC_MODEL='claude-opus-5-5-claude'
export ANTHROPIC_DEFAULT_OPUS_MODEL='claude-opus-5-5-claude'
export ANTHROPIC_DEFAULT_SONNET_MODEL='claude-sonnet-5-5-claude'
export ANTHROPIC_DEFAULT_HAIKU_MODEL='claude-haiku-5-5-claude'
claude
```

Claude Code appends `/v1/messages`, so do not append `/v1` to its base URL. Use a dedicated terminal without conflicting provider environment variables. To select Fable explicitly, set `ANTHROPIC_MODEL=claude-fable-5-2-claude`. Do not substitute the container's Claude login token for the gateway key.

## Codex — Responses

Set `CLAUDE_PROXY_GATEWAY_KEY` to your LiteLLM key, then merge these **named profile/provider blocks** into your private Codex configuration. Do not replace the entire file or change existing permission settings.

```toml
[profiles.claudemock]
model = "claude-opus-5-5-codex"
model_provider = "claudemock"
web_search = "disabled"

[model_providers.claudemock]
name = "claudemock via LiteLLM"
base_url = "http://<GATEWAY_HOST>:4000/v1"
wire_api = "responses"
env_key = "CLAUDE_PROXY_GATEWAY_KEY"
requires_openai_auth = false
supports_websockets = false
request_max_retries = 0
stream_max_retries = 0
```

```sh
codex --profile claudemock
```

Do not use the same model's Messages alias here. The proxy needs `RESPONSES_ENABLED=true`, a persistent state key, and the qualified coding-flow settings `RESPONSES_DEVELOPER_MESSAGE_MODE=hoist` and `RESPONSES_APPLY_PATCH_MODE=validated`. Both compatibility settings default to `reject`.

Qualification covered bounded real coding workflows with Codex 0.160.0. This snippet supplies connection settings, not a complete model catalog or a guarantee of every client feature. A custom-provider metadata fallback warning may appear. See the [project tests](OPUS-PROJECT-AUTO-COMPACTION.md) for qualification conditions involving a separate catalog.

## Other clients

Responses clients use the gateway's `/v1` base URL and a `-codex` alias. Anthropic Messages clients use the root base URL and a `-claude` alias. An “OpenAI-compatible” label alone does not establish Responses support. A `/v1/chat/completions`-only client does not use the same route as this Responses deployment.

Do not assume every OpenCode/Hermes version or feature is qualified. Check the selected route and the client's tools, thinking, and replay behavior individually. [Older connection fixtures](../integration/clients/profiles/README.md) document previous pass-through qualification and are not the default for a new installation.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| `Cannot POST /v1/v1/messages` | Remove duplicate `/v1` from the Messages provider and Claude Code base URLs |
| Model not found / access denied | New alias, gateway key model allowlist, and key expiry |
| Responses disabled | Proxy `RESPONSES_ENABLED`, state key, and `-codex` deployment |
| developer / apply_patch HTTP 400 | Server `hoist`/`validated` settings and the actual request contract |
| Login required / authentication not ready | Selected credential mode, login volume, official CLI auth status |
| Upstream 400/403/429 | Identify the failing layer; renaming does not resolve account restrictions or model access denial |
| Reasoning/compact replay failure | Preserve entire returned items; check model, account, proxy key, state key, and TTL |
| CLI `count_tokens` returns 501 | Known CLI-backend limitation |

Start with the short curl examples in the README, then verify client tool calls, replay, and compaction. One HTTP 200 does not establish long-session quality or full feature equivalence.
