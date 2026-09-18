---
name: myiam
description: myiam-cli로 MyIAM 서비스 설정(로그인 방법, 약관, 정책, 사용자 필드, UI 테마, OAuth2 클라이언트, API 설정)을 관리한다. 서비스 설정을 읽기/생성/수정/삭제/순서변경 하거나, 소속 조직(테넌트)·등급·멤버를 조회하거나, SDK 연동에 필요한 .env 값(service_uid, client_id, redirect uri, api/issuer URL)을 얻거나, 인증/대상 서비스 선택이 필요하거나, 서비스를 처음 만들고 API Key·Client Secret을 발급받는 방법을 안내해야 할 때 사용한다.
---

# MyIAM CLI

MyIAM 서비스의 관리자 패널 설정을 터미널에서 관리한다. 관리할 서비스는 `service use`로 고르고, 서버 주소와 CLI 자체 식별자는 빌드 타임에 바이너리에 내장되어 있어 실행 시점 설정이 없다.

이 CLI를 사용하려면 먼저 [myiam.io](https://myiam.io)에 가입해 관리할 서비스를 보유하고 있어야 한다 — 가입 및 서비스 생성은 웹 콘솔에서만 가능하며, CLI는 이미 존재하는 서비스의 설정만 관리한다.

MyIAM 서비스 자체(개념, 패널 필드별 의미 등)에 대한 전체 문서는 https://myiam.io/llms-full.txt 참고 — 이 스킬은 CLI 사용법만 다룬다.

## 설정 (패널 명령 전에 반드시 필요)

```bash
myiam-cli login                       # 브라우저 device-code 플로우, 이전에 선택한 대상 서비스를 복원
myiam-cli service list                # 관리 가능한 서비스 목록 (기본 출력이 JSON)
myiam-cli service use <uid>           # 이후 모든 패널 명령의 대상 서비스 선택
myiam-cli service select              # 목록 조회 + 대화형 선택을 한 번에 (사람용; 에이전트는 list + use 사용)
myiam-cli whoami                      # 로그인한 계정(uid, email)과 선택된 대상 서비스(uid, label, 내 역할) 확인
myiam-cli logout                      # 저장된 인증 정보 삭제
myiam-cli console [page]              # 웹 콘솔을 브라우저로 연다 (CLI에 없는 작업은 전부 여기서)
```

`login`은 인터랙티브 프롬프트 없이(헤드리스/CI에서도 그대로 동작) JSON 출력에 `target_service_uid`와 함께 `target_service_label`/`services`(접근 가능한 서비스 전체 목록, `service list`와 동일한 형태)를 담아 보여준다 — 이전에 선택한 서비스가 있으면 복원하고, 없으면 첫 번째 서비스를 자동 선택한다. `target_service_label`/`services`는 best-effort라 이미 서비스가 선택돼 있는 상태에서 이 부가 조회만 실패하면 로그인 자체는 성공한 채로 두 필드만 빠질 수 있고, `--quiet`에서는 애초에 조회하지 않아 항상 빠진다. 다른 서비스로 바꾸려면 `service use <uid>`(에이전트) 또는 `service select`(사람, 대화형)를 사용한다.

`login`/`logout`/`console`/`--preview`가 여는 브라우저는 `MYIAM_BROWSER` 환경변수로 고른다: 미설정은 OS 기본, `none`(열지 않음 — URL은 stderr에 출력된다), `chrome`/`firefox`/`edge`/`brave`, `safari`/`arc`(macOS 전용), `terminal-browser`(현재 터미널 탭의 terminal-browser에 새 탭, 없으면 분할 창으로 새로 띄움), `custom`(`MYIAM_BROWSER_COMMAND='firefox --private-window %s'`처럼 명령 지정, 셸을 거치지 않음). 실행에 실패하면 경고 후 OS 기본 브라우저로 연다. `login`과 `logout`은 같은 브라우저로 열어야 로그아웃 때 그 브라우저의 myiam 세션 쿠키가 정리된다.

### 웹 콘솔을 열어야 할 때는 `console` — `open`/`xdg-open`을 직접 실행하지 않는다

서비스 생성, API Key·Client Secret 발급, 가입자 관리처럼 CLI에 명령이 없는 작업은 웹 콘솔에서만 가능하다. 이때 **반드시 `myiam-cli console <page>`를 쓴다** — `open`/`xdg-open`/`start`를 직접 부르면 사용자가 `MYIAM_BROWSER`로 고른 브라우저(로그인돼 있는 그 브라우저)를 건너뛰게 된다. URL은 항상 stderr에도 출력되므로 브라우저를 못 여는 환경에서도 사용자에게 줄 주소가 나온다.

```bash
myiam-cli console                # 콘솔 홈
myiam-cli console new-service    # 서비스 생성 (/startup)
myiam-cli console api            # API 설정 — API Key 발급
myiam-cli console oc             # OAuth2 클라이언트 — Client Secret 발급
myiam-cli console user           # 가입자 목록 (CLI 미지원 영역)
myiam-cli console docs           # 개발 문서
```

그 밖에 `home` `main` `information`(`info`) `ui` `login-type`(`lt`) `field` `term` `policy` `tier` `tenant`를 받고, 이름이 없는 페이지는 `/`로 시작하는 경로를 그대로 준다(`myiam-cli console /service/term/<uid>/detail`). 전체 목록은 `myiam-cli schema`의 `commands.console.pages`. 콘솔이 보여주는 서비스는 콘솔 자신의 선택을 따르므로, CLI의 `service use`와 다른 서비스를 보고 있을 수 있다 — 사용자에게 콘솔 화면의 서비스명을 확인하도록 안내한다.

`service`/`oauth2-client` 패널 명령은 모두 대상 서비스 선택(`My-Target-Service` 헤더)이 필요하다 — `service use` 전에 호출하면 `TARGET_SERVICE_REQUIRED` 에러가 난다. `config get`/`set`은 없음 — 실행 시점에 바꿀 수 있는 값은 대상 서비스와 위 브라우저 선택뿐이다.

### 서비스를 처음 만드는 경우 — 웹 콘솔에서 생성 + API Key 발급

서비스 생성과 API Key 발급은 CLI에 명령이 없다. `service list`가 빈 배열(`[]`)이거나 사용자가 "새 서비스를 만들고 싶다"고 하면 CLI 명령을 찾지 말고, **`myiam-cli console new-service`로 서비스 생성 페이지를 열어준 뒤** 아래 순서를 안내한다 (`open`/`xdg-open`을 직접 실행하지 않는다 — 위 [웹 콘솔을 열어야 할 때](#웹-콘솔을-열어야-할-때는-console--openxdg-open을-직접-실행하지-않는다) 참고. 브라우저를 열 수 없는 환경이면 이 명령이 stderr에 출력한 URL을 그대로 제시한다).

1. **가입/로그인 후 관리자 콘솔에서 서비스를 생성한다.**
2. **API Key 발급** — `myiam-cli console api`로 열고 **API 설정** 섹션의 **키 아이콘**을 클릭한다. 같은 섹션에서 Endpoint와 서비스 UID도 보인다. (문서: https://myiam.io/docs/admin/service/api-settings)
3. API Key는 **생성 순간 한 번만 표시**되므로 그 자리에서 앱의 설정(`.env` 등)에 붙여넣도록 안내한다 — 어느 변수에 둘지는 [발급된 키를 앱에 넣는 위치](#발급된-키를-앱에-넣는-위치) 참고. 이후 콘솔·CLI 어디서도 원본을 다시 볼 수 없고, 잃어버리면 재생성해야 하며 **재생성하면 기존 키는 즉시 무효화**된다 — 이미 운영 중인 서비스라면 재생성 전에 사용자에게 확인한다.
4. 사용자가 완료했다고 하면 CLI로 돌아와 이어간다:

```bash
myiam-cli login                  # 아직 로그인 전일 때만 (service list는 매번 서버를 새로 조회하므로 재로그인 불필요)
myiam-cli service list           # 새 서비스가 보이는지 확인
myiam-cli service use <uid>
myiam-cli service api read       # created_at이 채워졌으면 API Key 발급 완료
myiam-cli service env            # 나머지 .env 값
```

**Client Secret은 이 단계에서 만들지 않는다 — 필요할 때만 발급한다.** PKCE 퍼블릭 클라이언트(`client_authentication_methods=none`)는 시크릿을 전혀 쓰지 않으므로, 앱 구조가 정해져 [퍼블릭 vs 서버 클라이언트](#퍼블릭-클라이언트pkce-vs-서버-클라이언트)를 고른 뒤 서버 클라이언트(`client_secret_basic`/`client_secret_post`)일 때만 발급을 안내한다.

키 값 자체를 대화에 붙여넣어 달라고 요청하지 않는다 — 사용자가 직접 `.env`에 넣게 하고, 에이전트는 `created_at`/`client_secret_issued_at`으로 발급 여부만 확인한다.

## 출력

모든 명령은 구조화된 JSON을 stdout에 출력한다: 성공 시 `{"ok": true, "data": ...}`, 실패 시 stderr에 `{"error", "code", "hint"}` — `error`에는 HTTP 상태와 서버 `_message`가 함께 담긴다.

**종료 코드로 다음 행동을 가른다 — 재시도해도 되는 것은 `3`뿐이다.**

| 종료 | 코드 | 다음 행동 |
|---|---|---|
| 1 | `AUTH_REQUIRED` | 미로그인, 또는 서버가 토큰·세션 갱신을 거부 → 사용자에게 `myiam-cli login` 안내. 단순 만료는 CLI가 알아서 갱신하므로 이 코드로 오지 않는다 |
| 2 | `FORBIDDEN` | 역할 부족 → 관리자 콘솔이나 권한 있는 계정 필요 |
| 3 | `SERVER_ERROR`(5xx) / `REQUEST_ERROR`(서버 도달 실패) | 잠시 후 같은 명령 재시도 |
| 4 | `NOT_FOUND`, `REQUEST_INVALID`, `TARGET_SERVICE_REQUIRED`, `INPUT_ERROR`, `PARSE_ERROR`, `VALIDATION_ERROR` | 요청을 고쳐야 한다 — uid 확인, 리소스 상태 확인(예: DRAFT 아닌 약관은 `publish` 불가), stdin·플래그 수정 |

공통 플래그: `--output, -o json|table` (기본 json), `--quiet, -q` (성공 시 출력 생략).

## 입력 관례

- **중첩되거나 복잡한 데이터는 stdin이 기본 입력 방식이다.** 개별 플래그는 자주 바뀌는 단일 필드용으로만 존재한다.
- stdin으로 패널 JSON을 읽는 명령은 두 부류로 나뉜다. `--from-stdin` **플래그**를 붙여야 하는 명령(`oc update`, `service field create`, `service policy create`, `service term create`, `service login-type create`/`update`, `service information/ui/api update`)과, 플래그 없이 인자만 주면 항상 stdin을 읽는 명령(`*-detail create`/`update`, `service policy/term/field update`, `position`/`update-sections` 전부)이 있다 — 후자에 `--from-stdin`을 붙이면 `unknown flag` 에러가 난다. 헷갈리면 `myiam-cli schema`에서 해당 명령의 `options`에 `--from-stdin`이 실제로 있는지로 구분한다.
- 모든 stdin 명령은 `{"data": {...패널 필드...}}` 형태를 받는다 — 이는 `read`의 출력 형태와 같아서 `read | jq '{data:.data}' | update` 파이프라인이 그대로 성립한다 (`update` 끝에 `--from-stdin`을 붙일지는 위 구분을 따른다).
- `--from-stdin` 플래그가 있는 명령이라도 개별 플래그(`--label` 등)와 동시에 주면 에러가 난다 — stdin이 패널 전체를 대체하므로 병합되지 않고, 하나만 골라야 한다.
- 리소스 자신의 식별자, 그리고 `list`에 반드시 필요한 상위 UID는 항상 위치 인자 `<uid>`다 (필수 플래그가 아님) — 예: `service policy-detail list <service-policy-uid>`, `service term-detail read <uid>`.

## 조직 / 테넌트 (`tenant`, 별칭 `org`) — 조회 위주

서비스는 조직(테넌트)에 속할 수 있고, **유료 등급은 서비스가 아니라 그 조직이 가진다**. 대상 서비스 선택이 필요 없으며 조직 uid는 위치 인자로 준다.

```bash
myiam-cli tenant list                              # 내가 역할을 가진 조직 + 조직으로 도달 못 하는 내 서비스 2종
myiam-cli tenant read <uid>                        # 이름/등급/상태/소속 서비스
myiam-cli tenant update <uid> --name "새 이름"      # 이름·아이콘만 (OWNER/ADMIN, 즉시 반영)
myiam-cli tenant update <uid> --icon '<svg>...</svg>'  # 인라인 SVG — 한 번 설정하면 API로 비울 수 없다
myiam-cli tenant request list <tenant-uid>         # 요청 이력 — 거절 사유(reason)를 볼 수 있는 유일한 경로
myiam-cli tenant member list <tenant-uid>          # 조직 멤버(역할) 목록
myiam-cli tenant member read <tenant-uid> <service-user-uid>
```

- `tenant list`의 `unassigned_services`는 어느 조직에도 없는 FREE 서비스, `unmanaged_tenant_services`는 조직에 속했지만 **내가 그 조직 역할이 없는** 서비스다(조회만 되고 그 조직 관리는 전부 403). 서비스는 정상 운영된다 — 서비스 역할과 조직 역할은 별개 축이다.
- **삭제 상태는 2단계다.** 삭제를 요청했지만 승인 전이면 `status`는 여전히 `NORMAL`이고 `pending_request_types`에만 `DELETE`가 잡힌다. 승인 후에야 `status: PENDING_DELETE` + `delete_after`가 채워진다.
- 조직 역할은 `OWNER`(조직당 1명) / `ADMIN`(일상 운영은 OWNER와 동등) / `MANAGER`(조회 전용).

- 조직 기능은 관리자 콘솔에서 아직 MyIAM 관리자에게만 열려 있다(정식 오픈 전 임시 게이트). CLI는 게이트를 두지 않고 서버 권한만 따르므로, 콘솔 메뉴에 조직이 안 보인다는 말을 들어도 CLI 조회는 정상이다.
- 등급을 올리려는 요청을 받으면 CLI로 시도하지 말고 **관리자 콘솔에서 등급 변경을 요청**하도록 안내한다 — 플랫폼 승인이 필요하고 결제 정보가 얽힌다. `tenant request list`로 그 요청이 승인/거절됐는지는 확인해줄 수 있다.

## OAuth2 클라이언트

```bash
myiam-cli oauth2-client read                      # 별칭: oc read
myiam-cli oc update --redirect-uris "https://a,https://b" --scopes "openid profile"
myiam-cli oc update --post-logout-redirect-uris "https://a"
myiam-cli oc update --grant-types "authorization_code,refresh_token"   # 쉼표 구분, 고정 enum
myiam-cli oc update --auth-methods none           # 쉼표 구분, 고정 enum (none|client_secret_basic|client_secret_post|...)
myiam-cli oc update --meta "..."
myiam-cli oc update --from-stdin                  # 중첩된 client_settings/token_settings (플래그와 동시 사용 불가)
myiam-cli oc token-presets                        # security|balanced|convenience|dev TTL 프리셋
myiam-cli oc update --token-preset balanced       # 프리셋 적용 (다른 필드는 그대로 둠)
```

Client Secret 생성은 CLI에 **없음** — 웹 콘솔 전용 (명확한 사용자 동작 + 1회성 노출이 필요해서 의도적으로 제외). **Client Secret은 서버 클라이언트(`client_authentication_methods`가 `client_secret_basic`/`client_secret_post`)일 때만 필요하다** — PKCE 퍼블릭 클라이언트(`none`)면 발급하지 않는다. 필요할 때는 `myiam-cli console oc`로 연 **서비스 > OAuth2 설정 > Client 기본 정보**에서 Client Secret 옆 **키 아이콘**으로 발급한다(문서: https://myiam.io/docs/admin/service/oauth2-settings). API Key와 마찬가지로 한 번만 표시되고 재생성하면 기존 값이 즉시 무효화된다. `oc read`의 `client_secret_issued_at`이 비어 있으면 아직 발급 전이고, 값 자체는 CLI로 읽을 수 없다.

### 발급된 키를 앱에 넣는 위치

발급된 값은 **MyIAM SDK를 쓰는 앱의 환경변수 파일(`.env` 등)이나 SDK 설정에 사용자가 직접 넣는다** — 콘솔에서 1회만 보이므로 그 자리에서 복사한다. 나머지 값(`service_uid`, `oauth2_client_id`, redirect URI, `api_base_url`, `issuer_url`)은 [`service env`](#sdk-연동값--service-env-하나로-전부)로 한 번에 얻고, 프레임워크별 변수 이름은 각 SDK quickstart 문서를 따른다.

| 앱 형태 | API Key | Client Secret |
|---|---|---|
| 서버(Next.js 서버, Spring Boot, NestJS 등) | 서버 환경변수 | 서버 클라이언트면 서버 환경변수 |
| 웹 SPA(React/Vue 등, `VITE_*`·`NEXT_PUBLIC_*`) | **넣지 않는다** — 번들에 그대로 실린다. REST API가 필요하면 서버에서 Server SDK로 호출 | 넣지 않는다 (PKCE) |
| 모바일(Expo `EXPO_PUBLIC_*`, Flutter `--dart-define`) | 공식 quickstart는 앱에 넣는다. 단 앱 번들에서 추출될 수 있다고 안내하고, 부담되면 `apiKey`를 빼고 `resolveUser`로 자체 백엔드를 경유한다 — 이때 회원가입 완료·사용자 액션 기능은 API Key가 필요하므로 백엔드 쪽에서 처리해야 한다 | 넣지 않는다 (PKCE) |

### 퍼블릭 클라이언트(PKCE) vs 서버 클라이언트

SPA·모바일처럼 시크릿을 숨길 수 없는 앱은 PKCE 퍼블릭 클라이언트로 잡아야 한다. **아래 세 값이 한 세트라 하나만 어긋나도 토큰 교환이 `invalid_client`로 떨어진다.**

| | 퍼블릭(PKCE, SPA/모바일) | 서버(백엔드가 시크릿 보관) |
|---|---|---|
| `client_authentication_methods` | `none` | `client_secret_basic` (또는 `client_secret_post`) |
| `settings.client.require-proof-key` | `true` | 선택 |
| Client Secret | 발급 불필요(앱에 둘 수 없음) | 웹 콘솔에서 발급 필수 |

`scopes`에 **`offline_access`가 없으면 refresh token이 발급되지 않아** access token이 만료되는 순간 로그아웃된 것처럼 보인다 — `authorization_grant_types`에도 `refresh_token`이 같이 있어야 한다.

```bash
myiam-cli oc update --auth-methods none \
  --grant-types "authorization_code,refresh_token" \
  --scopes "openid profile offline_access"
myiam-cli oc read | jq '{data: (.data | .client_settings["settings.client.require-proof-key"] = true)}' \
  | myiam-cli oc update --from-stdin
```

- `scopes`는 공백 구분, `authorization_grant_types`/`client_authentication_methods`(`--grant-types`/`--auth-methods`)는 쉼표 구분이다 (뒤 두 개는 고정 enum이라 오타면 `unknown value` 에러가 난다).
- 플래그로 준 값만 바뀌므로 위 세 값은 플래그가 안전하다. `require-proof-key`만 플래그가 없어 stdin 경로로 켜는데, `--from-stdin`은 패널 전체를 대체하므로 반드시 `oc read` 결과에서 시작한다.

## 서비스 개요 / 정보 / UI / 티어 / API

```bash
myiam-cli service main read                       # svc main read — 읽기 전용 대시보드 (약관 목록은 노출 중인 것만)
myiam-cli service main read --preview              # 로컬 브라우저 대시보드로 렌더링, Ctrl+C까지 블로킹

myiam-cli service information read                 # svc info read
myiam-cli service information update --from-stdin   # label, title, meta, footer, register_user, deregister_user
                                                     # footer: footerNtype은 "path"(어떤 service policy의 bare path, 예 "terms") 또는 "link"(외부 URL); footerNtitle(LanguageText)은 필수 — 약관(term)은 path가 없어 footer에 못 넣는다, 넣으려면 정책으로도 만들 것

myiam-cli service ui read                           # 테마, 색상, 브랜딩
myiam-cli service ui update --theme DARK            # 단일 필드 플래그
myiam-cli service ui update --from-stdin            # 색상/icon_html (중첩 데이터)
myiam-cli service ui preview                        # 실제 로그인 화면을 admin 서버가 렌더링한 그대로 열기 (--page/--theme/--lang, 연 뒤에는 페이지 안 탭/토글/드롭다운으로 계속 전환 가능)

myiam-cli service tier read                         # 읽기 전용: effective_tier(FREE|PRO|ENTERPRISE|UNLIMITED) + 고정 한도 + 소속 조직(tenant)

myiam-cli service api read
myiam-cli service api update --allowed-ips "1.2.3.4,5.6.7.8"

myiam-cli service env                               # SDK 연동값 한 번에 (아래 참고)
```

API Key 생성은 CLI에 **없음** — 웹 콘솔 전용([절차](#서비스를-처음-만드는-경우--웹-콘솔에서-생성--api-key-발급)). API Key와 Client Secret 모두 **읽을 수도 없다** — 서버가 해시만 보관하므로 `read` 응답에 값이 없고, `created_at`/`client_secret_issued_at`으로 발급 여부만 알 수 있다.

### SDK 연동값 — `service env` 하나로 전부

**myiam.io 개발 문서(quickstart)의 `.env` 예시에 나오는 값은 API Key/Client Secret을 빼면 전부 `service env` 하나로 얻는다.** 문서를 보고 값을 하나씩 찾아 여러 `read`를 호출하거나 사용자에게 되묻지 말고, 이 명령부터 실행한다.

```bash
myiam-cli service env            # JSON, 서버 1회 호출(/secured/user/context)
myiam-cli service env -o table   # 사람이 .env로 옮겨 적을 때
```

| 출력 필드 | 쓰임 |
|---|---|
| `service_uid` | 서비스 식별자 |
| `oauth2_client_id` | OAuth2 클라이언트 ID |
| `oauth2_redirect_uri` | 로그인 콜백 |
| `oauth2_post_logout_redirect_uri` | 로그아웃 후 이동 주소 |
| `api_base_url` | REST API 서버 |
| `issuer_url` | OIDC issuer 겸 로그인 화면 서버 |

#### 앱 개발을 시작하기 전에: redirect URI 2종을 먼저 등록한다

**앱을 새로 붙이는 작업이면 코드를 쓰기 전에 `redirect_uris`와 `post_logout_redirect_uris`부터 등록한다.** 이 둘이 비어 있거나 실제 앱 주소와 다르면 로그인/로그아웃이 앱으로 돌아올 곳이 없어 SDK 연동이 끝까지 가지 않는다 — 다 만들고 나서 발견하면 디버깅이 가장 오래 걸리는 지점이다. `service env` 결과의 `oauth2_redirect_uri`/`oauth2_post_logout_redirect_uri`가 비었는지부터 확인하고, 비었으면 사용자에게 앱이 뜰 주소를 물어 등록한다.

```bash
myiam-cli oc update \
  --redirect-uris "http://localhost:3000/callback,https://app.example.com/callback" \
  --post-logout-redirect-uris "http://localhost:3000,https://app.example.com"
myiam-cli service env      # 등록 확인
```

- 여러 개는 쉼표로 구분한다. **로컬 개발 주소와 배포 주소를 처음부터 같이** 넣어두면 배포 시점에 다시 손대지 않아도 된다.
- 서버는 등록된 값과 **정확히 일치**할 때만 통과시킨다 — 스킴(http/https), 포트, 경로, 끝 슬래시까지 앱이 실제로 쓰는 값 그대로 넣어야 한다. 모바일이면 커스텀 스킴(`myapp://callback`)이 그대로 들어간다.
- `oc update`는 플래그로 줄 때 현재 설정을 읽어와 준 플래그만 덮으므로 다른 필드는 안전하다. 반대로 `--from-stdin`은 패널 전체를 대체한다.
- **등록한 문자열과 앱 코드의 값이 글자 단위로 같아야 한다.** SDK 초기화의 redirect URI와 로그아웃 호출의 post-logout redirect URI를 여기 등록한 값 그대로(보통 `.env`로) 넣는다 — 로그아웃 호출에 post-logout redirect URI를 빼먹으면 세션은 끊기는데 앱으로 돌아오지 못해 빈 화면에 멈춘다.
- `service env`가 보여주는 redirect URI는 대표값 하나다 — 등록된 전체 목록은 `oc read`의 `redirect_uris`/`post_logout_redirect_uris`로 확인한다.

여기 없는 값은 API Key와 (서버 클라이언트일 때만) Client Secret뿐이고, 읽을 수 없으므로 웹 콘솔에서 발급받아 사용자가 직접 넣어야 한다. 프레임워크별 변수 이름(`VITE_*`, `EXPO_PUBLIC_MYIAM_*` 등)은 CLI가 아니라 각 quickstart 문서에 있다 — 값은 여기서, 이름은 문서에서 가져다 조합한다.

`--preview` (`service main read`, `service information read`, `service ui read`, `svc td read`, `svc pd read`에 있음)는 패널을 로컬 HTML 페이지(`127.0.0.1:0`)로 렌더링하고 브라우저를 연다; JSON 출력에 `preview_url`이 추가된다. 이건 JSON을 보기 편하게 만든 것일 뿐 실제 사용자 화면이 아니다 — 실제 화면은 아래 `service ui preview`로 본다.

예외적으로 `service ui preview`(별도 명령, `service ui read`의 플래그가 아님)는 admin 콘솔이 쓰는 바로 그 실제 preview 엔드포인트를 호출해 로그인/가입 등 실제 사용자 화면을 지금 설정된 브랜딩/테마 그대로 렌더링한다 (목업이 아니라 픽셀 단위로 동일) — `--page`(기본 `login`; login/signup/signup_form/deregister/edit_profile/edit_email/password_set/password_reset/passkey/policy/notice)로 처음 열 화면을, `--theme`(`light`|`dark`)로 테마를, `--lang`(`ko`|`en`|`ja`|`zh`, 기본 `ko`)로 언어를 고른다. 연 뒤에는 명령을 다시 실행하지 않고도 페이지 자체의 탭/Light-Dark 토글/언어 드롭다운으로 계속 바꿔볼 수 있다.

## 로그인 타입 (`service login-type`, 별칭 `svc lt`)

```bash
myiam-cli svc lt list                              # 현재 구성된 로그인 방법 목록
myiam-cli svc lt create --method EMAIL_OTP          # 플래그만으로 생성 (--from-stdin과 동시 사용 불가)
myiam-cli svc lt create --from-stdin                # SNS_* 등 oauth2 오버라이드가 필요할 때 - method는 stdin data 안에
myiam-cli svc lt read <uid>
myiam-cli svc lt update <uid> --client-id ... --client-secret ...
myiam-cli svc lt delete <uid>                      # 비활성화 = 삭제; 별도의 토글은 없음
echo '{"data":{"content":[{"id1":"uidA"},{"id1":"uidB"}]}}' | myiam-cli svc lt position   # 플래그 없음, 항상 stdin
myiam-cli svc lt sns-defaults SNS_KAKAO             # 인증 불필요; Naver/Kakao/Google/Apple의 OAuth2 기본값 조회
```

`method`(IDP/EMAIL_OTP/PASSKEY/SNS_NAVER/SNS_KAKAO/SNS_GOOGLE/SNS_APPLE)는 생성 시 고정된다. SNS_* 방식은 `sns-defaults` 값이 자동으로 채워지며, `--client-id`/`--client-secret` 또는 `--from-stdin`의 `data.oauth2`로 재정의할 수 있다. 기본값이 없어 직접 넣어야 하는 값은 `sns-defaults` 출력의 `user_provided_fields`에 나온다 — `SNS_APPLE`은 client_id/client_secret 외에 `apple_team_id`/`apple_key_id`/`apple_private_key`(developer.apple.com)도 필요하고, 이 셋은 플래그가 없어 `svc lt read <uid> | jq ... | svc lt update <uid> --from-stdin`으로만 넣는다 (stdin은 패널 전체를 대체하므로 반드시 `read` 결과에서 시작).

## 정책 (`service policy`) / 정책 버전 (`service policy-detail`, 별칭 `svc pd`)

```bash
myiam-cli service policy list
myiam-cli service policy create --label "이용약관" --type SINGLE --path terms   # --status 생략 시 DRAFT
myiam-cli service policy read <uid>
myiam-cli service policy update          # 플래그 없음, 항상 stdin으로 {"data":{...}} 읽음
myiam-cli service policy delete <uid>
myiam-cli service policy publish <uid>   # 노출 / draft <uid> 로 숨김
echo '{"data":{"content":[{"id1":"uidA"},...]}}' | myiam-cli service policy position   # 플래그 없음

myiam-cli svc pd list <service-policy-uid>          # 항목별 content 생략 — 본문은 read <uid>로 확인
myiam-cli svc pd create                             # 플래그 없음, stdin: service_policy_uid, label, version_code, language, title, content, version_string, new_expire_at, exposed_at, status(생략 시 NORMAL)
                                                     # label(이름) / version_code(첫 본문 1, 이후 +1)는 필수 — 약관 섹션의 "버전의 label / version_code" 참고
                                                     # exposed_at은 이름과 달리 사실상 필수, 포맷은 yyyy-MM-ddTHH:mm:ss (오프셋/Z 없는 로컬시간, 예: 2026-07-21T00:00:00)
myiam-cli svc pd read <uid>
myiam-cli svc pd read <uid> --preview               # content HTML을 실사용자 화면 스타일 문서 카드로 렌더링
myiam-cli svc pd update                             # 플래그 없음
myiam-cli svc pd delete <uid>
myiam-cli svc pd publish <uid>                      # 이 버전 노출 / draft <uid> 로 숨김
```

`--path`는 URL 경로 전체가 아니라 bare 값이다 (`/policy/terms`가 아니라 `terms`) — 이 값이 그대로 `service information update`의 footer1target/footer2target(`footerNtype: "path"`)와 매칭된다. `title`(LanguageText, `{"ko":"..."}`)은 `label`과 별개 필드이고 `create`에는 플래그가 없다 — 정책 페이지 렌더링과 footer 연결에 필요하므로 생성 직후 채워둘 것 (`policy update`는 플래그 없이 stdin만 읽는다 — `--from-stdin`을 붙이면 `unknown flag`):

```bash
myiam-cli service policy read <uid> | jq '{data: (.data | .title = {"ko":"이용약관","en":"Terms of Service"})}' \
  | myiam-cli service policy update
```

## 약관 (`service term`) / 약관 버전 (`service term-detail`, 별칭 `svc td`)

정책과 동일한 구조:

```bash
myiam-cli service term list            # --filter all 로 종료된 구약관까지
myiam-cli service term create --label "개인정보 수집·이용 동의" --type REQUIRED   # --status 생략 시 DRAFT
myiam-cli service term read <uid>
myiam-cli service term update          # 플래그 없음, 항상 stdin
myiam-cli service term delete <uid>
myiam-cli service term publish <uid>   # 노출 / revise 후에는 기존 약관 종료 + 재동의
myiam-cli service term draft <uid>     # 숨김 (동의 이력 유지) / 구약관이 있는 계열엔 쓰지 말 것
myiam-cli service term unpublish <uid> # 발행 취소 — 개정판을 초안으로 내리고 구약관 재노출
myiam-cli service term revise <uid>    # 재동의가 필요한 개정 초안 생성
echo '{"data":{"content":[{"id1":"uidA"},...]}}' | myiam-cli service term position   # 플래그 없음

myiam-cli svc td list <service-term-uid>          # 항목별 content 생략 — 본문은 read <uid>로 확인
myiam-cli svc td create                             # 플래그 없음, stdin: service_term_uid, label, version_code, language, title, content, version_string, status(생략 시 NORMAL)
                                                     # label(이름) / version_code(첫 본문 1, 이후 +1)는 필수 — 아래 절 참고
myiam-cli svc td read <uid>
myiam-cli svc td read <uid> --preview
myiam-cli svc td update                             # 플래그 없음
myiam-cli svc td delete <uid>
myiam-cli svc td publish <uid>                      # 이 버전 노출(재동의 없음) / draft <uid> 로 숨김
```

`type`은 약관은 REQUIRED/OPTIONAL, 정책은 SINGLE/BOARD — 그 외는 두 리소스가 서로 대칭이다.

### 어떤 약관·정책을 만들어야 하는가 — 사용자에게 확인하고 정한다

`term`은 **동의를 받고 이력이 남는 항목**, `policy`는 **동의 없이 공개만 하는 문서**(푸터 링크)다. 문서의 법적 성격에 따라 어느 쪽에 만들지가 갈린다 — 개인정보처리방침은 공개 의무 문서라 `policy`이고, 이것을 `term`으로 만들면 공개 문서를 동의 항목으로 받게 된다. 반대로 수집·이용 동의를 `policy`로만 두면 동의 이력이 남지 않는다.

| 문서 | 리소스 | 언제 필요한가 |
|---|---|---|
| 서비스 이용약관 | `term` REQUIRED + 공개용 `policy` | 사실상 항상 |
| 개인정보 수집·이용 동의 | `term` REQUIRED | 항상 — 필수 항목만 최소로 |
| 개인정보처리방침 | **`policy`** (동의 대상 아님, 공개 의무) | 항상 |
| 제3자 제공 동의 | `term`, 보통 OPTIONAL | **실제로 제3자에게 제공할 때만.** 받는 자·항목·목적·보유기간을 특정해야 한다 |
| 처리 위탁(결제대행·문자발송·클라우드 등) | 동의 불필요 — 처리방침에 수탁자 공개 | 제3자 제공과 혼동하지 말 것 |
| 마케팅·광고 수신 동의 | `term` OPTIONAL, 채널별로 분리 | 광고성 정보를 보낼 때. 야간(21~08시) 발송은 별도 동의 |
| 민감정보·고유식별정보 | `term` REQUIRED, 항목별 별도 동의 | 해당 정보를 수집할 때만 |
| 국외 이전 | `term` 또는 처리방침 고지 | 국외로 이전할 때 |
| 만 14세 미만 법정대리인 동의 | **MyIAM에 기능 없음** | 아동 가입을 허용한다면 서비스가 직접 구현해야 한다 |

- **선택 동의를 REQUIRED로 묶지 않는다** — 거부해도 가입이 되어야 하는 항목은 OPTIONAL이다. 서비스 제공에 필수불가결하지 않은 제3자 제공·마케팅이 여기 해당한다.
- **동의서 본문의 "수집 항목"은 `service field list` 결과와 일치시킨다.** 실제 수집 항목과 다른 동의는 의미가 없다. 본문에는 목적·항목·보유기간·거부할 권리와 그에 따른 불이익을 담는다.
- 해외 SNS 로그인(`service login-type list`의 Google/Apple 등)을 켰다면 국외 이전 고지 대상인지 확인한다.
- **선택 동의를 가입 후에 철회하는 화면은 사용자 화면에 없다.** 동의가 나오는 곳은 `service ui preview --page signup`("서비스 정책 동의") 하나뿐이고, `edit_profile`은 입력 필드 수정만 한다. 마케팅 수신 동의를 껐다 켜는 경로가 필요하면 서비스가 직접 구현해야 한다(광고 수신 거부 수단 제공은 법정 의무다).
- `policy`·`notice` 탭이 렌더링하는 것은 `policy` 리소스(공개 문서와 그 개정 이력)다 — 처리방침을 `term`으로 만들면 이 공개 페이지에 뜨지 않는다.
- **어떤 동의가 필요한지는 서비스가 실제로 하는 처리에 달렸다.** 에이전트가 단정해서 만들지 말고 "제3자에게 제공하는가 / 광고를 보내는가 / 민감정보를 받는가 / 만 14세 미만 가입을 받는가"를 먼저 묻고, 확인된 것만 DRAFT로 만든다.
- 동의 이력 조회와 증적은 CLI에 없다(관리자 콘솔 전용).

공개 문서는 `policy`로 만들고 푸터에 연결한다:

```bash
myiam-cli service policy create --label "개인정보처리방침" --type SINGLE --path privacy   # 동의가 아니라 공개
myiam-cli service policy read <uid> | jq '{data: (.data | .title = {"ko":"개인정보처리방침"})}' \
  | myiam-cli service policy update                                                      # title은 create에 플래그가 없다
```

### 버전의 `label` / `version_code` — 서버가 안 막는 필수값

`svc td create`/`svc pd create`의 `label`(이름)과 `version_code`(버전 코드)는 관리자 콘솔에서 필수 입력인데 **서버가 검증하지 않아 빼먹어도 create가 성공한다** — `label`은 빈 문자열, `version_code`는 `0`으로 저장된다. 항상 둘 다 넣는다.

- `label` — 콘솔 버전 목록의 "이름" 열. 보통 상위 약관·정책의 `label`을 그대로 쓴다.
- `version_code` — **사용자에게 보여줄 본문을 고르는 정렬 키**(서버가 version_code 내림차순으로 하나를 뽑는다). **첫 본문은 `1`, 같은 약관·정책에 버전을 추가할 때마다 +1.** 생략해서 전부 `0`이 되면 동률이라 어느 본문이 노출될지 정해지지 않는다. 다음 값은 `svc td list <상위-uid>`(정책은 `svc pd list`)로 현재 최댓값을 확인해 정한다.

`create`뿐 아니라 `update`도 마찬가지다 — 패널 `update`는 전체 대체라 `label`/`version_code`를 빼고 보내면 기존 값이 빈 문자열과 `0`으로 덮인다. 항상 `read` 결과에서 시작한다.

### 노출 상태 (`status`) — 초안으로 준비했다가 한 번에 공개하기

약관·정책과 그 버전 모두 `status`를 갖는다. `NORMAL`이 노출, `DRAFT`가 초안(사용자에게 안 보임)이다. 초안으로 여러 건을 미리 만들어 둔 뒤 한 번에 노출로 바꾸는 것이 의도된 사용법이다. **`term create`/`policy create`는 `--status`를 생략하면 DRAFT로 만든다**(관리자 콘솔과 동일). 버전(`term-detail`/`policy-detail`) 생성은 `NORMAL`이 기본이다 — **이미 노출 중인 약관·정책에 버전을 추가하면 곧바로 사용자에게 보인다.**

**에이전트가 약관·정책 본문을 작성할 때는 초안(DRAFT)까지만 만들고 `publish`/`revise`는 하지 않는다.** 사용자에게 그대로 노출되는 법적 문서이므로 사람이 내용을 검토한 뒤 직접 발행한다. 버전(`svc td create`/`svc pd create`)의 `status`는 **상위 약관·정책의 상태를 먼저 `read`로 확인한 뒤** 정한다:

- 상위가 **NORMAL(노출 중)** → 버전에 `"status":"DRAFT"`를 반드시 넣는다. 생략하면 기본 NORMAL이라 검토 전 본문이 곧바로 공개된다. 검토 후 발행은 `svc td publish <버전-uid>`.
- 상위가 **DRAFT**(방금 `create`/`revise`로 만든 것) → 버전은 `status`를 생략(NORMAL)한다. 상위가 숨겨져 있어 공개되지 않고, 나중에 사람이 상위를 `publish`하면 본문이 함께 보인다. 여기에 DRAFT를 넣으면 상위를 발행해도 본문 버전이 숨겨져 **빈 약관이 노출**된다. 초안 생성과 `read --preview` 확인까지 마친 뒤, 발행 명령(`service term publish <uid>` 등)을 uid와 함께 안내하고 멈춘다.

```bash
myiam-cli service term create --label "개인정보 수집·이용 동의" --type REQUIRED   # DRAFT로 생성됨
echo '{"data":{"service_term_uid":"<uid>","label":"개인정보 수집·이용 동의","version_code":1,"language":"ko","title":"...","content":"<p>...</p>"}}' \
  | myiam-cli svc td create                                                # 본문 작성 — 상위가 DRAFT라 status 생략

# 이미 노출 중인 약관에 본문 버전을 추가할 때만:
echo '{"data":{"service_term_uid":"<노출중-uid>","label":"개인정보 수집·이용 동의","version_code":2,"language":"ko","title":"...","content":"<p>...</p>","status":"DRAFT"}}' \
  | myiam-cli svc td create                                                # version_code는 기존 최댓값+1

myiam-cli service term publish <uid>        # 노출 (관리자 콘솔의 "지금부터 사용")
myiam-cli service term draft <uid>          # 숨김 (관리자 콘솔의 "초안으로 변경")
```

같은 `publish`/`draft` 쌍이 `service policy`, `svc td`(약관 버전), `svc pd`(정책 버전)에도 있다. 이 명령들은 패널을 `read`한 뒤 `status`만 덮어 `update`한다 — **패널 `update`에는 부분 업데이트가 없으므로** `echo '{"data":{"uid":"...","status":"NORMAL"}}' | myiam-cli service term update` 같은 호출은 `label`/`type`을 빈 값으로 덮어쓴다. 상태만 바꿀 땐 반드시 `publish`/`draft`를 쓴다.

`DRAFT` 버전은 버전 코드가 더 높아도 사용자에게 최신 버전으로 잡히지 않는다. 정책 버전의 `exposed_at`과는 별개로 동작한다 — `exposed_at`이 이미 지났어도 `DRAFT`면 공개되지 않는다. `read`/`list`는 관리용이라 초안도 그대로 조회되므로 `read --preview`로 공개 전 내용을 확인할 수 있다.

실제 사용자 화면이 버전을 고르는 규칙:

- **약관 본문**: 요청 언어 → 한국어 → 아무 언어 순으로, 각 단계에서 `NORMAL` 버전 중 버전 코드가 가장 높은 것. 노출 중인 약관에 `NORMAL` 버전이 하나도 없으면 에러 없이 **제목·본문이 빈 동의 항목**으로 뜬다. `exposed_at`은 약관에 없다.
- **정책 본문**: 요청 언어만(대체 언어 없음), `NORMAL`이면서 `exposed_at`이 현재 이전인 버전만.

**`service ui preview`로는 초안이 숨겨졌는지 확인할 수 없다.** 관리자 미리보기는 의도적으로 `DRAFT` 약관과 `DRAFT` 버전까지 렌더링하고(`CLOSED`만 제외), 요청 언어만 보며 대체 언어도 쓰지 않는다. 노출 여부는 미리보기가 아니라 `service term list`/`svc td read`의 `status`로 판단한다.

### 약관 개정 (`revise`) — 재동의를 받아야 할 때

약관에만 **개정 계열**이 있다. `root_service_term_uid`가 같은 약관들이 한 약관의 개정 이력이다.

```bash
myiam-cli service term revise <uid>     # 같은 계열의 새 DRAFT 약관 생성 (본문은 복사되지 않음)
myiam-cli svc td create                 # 새 약관의 본문 작성 (stdin, service_term_uid = 위에서 만든 uid, status 생략 — 상위가 DRAFT)
                                        # 개정판은 uid가 다른 새 약관이므로 version_code는 다시 1부터, label도 새로 넣는다
myiam-cli service term publish <새-uid>  # 기존 약관 종료 + 전원 재동의
myiam-cli service term list --filter all # 종료(CLOSED)된 구약관까지 포함해 이력 확인
```

**재동의가 발생하는 경로는 이것 하나뿐이다.** 같은 약관을 `draft`로 내렸다 `publish`로 다시 올리거나, `svc td`로 새 본문 버전을 노출하는 것은 동의 이력에 영향을 주지 않는다 — 오탈자 수정처럼 다시 동의받을 필요가 없는 변경은 `svc td create`로 새 버전을 추가한다. 어느 쪽인지 애매하면 사용자에게 "이 변경으로 전 사용자에게 다시 동의를 받아야 하는지" 확인한 뒤 고른다.

**잘못 발행했으면 `draft`가 아니라 `unpublish`로 되돌린다.**

```bash
myiam-cli service term unpublish <개정판-uid>   # 개정판 → DRAFT, 직전 구약관 → NORMAL, 재동의 요구도 취소
```

`draft`는 구약관을 `CLOSED`로 둔 채 개정판만 내리므로 계열에 노출 중인 약관이 하나도 남지 않는다 — REQUIRED 약관이면 가입·동의 화면에서 아예 사라진다. 반대로 구약관이 없는 최초 약관은 `unpublish`가 400을 내므로 `draft`를 쓴다.

구약관은 삭제되지 않고 `CLOSED`로 남는다(동의 이력 보존). 기본 `term list`에서는 감춰지고 `--filter all`로만 보인다.

**개정은 `publish`로 끝나지 않는다.** 사용자에게 불리한 변경은 시행 30일 전 공지와 이메일 개별 통지가 법정 요건인데, 그 대량 고지 메일 발송은 관리자 콘솔 전용이라 CLI에 명령이 없다. `publish`까지 마쳤으면 "남은 고지 메일 발송은 관리자 콘솔에서 해야 한다"고 반드시 알린다.

## 사용자 필드 (`service field`)

```bash
myiam-cli service field list
myiam-cli service field create --label "닉네임" --field-key nickname --field-type NICKNAME
myiam-cli service field create --from-stdin          # 위 세 플래그와 동시 사용 불가 - data 안에 label/field_key/field_type 전부 넣을 것
myiam-cli service field read <uid>
myiam-cli service field update          # 플래그 없음, 항상 stdin — field_type/field_key는 변경 불가, 그대로 다시 보낼 것
myiam-cli service field delete <uid>                  # 이것이 곧 "수집 중단"이다 — 별도의 활성/비활성 플래그는 없음
echo '{"data":{"content":[{"id1":"uidA"},...]}}' | myiam-cli service field position          # 플래그 없음
echo '{"data":{"content":[{"uid":"...","section":1},...]}}' | myiam-cli service field update-sections   # 플래그 없음
```

`field_type`(CUSTOM, EMAIL, NAME, NICKNAME, GENDER, BIRTH_YEAR, BIRTHDAY, DATE_OF_BIRTH, MOBILE_NUMBER, HOME_NUMBER, PHONE_NUMBER, ADDRESS)과 `field_key`는 생성 후 영구히 고정된다. `CUSTOM`은 임의의 `field_key`를 직접 정할 수 있고, 그 외 타입은 기존에 존재하던 시스템 필드를 재활성화하는 것이다 — `field_key`는 `field_type`을 소문자로 바꾼 값이다(예: `EMAIL` → `email`, `MOBILE_NUMBER` → `mobile_number`). `field list`/`field read`는 field_key를 노출하지 않으므로(생성 응답에서만 echo됨) 조회하지 말고 이 규칙으로 계산할 것 — 예전에 삭제한 시스템 필드를 다시 켜는 방법도 이것이다.

`--field-type`/`--field-key`/`--label`과 `--from-stdin`을 동시에 주면 에러가 난다 — 반드시 하나만 골라야 하고, `--from-stdin`을 쓸 땐 세 값을 전부 stdin `data` 안에 넣어야 한다.

## 작업 흐름 가이드

1. **최초 설정** → `login` (자동 선택된 서비스가 없으면 `service list` → `service use <uid>`). `service list`가 빈 배열(`[]`)을 반환하면 아직 [myiam.io](https://myiam.io)에 가입해 관리할 서비스를 만들지 않은 것이다 — CLI 명령을 더 시도하지 말고 [서비스를 처음 만드는 경우](#서비스를-처음-만드는-경우--웹-콘솔에서-생성--api-key-발급) 순서대로 `myiam-cli console new-service`로 콘솔을 열어 서비스 생성·API Key 발급을 안내한 뒤, 완료되면 다시 `service list`로 확인한다.
2. **앱 개발 시작** → (a) 신규 서비스면 웹 콘솔에서 API Key 발급([절차](#서비스를-처음-만드는-경우--웹-콘솔에서-생성--api-key-발급)) — Client Secret은 (c)에서 서버 클라이언트로 정해졌을 때만 발급, (b) `oc update --redirect-uris ... --post-logout-redirect-uris ...`로 콜백 주소 등록(로컬+배포 함께)하고 같은 값을 앱 코드에도 설정, (c) PKCE 앱이면 `client_authentication_methods=none` + `require-proof-key=true` + `scopes`에 `offline_access`(Client Secret 불필요), 백엔드가 시크릿을 보관하는 서버 앱이면 그때 웹 콘솔에서 Client Secret 발급. 빠뜨리면 순서대로 로그인/로그아웃이 앱으로 못 돌아오거나 토큰 갱신이 안 된다
3. **SDK 연동 / .env 채우기** → `service env` 한 번 (개발 문서의 값은 여기서 다 나온다; API Key와 서버 클라이언트의 Client Secret만 웹 콘솔 — 넣을 위치는 [표](#발급된-키를-앱에-넣는-위치) 참고)
4. **현재 설정 확인** → `service main read`로 개요 대시보드부터 보고, `information`/`ui`/`login-type`/`term`/`policy`/`field`로 세부 진입
5. **필드 하나만 변경** → 해당 플래그 사용 (예: `service ui update --theme DARK`)
6. **중첩/복잡한 데이터 변경** → `read | jq '{data:(.data | 수정)}' | update` — `--from-stdin`은 플래그가 있는 명령(`oc`, `information`/`ui`/`api`, `login-type`)에만 붙인다. `policy`/`term`/`field`/`*-detail`의 `update`에 붙이면 `unknown flag`
7. **순서 변경** (`login-type`, `policy`, `term`, `field`) → `position`에 (플래그 없이, stdin으로) `{"data":{"content":[{"id1":"<uid>"},...]}}` 형태로 전달
8. **눈으로 확인** → 실제 로그인·가입 화면(브랜딩/테마 반영)은 `service ui preview`, 약관·정책 본문이나 패널 값은 해당 `read --preview` (main/information/ui/term-detail/policy-detail만 지원; 받은 JSON을 로컬에서 렌더링한 것일 뿐 실제 사용자 화면은 아님)
9. **스크립팅/CI** → `login`으로 인증 정보를 저장해두면(OS 키체인, 헤드리스 환경은 `~/.myiam/credentials.yaml` 폴백) 같은 명령을 그대로 헤드리스로 사용 가능

## CLI에서 의도적으로 제외한 것

- Client Secret / API Key 생성 — 웹 콘솔 전용 (1회성 노출 UI, 명확한 사용자 동작 필요)
- 서비스 생성/파기, 운영자 역할 초대/승인, 소유자 이전 — 되돌리기 어렵거나 사람 간 권한 위임이 필요한 작업, 관리자 콘솔 전용
- 조직(테넌트) 쓰기 동작 대부분 — 생성·삭제·복구·등급변경 요청(플랫폼 승인 필요), 결제/사업자 정보(개인정보), 멤버 초대·역할 변경·소유권 이관, 서비스 이동/양수도. CLI에 있는 쓰기는 `tenant update`(이름·아이콘)뿐이다
- 약관 개정 고지 등 대량 메일 발송 — 발송 건수를 사람이 직접 확인·입력해야 하는 되돌릴 수 없는 작업
- 최종 서비스 가입자(회원) 관리 — 이 CLI는 서비스의 "설정"을 다루는 도구이지 "가입자"를 다루는 도구가 아니다

이 목록에 있는 작업을 요청받으면 명령을 찾지 말고 **관리자 콘솔에서 수행해야 한다고 안내**한다. 조회 명령(`tenant read`/`request list`/`member list`)으로 현재 상태와 요청 결과를 확인해주는 것까지가 CLI의 역할이다.

## 스키마

`myiam-cli schema`로 전체 명령어 스펙(옵션, enum 값, stdin 필드 형태)을 기계가 읽을 수 있는 형태로 확인할 수 있다 — 명령어가 바뀌면 `cmd/schema.go`와 항상 동기화할 것.
