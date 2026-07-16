---
name: myiam-cli
description: myiam-cli로 MyIAM 서비스 설정(로그인 방법, 약관, 정책, 사용자 필드, UI 테마, OAuth2 클라이언트, API 설정)을 관리한다. 서비스 설정을 읽기/생성/수정/삭제/순서변경 하거나, 인증/대상 서비스 선택이 필요할 때 사용한다.
---

# MyIAM CLI

하나의 고정된 MyIAM 서비스의 관리자 패널 설정을 터미널에서 관리한다. 서버 주소와 CLI 자체 식별자는 빌드 타임에 바이너리에 내장되어 있어 실행 시점 설정이 없다.

이 CLI를 사용하려면 먼저 [myiam.io](https://myiam.io)에 가입해 관리할 서비스(테넌트)를 보유하고 있어야 한다 — 가입 및 서비스 생성은 웹 콘솔에서만 가능하며, CLI는 이미 존재하는 서비스의 설정만 관리한다.

## 설정 (패널 명령 전에 반드시 필요)

```bash
myiam-cli login                       # 브라우저 device-code 플로우, 이전에 선택한 대상 서비스를 복원
myiam-cli service list                # 관리 가능한 서비스 목록 (기본 출력이 JSON)
myiam-cli service use <uid>           # 이후 모든 패널 명령의 대상 서비스 선택
myiam-cli service select              # 목록 조회 + 대화형 선택을 한 번에 (사람용; 에이전트는 list + use 사용)
myiam-cli logout                      # 저장된 인증 정보 삭제
```

`service`/`oauth2-client` 패널 명령은 모두 대상 서비스 선택(`My-Target-Service` 헤더)이 필요하다 — `service use` 전에 호출하면 `TARGET_SERVICE_REQUIRED` 에러가 난다. `config get`/`set`은 없음 — 실행 시점에 바꿀 수 있는 값은 대상 서비스뿐이다.

## 출력

모든 명령은 구조화된 JSON을 stdout에 출력한다: 성공 시 `{"ok": true, "data": ...}`, 실패 시 stderr에 `{"error", "code", "hint"}` (명령은 Go 에러를 반환하지 않고 종료 코드로 실패를 표시한다). 공통 플래그: `--output, -o json|table` (기본 json), `--quiet, -q` (성공 시 출력 생략).

## 입력 관례

- **중첩되거나 복잡한 데이터는 `--from-stdin`이 기본 입력 방식이다.** 개별 플래그는 자주 바뀌는 단일 필드용으로만 존재한다.
- 모든 `--from-stdin` 명령은 `{"data": {...패널 필드...}}` 형태를 받는다 — 이는 `read`의 출력 형태와 같아서 `read | jq '{data:.data}' | update --from-stdin` 파이프라인이 그대로 성립한다.
- 리소스 자신의 식별자, 그리고 `list`에 반드시 필요한 상위 UID는 항상 위치 인자 `<uid>`다 (필수 플래그가 아님) — 예: `service policy-detail list <service-policy-uid>`, `service term-detail read <uid>`.
- `position`/`update-sections`는 플래그가 아예 없다 — 항상 `{"data":{"content":[...]}}` 형태로 `--from-stdin`만 받는다.

## OAuth2 클라이언트

```bash
myiam-cli oauth2-client read                      # 별칭: oc read
myiam-cli oc update --redirect-uris "https://a,https://b" --scopes "openid profile" --json
myiam-cli oc update --from-stdin                  # 중첩된 client_settings/token_settings
myiam-cli oc token-presets                        # security|balanced|convenience|dev TTL 프리셋
myiam-cli oc update --token-preset balanced       # 프리셋 적용 (다른 필드는 그대로 둠)
```

Client Secret 생성은 CLI에 **없음** — 웹 콘솔 전용 (명확한 사용자 동작 + 1회성 노출이 필요해서 의도적으로 제외).

## 서비스 개요 / 정보 / UI / 티어 / API

```bash
myiam-cli service main read                       # svc main read — 읽기 전용 대시보드
myiam-cli service main read --preview              # 로컬 브라우저 대시보드로 렌더링, Ctrl+C까지 블로킹

myiam-cli service information read                 # svc info read
myiam-cli service information update --from-stdin   # label, title, meta, footer, register_user, deregister_user

myiam-cli service ui read                           # 테마, 색상, 브랜딩
myiam-cli service ui update --theme DARK            # 단일 필드 플래그
myiam-cli service ui update --from-stdin            # 색상/icon_html (중첩 데이터)

myiam-cli service tier read                         # 읽기 전용: FREE|PRO|ENTERPRISE|UNLIMITED + 고정 한도

myiam-cli service api read
myiam-cli service api update --allowed-ips "1.2.3.4,5.6.7.8"
```

API Key 생성도 마찬가지로 CLI에 **없음** — 웹 콘솔 전용.

`--preview` (아래 `main read`, `information read`, `ui read`, `*-detail read` 명령에 있음)는 패널을 로컬 HTML 페이지(`127.0.0.1:0`)로 렌더링하고 브라우저를 연다; JSON 출력에 `preview_url`이 추가된다. 이건 JSON을 보기 편하게 만든 것일 뿐, admin 콘솔의 실제 `PreviewController`(별개 기능, `docs/preview.md` 참고)를 대체하는 게 아니다.

## 로그인 타입 (`service login-type`, 별칭 `svc lt`)

```bash
myiam-cli svc lt list                              # 현재 구성된 로그인 방법 목록
myiam-cli svc lt create --method EMAIL_OTP --from-stdin
myiam-cli svc lt read <uid>
myiam-cli svc lt update <uid> --client-id ... --client-secret ...
myiam-cli svc lt delete <uid>                      # 비활성화 = 삭제; 별도의 토글은 없음
echo '{"data":{"content":[{"id1":"uidA"},{"id1":"uidB"}]}}' | myiam-cli svc lt position --from-stdin
myiam-cli svc lt sns-defaults SNS_KAKAO             # 인증 불필요; Naver/Kakao/Google/Apple의 OAuth2 기본값 조회
```

`method`(IDP/EMAIL_OTP/PASSKEY/SNS_NAVER/SNS_KAKAO/SNS_GOOGLE/SNS_APPLE)는 생성 시 고정된다. SNS_* 방식은 `sns-defaults` 값이 자동으로 채워지며, `--client-id`/`--client-secret` 또는 `--from-stdin`의 `data.oauth2`로 재정의할 수 있다.

## 정책 (`service policy`) / 정책 버전 (`service policy-detail`, 별칭 `svc pd`)

```bash
myiam-cli service policy list
myiam-cli service policy create --label "이용약관" --type SINGLE --path /policy/terms
myiam-cli service policy read <uid>
myiam-cli service policy update --from-stdin
myiam-cli service policy delete <uid>
echo '{"data":{"content":[{"id1":"uidA"},...]}}' | myiam-cli service policy position --from-stdin

myiam-cli svc pd list <service-policy-uid>          # 항목별 content 생략 — 본문은 read <uid>로 확인
myiam-cli svc pd create --from-stdin                # service_policy_uid, language, title, content, version_string, new_expire_at, exposed_at
myiam-cli svc pd read <uid>
myiam-cli svc pd read <uid> --preview               # content HTML을 실사용자 화면 스타일 문서 카드로 렌더링
myiam-cli svc pd update --from-stdin
myiam-cli svc pd delete <uid>
```

## 약관 (`service term`) / 약관 버전 (`service term-detail`, 별칭 `svc td`)

정책과 동일한 구조:

```bash
myiam-cli service term list
myiam-cli service term create --label "개인정보처리방침" --type REQUIRED
myiam-cli service term read <uid>
myiam-cli service term update --from-stdin
myiam-cli service term delete <uid>
myiam-cli service term position --from-stdin

myiam-cli svc td list <service-term-uid>
myiam-cli svc td create --from-stdin                # service_term_uid, language, title, content, version_string
myiam-cli svc td read <uid>
myiam-cli svc td read <uid> --preview
myiam-cli svc td update --from-stdin
myiam-cli svc td delete <uid>
```

`type`은 약관은 REQUIRED/OPTIONAL, 정책은 SINGLE/BOARD — 그 외는 두 리소스가 서로 대칭이다.

## 사용자 필드 (`service field`)

```bash
myiam-cli service field list
myiam-cli service field create --label "닉네임" --field-key nickname --field-type NICKNAME --from-stdin
myiam-cli service field read <uid>
myiam-cli service field update --from-stdin          # field_type/field_key는 변경 불가 — 그대로 다시 보낼 것
myiam-cli service field delete <uid>                  # 이것이 곧 "수집 중단"이다 — 별도의 활성/비활성 플래그는 없음
echo '{"data":{"content":[{"id1":"uidA"},...]}}' | myiam-cli service field position --from-stdin
echo '{"data":{"content":[{"uid":"...","section":1},...]}}' | myiam-cli service field update-sections --from-stdin
```

`field_type`(CUSTOM, EMAIL, NAME, NICKNAME, GENDER, BIRTH_YEAR, BIRTHDAY, DATE_OF_BIRTH, MOBILE_NUMBER, HOME_NUMBER, PHONE_NUMBER, ADDRESS)과 `field_key`는 생성 후 영구히 고정된다. `CUSTOM`은 임의의 `field_key`를 직접 정할 수 있고, 그 외 타입은 기존에 존재하던 시스템 필드를 재활성화하는 것이다 (`field list`에서 둘 다 확인 — 비활성화된 시스템 필드도 함께 조회됨) — 예전에 삭제한 시스템 필드를 다시 켜는 방법도 이것이다.

## 작업 흐름 가이드

1. **최초 설정** → `login` (자동 선택된 서비스가 없으면 `service list` → `service use <uid>`). `service list`가 빈 배열(`[]`)을 반환하면 아직 [myiam.io](https://myiam.io)에 가입해 관리할 서비스를 만들지 않은 것이다 — CLI 명령을 더 시도하지 말고 사용자에게 myiam.io 가입 및 서비스 생성부터 안내한 뒤, 완료되면 다시 `service list`로 확인한다.
2. **현재 설정 확인** → `service main read`로 개요 대시보드부터 보고, `information`/`ui`/`login-type`/`term`/`policy`/`field`로 세부 진입
3. **필드 하나만 변경** → 해당 플래그 사용 (예: `service ui update --theme DARK`)
4. **중첩/복잡한 데이터 변경** → `read | jq '{data:.data}' | jq로 수정 | update --from-stdin`
5. **순서 변경** (`login-type`, `policy`, `term`, `field`) → `position --from-stdin`에 `{"data":{"content":[{"id1":"<uid>"},...]}}` 형태로 전달
6. **변경 전 눈으로 확인** → 해당 `read --preview` (main/information/ui/term-detail/policy-detail만 지원); 이미 받은 JSON을 로컬에서 렌더링한 것일 뿐 실제 사용자 화면은 아님
7. **스크립팅/CI** → `login`으로 인증 정보를 저장해두면(OS 키체인, 헤드리스 환경은 `~/.myiam/credentials.yaml` 폴백) 같은 명령을 그대로 헤드리스로 사용 가능

## CLI에서 의도적으로 제외한 것

- Client Secret / API Key 생성 — 웹 콘솔 전용 (1회성 노출 UI, 명확한 사용자 동작 필요)
- 서비스(테넌트) 생성/파기, 운영자 역할 초대/승인, 소유자 이전 — 되돌리기 어렵거나 사람 간 권한 위임이 필요한 작업, 관리자 콘솔 전용
- 최종 서비스 가입자(회원) 관리 — 이 CLI는 서비스의 "설정"을 다루는 도구이지 "가입자"를 다루는 도구가 아니다

## 스키마

`myiam-cli schema`로 전체 명령어 스펙(옵션, enum 값, stdin 필드 형태)을 기계가 읽을 수 있는 형태로 확인할 수 있다 — 명령어가 바뀌면 `cmd/schema.go`와 항상 동기화할 것.
