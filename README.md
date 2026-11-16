# claudemock

**English** | [한국어](README.ko.md)

**A Docker proxy connecting Claude to Anthropic Messages and OpenAI Responses clients.**

Connect coding clients such as Claude Code and Codex to the same Claude account through a separate LiteLLM gateway. The default backend runs inference through the official Claude CLI and Agent SDK. Client tools, including file edits and shell commands, execute in the calling client.

[Quick start](docs/QUICKSTART.md) · [Client setup](docs/CLIENTS.md) · [LiteLLM configuration](docs/LITELLM-NATIVE-RESPONSES.md) · [Compaction](docs/RESPONSES-COMPACTION.md) · [Acceptance records](docs/ACCEPTANCE.md)

```text
Claude Code ── Messages ──┐
                         ├── LiteLLM ── claudemock ── Claude CLI ── Claude
Codex ─────── Responses ─┘                  │
                                    Responses adapter
```

LiteLLM handles authentication and model routing in a separate container. claudemock handles Responses translation and thinking-state restoration. Clients can also connect directly to claudemock.

## Requirements

- Docker and Docker Compose v2. Build the image from this repository.
- Claude credentials with access to the selected model. Choose official CLI login inside the container or an existing API key/setup-token.
- For gateway use, LiteLLM must be able to reach the proxy's published port.
- Default image: **Claude CLI 2.1.322 / Agent SDK 0.3.322**. Revalidate compatibility when changing versions.

## Quick start

```sh
git clone https://github.com/binary1215/claudemock.git
cd claudemock
```

Choose one installation path based on how you want to authenticate.

