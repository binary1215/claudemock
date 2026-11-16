# claudemock

[English](README.md) | **한국어**

**Claude를 Anthropic Messages와 OpenAI Responses API로 연결하는 Docker 프록시.**

Claude Code, Codex 등 코딩 클라이언트를 별도 LiteLLM gateway를 통해 같은 Claude 계정에 연결합니다. 기본 백엔드는 공식 Claude CLI와 Agent SDK로 추론하며, 파일 수정·셸 실행 같은 클라이언트 도구는 해당 클라이언트에서 실행합니다.

[빠른 시작](docs/ko/QUICKSTART.md) · [클라이언트 연결](docs/ko/CLIENTS.md) · [LiteLLM 설정](docs/LITELLM-NATIVE-RESPONSES.md) · [컴팩션](docs/RESPONSES-COMPACTION.md) · [검증 기록](docs/ACCEPTANCE.md)

```text
Claude Code ── Messages ──┐
                         ├── LiteLLM ── claudemock ── 공식 Claude CLI ── Claude
Codex ─────── Responses ─┘                  │
                                    Responses 형식 변환
```

LiteLLM은 별도 컨테이너에서 인증과 모델 라우팅을 담당합니다. Responses 변환과 thinking 상태 복원은 claudemock이 담당합니다. 직접 claudemock에 연결할 수도 있습니다.

## 요구사항

- Docker와 Docker Compose v2. 이미지는 저장소 소스로 빌드합니다.
- 사용할 모델에 접근 가능한 Claude 인증 정보. 컨테이너의 공식 CLI 로그인 또는 기존 API 키/setup-token을 선택합니다.
- LiteLLM 경유 시, LiteLLM 서버에서 프록시의 공개 포트에 접근할 수 있어야 합니다.
- 기본 이미지: **Claude CLI 2.1.322 / Agent SDK 0.3.322**. 버전 변경 시 호환성을 다시 확인해야 합니다.

## 빠른 시작

```sh
git clone https://github.com/binary1215/claudemock.git
cd claudemock
```

인증 방법에 따라 하나의 설치 경로를 선택합니다.

