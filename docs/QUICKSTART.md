# Docker 빠른 시작

[README](../README.md) → 설치 → [LiteLLM 등록](LITELLM-NATIVE-RESPONSES.md) → [클라이언트 연결](CLIENTS.md)

새 설치 기준입니다. 기존 배포에는 다른 Compose 프로젝트/볼륨을 덧씌우지 말고 [업그레이드 절차](../integration/deployment/README-lifecycle.md)를 사용하세요. 아래 명령은 Linux/macOS/WSL의 Bash 계열 셸 기준입니다.

## 준비

```sh
git clone https://github.com/binary1215/claudemock.git
cd claudemock
```

Docker Compose v2가 필요합니다. 프록시를 실행할 호스트와 LiteLLM 호스트는 서로 달라도 됩니다. 아래 두 인증 경로 중 **하나만** 선택하세요. 둘은 서비스명과 데이터 볼륨이 다릅니다.

<a id="container-login"></a>
## A. 컨테이너에서 공식 CLI 로그인

### 1. 환경변수

```sh
cp .env.example .env
openssl rand -base64 32
openssl rand -base64 32
```

서로 다른 두 난수를 생성하여 각각 `ADMIN_API_SECRET`, `RESPONSES_STATE_KEY`에 넣습니다. 실제 값은 Git에 올리지 않습니다. `.env`에서 다음 값을 설정합니다.

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

마지막 두 설정은 검증한 Codex 코딩 흐름에 사용됩니다. `hoist`는 developer 지시문을 앞쪽 system 문맥으로 옮기고, `validated`는 알려진 apply_patch 문법의 완성된 출력을 검사합니다. 기본값은 둘 다 `reject`이며, 제약 디코딩이나 파일 수정 권한 부여가 아닙니다. Messages만 쓴다면 Responses 관련 설정을 켤 필요가 없습니다.

선택적으로 `CLAUDE_AUTH_EMAIL=you@example.com`을 지정하면 로그인 화면의 이메일을 미리 채웁니다. 코드나 토큰을 이 변수에 넣지 않습니다.

### 2. 별도 호스트의 LiteLLM에 포트 공개

기본 `docker-compose.yml`의 `app`은 3456 포트를 Docker 내부에만 노출합니다. 다음 **추가 파일**을 만들고 `PROXY_BIND_IP`를 프록시 호스트의 실제 LAN IP로 지정합니다.

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

`.env`에 추가합니다.

```dotenv
PROXY_BIND_IP=<proxy-host-LAN-IP>
PROXY_PORT=13456
```

이 파일에는 비밀값이 없습니다. 공개 인터넷용 TLS 구성은 포함하지 않습니다. 같은 Docker 네트워크로 연결하는 배포라면 `ports` 항목을 생략하고 네트워크 주소를 사용할 수 있습니다. 위 `environment` 항목은 현재 기본 Compose에서 빠진 developer 모드 전달을 보완하므로 Codex용 `hoist` 사용 시 유지하세요.

### 3. 실행하고 로그인

아래 함수는 같은 Compose 파일 조합을 반복해서 사용하기 위한 셸 단축 함수입니다.

```sh
dc() { docker compose -f docker-compose.yml -f docker-compose.cli-auth.yml -f compose.lan.yml "$@"; }
dc up -d --build app
dc exec app claude-auth login
dc exec app claude-auth status
```

1. 터미널에 나온 공식 로그인 URL을 브라우저에서 엽니다.
2. 사이트 로그인을 마칩니다. 이메일 로그인 번호는 해당 사이트에 입력합니다.
3. 마지막 OAuth callback 코드는 **대기 중인 같은 터미널**에 붙여넣습니다.
4. 공식 CLI가 `claude-auth` 볼륨에 인증 정보를 저장합니다.

프록시가 먼저 실행되어도 로그인 전 요청은 준비되지 않은 인증 상태로 실패할 수 있습니다. 로그인 후에는 인증 정보를 읽어 사용합니다. 자동갱신은 `claude_cli` 인증 모드에서 공식 CLI에 맡깁니다. 수동 점검은 `dc exec app claude-auth refresh`, 로그아웃은 `dc exec app claude-auth logout`입니다. [상세 수명주기](CONTAINER-AUTH.md)