| Authentication | Guide | Behavior |
| --- | --- | --- |
| Log in inside the container | [Official CLI login](docs/QUICKSTART.md#container-login) | Browser authorization, code entry in the CLI, persistent auth volume, optional automatic refresh |
| Supply an existing token/API key | [Environment credentials](docs/QUICKSTART.md#configured-credentials) | Uses `compose.adapter.yml`; a setup-token alone does not enable automatic refresh |

The deployment examples publish port `13456`. Replace `<PROXY_HOST>` with your host. A health check is not an inference request or remote login validation.

```sh
curl http://<PROXY_HOST>:13456/health
```

The installation guide covers port publishing, proxy API key creation, and a first model request. Keep `.env`, authentication volumes, databases, and the Responses state key private.

## Which model name should I use?

**Select the suffix according to the API you call through LiteLLM.** It is not selected automatically from the client application's name.

| Request API | Model suffix | Typical client |
| --- | --- | --- |
| `/v1/messages` | `-claude` | Claude Code |
| `/v1/responses` | `-codex` | Codex or another Responses client |
| `/v1/responses/compact` | `-codex` | A client explicitly calling standalone compact |

| Native Claude model ID | LiteLLM Messages alias | LiteLLM Responses alias |
| --- | --- | --- |
| `claude-opus-5-5` | `claude-opus-5-5-claude` | `claude-opus-5-5-codex` |
| `claude-sonnet-5-5` | `claude-sonnet-5-5-claude` | `claude-sonnet-5-5-codex` |
| `claude-haiku-5-5` | `claude-haiku-5-5-claude` | `claude-haiku-5-5-codex` |
| `claude-fable-5-2` | `claude-fable-5-2-claude` | `claude-fable-5-2-codex` |

These suffixes are gateway aliases. **Direct proxy requests use the native model ID without a suffix.** Both aliases in a row use the same Claude model. Model availability depends on the account.

The previous `…-han` and `…-responses-han` aliases have been renamed. Update existing client model names and check key-specific model allowlists as well.

## Connect LiteLLM

Merge the [eight-model configuration example](integration/litellm/native-responses.example.yaml) into your existing LiteLLM configuration, or register equivalent values through its administration UI. Replace `YOUR_PROXY_HOST` and the port with your proxy endpoint. Set LiteLLM's `CLAUDE_PROXY_API_KEY` environment variable to an **API key issued by the proxy**.

| Deployment alias | LiteLLM provider/model | Upstream API base in LiteLLM |
| --- | --- | --- |
| `claude-opus-5-5-claude` | `anthropic/claude-opus-5-5` | `http://YOUR_PROXY_HOST:13456` |
| `claude-opus-5-5-codex` | `openai/claude-opus-5-5` | `http://YOUR_PROXY_HOST:13456/v1` |

`openai/` selects LiteLLM's Responses transport; the actual model is Claude. Messages and Responses reach the proxy in their respective formats, and claudemock adapts them to the CLI. This configuration requires neither LiteLLM/Codex source patches nor a separate Enterprise pass-through endpoint.

Appending `/v1` to the Messages API base can produce `/v1/v1/messages` errors. For Responses deployments, set `mode: responses` and `supported_endpoints` as shown in the example. See the [gateway guide](docs/LITELLM-NATIVE-RESPONSES.md) for behavior and limitations.

## Usage examples

These examples go **through LiteLLM**. In a Bash-compatible shell, privately set `LITELLM_URL` and `LITELLM_API_KEY`. For example, use `http://<GATEWAY_HOST>:4000` as the URL, without a trailing `/v1`.

Messages:

```sh
curl "$LITELLM_URL/v1/messages" \
  -H "Authorization: Bearer $LITELLM_API_KEY" \
  -H 'anthropic-version: 2023-06-01' \
  -H 'Content-Type: application/json' \
  -d '{"model":"claude-opus-5-5-claude","max_tokens":2048,"messages":[{"role":"user","content":"Reply with OK."}]}'
```

Responses:

```sh
curl "$LITELLM_URL/v1/responses" \
  -H "Authorization: Bearer $LITELLM_API_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"model":"claude-opus-5-5-codex","input":"Reply with OK.","store":false}'
```

See the [client guide](docs/CLIENTS.md) for Claude Code/Codex settings, credential types, and troubleshooting. The `-codex` deployment is not a Chat Completions backend.

## Features and limits

| Feature | Current support |
| --- | --- |
| Messages / Responses | Streaming, supported tool calls and tool-result continuation |
| Thinking | Supported signed/redacted thinking preserved in authenticated opaque Responses state |
| Caching | Upstream cache usage reported; cache hits and invoice amounts are not guaranteed |
| Compaction | CLI automatic compaction, qualified client-local compaction, standalone `/v1/responses/compact` |
| Authentication | CLI login volume or configured credentials; optional automatic refresh in CLI-login mode |
| Operations | Proxy API keys, usage/task administration, optional separate admin dashboard |
| `/v1/messages/count_tokens` | Forwarded in HTTP mode; `501` in CLI mode |
| `/v1/embeddings` | Optional, with a separately connected Ollama instance |

This is not a complete OpenAI API replacement. Chat Completions, `store:true`, `previous_response_id`, background Responses, and inline `context_management` are unsupported. Tools, content types, and inference controls also have backend-specific limits. [Responses contract](docs/RESPONSES-ADAPTER.md) · [CLI contract](docs/CLI-INFERENCE.md)

The client owns conversation history. Return Responses reasoning state unchanged on later requests; worker sessions are not a durable conversation database. After compaction, replace the old conversational input with the **entire returned `output` array**, then append new input. Preserve the state key, account, model, proxy-key scope, and TTL requirements. [Compaction guide](docs/RESPONSES-COMPACTION.md)

The qualified Codex 0.160.0 custom-provider configuration uses local compaction. Exposing `/responses/compact` on the server does not automatically make Codex call it.

## Verified scope

- Linux qualification on 2026-11-13: **299 checks passed**. Six live Opus requests covering tools, compact, repeated compact, and replay after a restart returned HTTP 200. [Receipt](docs/evidence/20261113-responses-compact.json)
- In that run, the first input window shrank from 24,869 to 3,444 JSON bytes; a shorter second window grew from 3,909 to 3,976 bytes. Compaction does not always reduce size.
- The live compact input contained no reasoning item, so that result does not establish live signed-thinking preservation across compact. Distinguish scripted tests from [real-client qualification](docs/LITELLM-NATIVE-RESPONSES.md#verification-and-rollback).
- See the [Opus project tests](docs/OPUS-PROJECT-AUTO-COMPACTION.md) for file editing and automatic compaction. These tests do not establish equivalence at the default maximum context, over extended operation, or across every client/model.

Updating documentation examples does not constitute a new live inference test.

## Operations and development

| Task | Guide |
| --- | --- |
| New Docker installation, authentication, key creation | [Quick start](docs/QUICKSTART.md) |
| Client configuration, model selection, troubleshooting | [Client guide](docs/CLIENTS.md) |
| Login, refresh, logout | [Authentication lifecycle](docs/CONTAINER-AUTH.md) |
| Model-specific options | [Model compatibility](docs/MODEL-COMPATIBILITY.md) |
| Backup, upgrade, rollback | [Deployment lifecycle](integration/deployment/README-lifecycle.md) |
| Running checks | [Verification guide](docs/VERIFICATION.md) |

Existing clones can update their remote with `git remote set-url origin https://github.com/binary1215/claudemock.git`. The default branch is `master`. If you rename a local checkout, retain the existing Compose project name, database/auth volumes, and Responses state key. Do not run `docker compose down -v` as an upgrade step.

For local development, use Node.js 22.

```sh
npm ci
npm run install:cli
npm test
node --test --test-concurrency=1 integration/cli-backend/*.test.mjs
```

These commands use local fixtures. Live model/client qualification is a separate opt-in procedure. `src/` contains HTTP, translation, and state management; `integration/cli-backend/` contains the CLI worker; `admin/` is the separate administration UI.

## Attribution and license

This is a fork of [MehdiMohseni82/claude-code-proxy](https://github.com/MehdiMohseni82/claude-code-proxy), renamed from `claude-code-proxy` to `claudemock`. The documentation's installation-and-examples-first structure was informed by [GPTMock](https://github.com/rapidrabbit76/GPTMock). The projects have different API capabilities and authentication implementations.

This repository uses the MIT license; dependencies retain their own licenses and terms. [Codex grammar attribution](licenses/CODEX-NOTICE.md)
