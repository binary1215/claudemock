# 클라이언트 연결

[README](../README.md) · [설치](QUICKSTART.md) · [LiteLLM 등록](LITELLM-NATIVE-RESPONSES.md)

## 먼저 확인할 세 가지

1. **요청 API**: Messages는 `-claude`, Responses/compact는 `-codex` 모델 별칭을 선택합니다.
2. **접속 주소**: 클라이언트는 LiteLLM 주소, LiteLLM은 claudemock 주소를 사용합니다.
3. **키 종류**: 클라이언트에는 LiteLLM 키를 전달합니다.

| 비밀값 | 사용하는 위치 |
| --- | --- |
| Claude OAuth/API 인증 정보 | claudemock → Claude |
| `ADMIN_API_SECRET` | claudemock의 관리 API 호출 |
| 프록시 API 키 | LiteLLM → claudemock |
| LiteLLM 사용자 키 | 코딩 클라이언트 → LiteLLM |

`-claude`/`-codex`는 실제 모델 종류나 클라이언트 접근 제한이 아닙니다. endpoint와 provider 설정을 구분하는 이름입니다. LiteLLM은 모든 요청을 Chat Completions로 바꾸지 않습니다. 이 구성은 Messages와 Responses를 각각 같은 형식으로 프록시까지 전달합니다.

## Claude Code — Messages

별도 터미널에 다음 변수를 설정하고 실행합니다. 실제 gateway 주소와 그곳에서 발급한 키를 사용하세요.

```sh
export ANTHROPIC_BASE_URL='http://<GATEWAY_HOST>:4000'
export ANTHROPIC_AUTH_TOKEN='<your-LiteLLM-key>'
export ANTHROPIC_MODEL='claude-opus-5-5-claude'
export ANTHROPIC_DEFAULT_OPUS_MODEL='claude-opus-5-5-claude'
export ANTHROPIC_DEFAULT_SONNET_MODEL='claude-sonnet-5-5-claude'
export ANTHROPIC_DEFAULT_HAIKU_MODEL='claude-haiku-5-5-claude'
claude
```

Claude Code가 `/v1/messages`를 붙이므로 base URL에는 `/v1`을 덧붙이지 않습니다. 기존 provider 환경변수와 혼용하지 않는 전용 터미널을 사용하세요. Fable은 `ANTHROPIC_MODEL=claude-fable-5-2-claude`로 명시 선택합니다. 위 키를 컨테이너의 Claude 로그인 토큰으로 대체하지 마세요.

## Codex — Responses

`CLAUDE_PROXY_GATEWAY_KEY`에 LiteLLM 키를 넣고 아래 **이름이 지정된 profile/provider 블록**을 개인 Codex 설정에 병합합니다. 파일 전체를 덮어쓰거나 기존 권한 설정을 바꾸지 않습니다.

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

같은 모델의 Messages용 별칭을 여기 넣지 않습니다. 프록시에는 `RESPONSES_ENABLED=true`, 영구 상태 키, 검증한 코딩 흐름을 위한 `RESPONSES_DEVELOPER_MESSAGE_MODE=hoist`, `RESPONSES_APPLY_PATCH_MODE=validated`가 필요합니다. 기본값은 앞의 두 호환 옵션 모두 `reject`입니다.

검증 대상은 Codex 0.160.0의 제한된 실제 코딩 흐름입니다. 이 snippet은 연결 설정이며 모델 catalog 전체와 모든 클라이언트 기능의 보증은 아닙니다. custom-provider 모델 메타데이터 fallback 경고가 날 수 있습니다. 별도 catalog를 사용한 검증 조건은 [프로젝트 테스트](OPUS-PROJECT-AUTO-COMPACTION.md)에 있습니다.

## 다른 클라이언트

Responses를 지원하는 클라이언트는 gateway의 `/v1` base와 `-codex` 별칭을 사용합니다. Anthropic Messages를 지원하면 root base와 `-claude` 별칭을 사용합니다. OpenAI 호환이라는 표시만으로는 Responses 지원을 알 수 없습니다. `/v1/chat/completions` 전용 클라이언트는 이 Responses 등록과 같은 경로가 아닙니다.

OpenCode/Hermes의 모든 버전·기능을 검증했다고 보지 마세요. 현재 경로와 해당 클라이언트의 tool/thinking/replay 동작을 각각 확인해야 합니다. [이전 연결 fixture](../integration/clients/profiles/README.md)는 과거 pass-through 경로를 사용한 검증 자료이며, 신규 연결의 기본 설정이 아닙니다.

## 오류 점검

| 증상 | 확인할 항목 |
| --- | --- |
| `Cannot POST /v1/v1/messages` | Messages provider와 Claude Code의 base URL에서 중복 `/v1` 제거 |
| 모델을 찾지 못함 / 권한 오류 | 새 모델 이름, gateway 키의 모델 허용 목록과 만료 확인 |
| Responses가 비활성화됨 | 프록시의 `RESPONSES_ENABLED`와 상태 키, `-codex` 등록 확인 |
| developer / apply_patch 관련 400 | 서버의 `hoist`/`validated` 설정과 실제 요청 계약 확인 |
| 로그인 필요 / 인증 준비 오류 | 선택한 인증 모드, 로그인 볼륨, 공식 CLI auth 상태 확인 |
| upstream 400/403/429 | 어느 계층의 오류인지 확인. 이름 변경은 계정 제한·모델 접근 거절을 해결하지 않음 |
| reasoning/compact 상태 재입력 실패 | 전체 반환 item 보존, 모델·계정·프록시 키·상태 키·TTL 확인 |
| CLI `count_tokens`가 501 | 현재 CLI 백엔드의 알려진 제한 |

처음에는 README의 짧은 curl 예제로 연결을 확인하고, 다음으로 클라이언트의 도구 호출·재입력·컴팩션을 확인합니다. HTTP 200 한 번은 장기 세션 품질이나 모든 기능의 동등성을 증명하지 않습니다.