이 명령은 `app`만 시작합니다. admin dashboard, nginx, Ollama를 함께 띄우는 명령이 아닙니다. 데이터는 `app-data`, 인증은 `claude-auth` 볼륨에 보존됩니다.

<a id="configured-credentials"></a>
## B. 환경변수로 기존 인증 정보 사용

```sh
cp .env.adapter.example .env
```

`.env`의 `PROXY_BIND_IP`, `ADMIN_API_SECRET`를 설정하고, `CLAUDE_CODE_OAUTH_TOKEN`에 공식 `claude setup-token` 결과를 넣거나 `ANTHROPIC_API_KEY`를 설정합니다. Responses를 사용한다면 A 절의 Responses 설정과 독립적인 상태 키를 추가합니다.

```sh
docker compose -f compose.adapter.yml up -d --build
docker compose -f compose.adapter.yml ps
```

서비스는 `claude-proxy`, 기본 공개 포트는 13456, 데이터 볼륨은 `claude-proxy-data`입니다. 기본 추론 백엔드는 여기서도 CLI입니다. `UPSTREAM_AUTH_SOURCE=configured`는 DB에 저장한 인증 → OAuth 환경변수 → API 키 환경변수 순서로 선택합니다.

setup-token만 환경변수에 넣으면 refresh token을 가진 로그인 볼륨이 생기는 것은 아닙니다. 자동갱신이 필요하면 A 경로를 사용하세요. A의 `app`용 overlay를 이 `claude-proxy` 서비스에 그대로 적용하지 마세요.

CLI 없는 HTTP 중계를 명시적으로 선택하려면 `CLAUDE_INFERENCE_BACKEND=http`, `PROXY_BUILD_TARGET=http`를 함께 설정하고 다시 빌드합니다. 이 모드에서는 standalone Responses compact를 사용할 수 없습니다.

## 프록시 API 키 발급과 최초 확인

이 절은 A/B 공통입니다. 먼저 다음 환경변수를 터미널에 비공개로 준비하세요.

- `PROXY_URL`: `http://<PROXY_HOST>:13456` (끝에 `/v1` 없음)
- `ADMIN_API_SECRET`: 컨테이너에 설정한 관리 비밀값

```sh
curl "$PROXY_URL/health"
curl "$PROXY_URL/api/admin/keys" \
  -H "Authorization: Bearer $ADMIN_API_SECRET" \
  -H 'Content-Type: application/json' \
  -d '{"name":"litellm"}'
```

키 생성은 **한 번만** 실행합니다. 성공 응답의 `key` 값을 비공개로 저장하세요. 전체 키는 생성 시 반환되며, DB에는 해시와 표시용 prefix가 남습니다. 이후 요청은 이 키를 `CLAUDE_PROXY_API_KEY` 환경변수에 넣어 사용합니다.

```sh
curl "$PROXY_URL/v1/messages" \
  -H "Authorization: Bearer $CLAUDE_PROXY_API_KEY" \
  -H 'anthropic-version: 2023-06-01' \
  -H 'Content-Type: application/json' \
  -d '{"model":"claude-opus-5-5","max_tokens":2048,"messages":[{"role":"user","content":"Reply with OK."}]}'
```

여기는 **프록시 직접 호출**이므로 모델명에 `-claude`/`-codex`를 붙이지 않습니다. 이 명령은 실제 모델 요청을 보냅니다. HTTP 200은 단건 연결 확인이며, 도구/컴팩션/장기 세션의 검증을 대신하지 않습니다.

이제 [LiteLLM의 8개 모델 설정](../integration/litellm/native-responses.example.yaml)을 적용하고 [클라이언트 연결](CLIENTS.md)로 진행합니다. 기존 서버에서 환경변수 기반 키를 쓰지 않는다면 관리 UI의 기존 named credential 방식을 유지해도 됩니다.

## 유지해야 하는 값

DB 볼륨, 인증 볼륨(A), Responses 상태 키, 기존 Compose 프로젝트명을 유지합니다. 새 상태 키 생성이나 다른 계정 로그인은 이전 대화의 opaque 상태 재입력을 무효화할 수 있습니다. `/health`와 `claude-auth status`는 로컬 준비 상태이며 실제 계정의 모델 접근 권한을 보장하지 않습니다.