| 원하는 방식 | 설치 가이드 | 특징 |
| --- | --- | --- |
| 컨테이너 안에서 로그인 | [공식 CLI 로그인](docs/ko/QUICKSTART.md#container-login) | 브라우저 인증 후 CLI에 코드 입력, 인증 볼륨 유지, 자동갱신 선택 가능 |
| 이미 발급한 토큰/API 키 사용 | [환경변수 인증](docs/ko/QUICKSTART.md#configured-credentials) | `compose.adapter.yml` 사용, setup-token만으로 자동갱신되지는 않음 |

기본 배포 포트 예시는 `13456`입니다. 아래 `<PROXY_HOST>`는 직접 설치한 호스트로 바꾸세요. 건강 상태 조회는 모델 호출이나 로그인 유효성 검증을 대신하지 않습니다.

```sh
curl http://<PROXY_HOST>:13456/health
```

설치 가이드에는 포트 공개, 프록시 API 키 발급, 최초 모델 호출까지 포함되어 있습니다. `.env`·인증 볼륨·DB·Responses 상태 키는 비공개로 보관하세요.

## 어떤 모델 이름을 선택하나요?

**LiteLLM으로 보내는 요청 API에 맞춰 접미사를 선택합니다.** 클라이언트 프로그램 이름으로 자동 판별하지 않습니다.

| 요청 API | 모델명 접미사 | 대표 클라이언트 |
| --- | --- | --- |
| `/v1/messages` | `-claude` | Claude Code |
| `/v1/responses` | `-codex` | Codex, Responses를 지원하는 클라이언트 |
| `/v1/responses/compact` | `-codex` | standalone compact를 호출하는 클라이언트 |

| 실제 Claude 모델 ID | Messages용 LiteLLM 이름 | Responses용 LiteLLM 이름 |
| --- | --- | --- |
| `claude-opus-5-5` | `claude-opus-5-5-claude` | `claude-opus-5-5-codex` |
| `claude-sonnet-5-5` | `claude-sonnet-5-5-claude` | `claude-sonnet-5-5-codex` |
| `claude-haiku-5-5` | `claude-haiku-5-5-claude` | `claude-haiku-5-5-codex` |
| `claude-fable-5-2` | `claude-fable-5-2-claude` | `claude-fable-5-2-codex` |

접미사는 gateway에서 등록한 별칭입니다. **프록시를 직접 호출할 때는 접미사 없는 실제 모델 ID**를 사용합니다. 같은 행의 두 이름은 같은 Claude 모델을 사용합니다. 모델 접근 가능 여부는 계정에 따라 달라질 수 있습니다.

이전 `…-han`, `…-responses-han` 이름은 새 이름으로 변경되었습니다. 기존 클라이언트의 모델명과 키별 모델 허용 목록도 함께 확인하세요.

## LiteLLM 연결

[8개 모델 설정 예제](integration/litellm/native-responses.example.yaml)를 기존 LiteLLM 설정에 병합하거나, 관리 UI에서 동일한 값을 등록합니다. 예제의 `YOUR_PROXY_HOST`와 포트를 실제 프록시 주소로 바꾸고, LiteLLM 환경변수 `CLAUDE_PROXY_API_KEY`에 **프록시에서 발급한 API 키**를 넣습니다.

| 등록 항목 | LiteLLM provider/model | LiteLLM의 upstream API base |
| --- | --- | --- |
| `claude-opus-5-5-claude` | `anthropic/claude-opus-5-5` | `http://YOUR_PROXY_HOST:13456` |
| `claude-opus-5-5-codex` | `openai/claude-opus-5-5` | `http://YOUR_PROXY_HOST:13456/v1` |

`openai/`는 LiteLLM의 Responses 전송 방식을 선택하는 값입니다. 실제 모델은 Claude입니다. Messages는 Messages로, Responses는 Responses로 프록시에 전달하고, claudemock에서 CLI 형식으로 연결합니다. LiteLLM/Codex 소스 패치나 별도 Enterprise pass-through 설정은 이 구성의 요구사항이 아닙니다.

Messages API base에 `/v1`을 덧붙이면 `/v1/v1/messages` 오류가 날 수 있습니다. Responses 등록에는 `mode: responses`와 `supported_endpoints`를 예제대로 설정하세요. 자세한 동작과 제약은 [gateway 가이드](docs/LITELLM-NATIVE-RESPONSES.md)에 있습니다.

## 호출 예제

아래 예제는 **LiteLLM 경유**입니다. Bash 계열 셸에서 `LITELLM_URL`과 `LITELLM_API_KEY`를 비공개 환경변수로 준비합니다. 예: URL은 `http://<GATEWAY_HOST>:4000`이며 끝에 `/v1`을 붙이지 않습니다.

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

Claude Code·Codex의 실제 연결 설정, 키 종류, 오류별 점검 방법은 [클라이언트 가이드](docs/ko/CLIENTS.md)를 참고하세요. `-codex`는 Chat Completions용 등록이 아닙니다.

## 지원 기능과 한계

| 기능 | 현재 지원 범위 |
| --- | --- |
| Messages / Responses | 스트리밍, 지원되는 도구 호출과 결과 재입력 |
| Thinking | 지원되는 signed/redacted thinking을 Responses의 인증된 opaque 상태로 보존 |
| 캐싱 | upstream cache 사용량 전달. 캐시 적중·실제 청구액은 보장하지 않음 |
| 컴팩션 | CLI 자동 컴팩션, 검증된 클라이언트 로컬 컴팩션, standalone `/v1/responses/compact` |
| 인증 | CLI 로그인 볼륨 또는 설정된 인증 정보, CLI 로그인 모드의 선택적 자동갱신 |
| 운영 | 프록시 API 키, 사용량/작업 관리, 선택적 별도 admin dashboard |
| `/v1/messages/count_tokens` | HTTP 백엔드에서 전달, CLI 백엔드에서는 `501` |
| `/v1/embeddings` | 별도 Ollama를 연결한 경우 선택적으로 제공 |

OpenAI API 전체 대체품은 아닙니다. Chat Completions, `store:true`, `previous_response_id`, background Responses, inline `context_management`는 지원하지 않습니다. 도구·콘텐츠·추론 옵션에도 백엔드별 제한이 있습니다. [Responses 계약](docs/RESPONSES-ADAPTER.md) · [CLI 계약](docs/CLI-INFERENCE.md)

대화 이력은 클라이언트가 소유합니다. Responses 응답의 reasoning 상태를 다음 요청에 그대로 돌려보내야 하며, worker 세션은 영구 대화 DB가 아닙니다. 컴팩션 후에는 반환된 **전체 `output` 배열**로 이전 대화 입력을 교체하고 새 입력을 덧붙입니다. 상태 키·계정·모델·프록시 키 범위와 TTL도 유지되어야 합니다. [컴팩션 사용법](docs/RESPONSES-COMPACTION.md)

검증한 Codex 0.160.0 custom-provider 구성은 로컬 컴팩션을 사용합니다. 서버에 `/responses/compact`가 있다고 Codex가 자동으로 그 경로를 사용하는 것은 아닙니다.

## 검증된 범위

- 2026-11-13 Linux 검증: **299개 검사 통과**. 실제 Opus의 도구 호출·compact·반복 compact·재시작 후 복원 6건 HTTP 200. [영수증](docs/evidence/20261113-responses-compact.json)
- 같은 검증에서 첫 입력 창은 24,869 → 3,444 JSON bytes로 줄었고, 짧은 두 번째 창은 3,909 → 3,976 bytes로 증가했습니다. 압축이 항상 크기를 줄이지는 않습니다.
- 해당 live compact 입력에는 reasoning 항목이 없어, 이 결과만으로 signed thinking의 live compact 보존을 증명하지 않습니다. 별도 scripted tests와 [실제 클라이언트 검증](docs/LITELLM-NATIVE-RESPONSES.md#verification-and-rollback)을 구분합니다.
- 파일 편집과 자동 컴팩션은 [Opus 프로젝트 테스트](docs/OPUS-PROJECT-AUTO-COMPACTION.md)를 참고하세요. 기본 최대 문맥·장기간·모든 클라이언트/모델의 동등성을 보장하지 않습니다.

문서 예제 변경 자체가 새 live 추론 테스트를 의미하지 않습니다.

## 운영과 개발 문서

| 목적 | 문서 |
| --- | --- |
| 신규 Docker 설치, 인증, 키 발급 | [빠른 시작](docs/ko/QUICKSTART.md) |
| 클라이언트 연결, 모델 선택, 오류 해결 | [클라이언트 가이드](docs/ko/CLIENTS.md) |
| 로그인·갱신·로그아웃 | [인증 수명주기](docs/CONTAINER-AUTH.md) |
| 모델별 옵션 | [모델 호환성](docs/MODEL-COMPATIBILITY.md) |
| 백업·업그레이드·롤백 | [배포 수명주기](integration/deployment/README-lifecycle.md) |
| 테스트 수행 | [검증 가이드](docs/VERIFICATION.md) |

기존 클론은 `git remote set-url origin https://github.com/binary1215/claudemock.git`으로 원격 주소를 바꿀 수 있습니다. 기본 브랜치는 `master`입니다. 로컬 폴더명을 바꿔도 기존 Compose 프로젝트명, DB·인증 볼륨과 Responses 상태 키는 유지하세요. 업그레이드 과정에서 `docker compose down -v`를 실행하지 마세요.

로컬 개발은 Node.js 22를 사용합니다.

```sh
npm ci
npm run install:cli
npm test
node --test --test-concurrency=1 integration/cli-backend/*.test.mjs
```

테스트 명령은 로컬 fixture를 사용합니다. 실제 모델·클라이언트 검증은 별도 opt-in 절차입니다. `src/`는 HTTP/변환/상태 관리, `integration/cli-backend/`는 CLI worker, `admin/`은 별도 관리 UI입니다.

## 출처와 라이선스

[MehdiMohseni82/claude-code-proxy](https://github.com/MehdiMohseni82/claude-code-proxy)의 fork입니다. 저장소 이름은 `claude-code-proxy`에서 `claudemock`으로 변경되었습니다. 설치·사용 예제를 먼저 제공하는 문서 구성은 [GPTMock](https://github.com/rapidrabbit76/GPTMock)을 참고했으며, 두 프로젝트의 API 지원 범위와 인증 구현은 다릅니다.

저장소는 MIT 라이선스이며 의존성은 각각의 라이선스와 이용 조건을 따릅니다. [Codex grammar 출처](licenses/CODEX-NOTICE.md)
