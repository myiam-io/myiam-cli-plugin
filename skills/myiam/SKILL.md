---
name: myiam
description: myiam-cli로 MyIAM 서비스 설정(로그인 방법, 약관, 정책, 사용자 필드, UI 테마, OAuth2 클라이언트, API 설정)을 관리한다. 서비스 설정을 읽기/생성/수정/삭제/순서변경 하거나, 소속 조직(테넌트)·등급·멤버를 조회하거나, 인증/대상 서비스 선택이 필요할 때 사용한다.
---

# MyIAM CLI

하나의 고정된 MyIAM 서비스의 관리자 패널 설정을 터미널에서 관리한다. 서버 주소와 CLI 자체 식별자는 빌드 타임에 바이너리에 내장되어 있어 실행 시점 설정이 없다.

이 CLI를 사용하려면 먼저 [myiam.io](https://myiam.io)에 가입해 관리할 서비스(테넌트)를 보유하고 있어야 한다 — 가입 및 서비스 생성은 웹 콘솔에서만 가능하며, CLI는 이미 존재하는 서비스의 설정만 관리한다.

MyIAM 서비스 자체(개념, 패널 필드별 의미 등)에 대한 전체 문서는 https://myiam.io/llms-full.txt 참고 — 이 스킬은 CLI 사용법만 다룬다.

## 설정 (패널 명령 전에 반드시 필요)

```bash
myiam-cli login                       # 브라우저 device-code 플로우, 이전에 선택한 대상 서비스를 복원
myiam-cli service list                # 관리 가능한 서비스 목록 (기본 출력이 JSON)
myiam-cli service use <uid>           # 이후 모든 패널 명령의 대상 서비스 선택
myiam-cli service select              # 목록 조회 + 대화형 선택을 한 번에 (사람용; 에이전트는 list + use 사용)
myiam-cli logout                      # 저장된 인증 정보 삭제
```

`login`은 인터랙티브 프롬프트 없이(헤드리스/CI에서도 그대로 동작) JSON 출력에 `target_service_uid`와 함께 `target_service_label`/`services`(접근 가능한 서비스 전체 목록, `service list`와 동일한 형태)를 담아 보여준다 — 이전에 선택한 서비스가 있으면 복원하고, 없으면 첫 번째 서비스를 자동 선택한다. `target_service_label`/`services`는 best-effort라 이미 서비스가 선택돼 있는 상태에서 이 부가 조회만 실패하면 로그인 자체는 성공한 채로 두 필드만 빠질 수 있고, `--quiet`에서는 애초에 조회하지 않아 항상 빠진다. 다른 서비스로 바꾸려면 `service use <uid>`(에이전트) 또는 `service select`(사람, 대화형)를 사용한다.

`service`/`oauth2-client` 패널 명령은 모두 대상 서비스 선택(`My-Target-Service` 헤더)이 필요하다 — `service use` 전에 호출하면 `TARGET_SERVICE_REQUIRED` 에러가 난다. `config get`/`set`은 없음 — 실행 시점에 바꿀 수 있는 값은 대상 서비스뿐이다.

## 출력

모든 명령은 구조화된 JSON을 stdout에 출력한다: 성공 시 `{"ok": true, "data": ...}`, 실패 시 stderr에 `{"error", "code", "hint"}` (명령은 Go 에러를 반환하지 않고 종료 코드로 실패를 표시한다). 공통 플래그: `--output, -o json|table` (기본 json), `--quiet, -q` (성공 시 출력 생략).

## 입력 관례

- **중첩되거나 복잡한 데이터는 stdin이 기본 입력 방식이다.** 개별 플래그는 자주 바뀌는 단일 필드용으로만 존재한다.
- stdin으로 패널 JSON을 읽는 명령은 두 부류로 나뉜다. `--from-stdin` **플래그**를 붙여야 하는 명령(`oc update`, `service field create`, `service policy create`, `service term create`, `service login-type create`/`update`, `service information/ui/api update`)과, 플래그 없이 인자만 주면 항상 stdin을 읽는 명령(`*-detail create`/`update`, `service policy/term/field update`, `position`/`update-sections` 전부)이 있다 — 후자에 `--from-stdin`을 붙이면 `unknown flag` 에러가 난다. 헷갈리면 `myiam-cli schema`에서 해당 명령의 `options`에 `--from-stdin`이 실제로 있는지로 구분한다.
- 모든 stdin 명령은 `{"data": {...패널 필드...}}` 형태를 받는다 — 이는 `read`의 출력 형태와 같아서 `read | jq '{data:.data}' | update --from-stdin` 파이프라인이 그대로 성립한다.
- `--from-stdin` 플래그가 있는 명령이라도 개별 플래그(`--label` 등)와 동시에 주면 에러가 난다 — stdin이 패널 전체를 대체하므로 병합되지 않고, 하나만 골라야 한다.
- 리소스 자신의 식별자, 그리고 `list`에 반드시 필요한 상위 UID는 항상 위치 인자 `<uid>`다 (필수 플래그가 아님) — 예: `service policy-detail list <service-policy-uid>`, `service term-detail read <uid>`.

## 조직 / 테넌트 (`tenant`, 별칭 `org`) — 조회 위주

서비스는 조직(테넌트)에 속할 수 있고, **유료 등급은 서비스가 아니라 그 조직이 가진다**. 대상 서비스 선택이 필요 없으며 조직 uid는 위치 인자로 준다.

```bash
myiam-cli tenant list                              # 내가 역할을 가진 조직 + 조직으로 도달 못 하는 내 서비스 2종
myiam-cli tenant read <uid>                        # 이름/등급/상태/소속 서비스
myiam-cli tenant update <uid> --name "새 이름"      # 이름·아이콘만 (OWNER/ADMIN, 즉시 반영)
myiam-cli tenant request list <tenant-uid>         # 요청 이력 — 거절 사유(reason)를 볼 수 있는 유일한 경로
myiam-cli tenant member list <tenant-uid>          # 조직 멤버(역할) 목록
myiam-cli tenant member read <tenant-uid> <service-user-uid>
```

- `tenant list`의 `unassigned_services`는 어느 조직에도 없는 FREE 서비스, `unmanaged_tenant_services`는 조직에 속했지만 **내가 그 조직 역할이 없는** 서비스다(조회만 되고 그 조직 관리는 전부 403). 서비스는 정상 운영된다 — 서비스 역할과 조직 역할은 별개 축이다.
- **삭제 상태는 2단계다.** 삭제를 요청했지만 승인 전이면 `status`는 여전히 `NORMAL`이고 `pending_request_types`에만 `DELETE`가 잡힌다. 승인 후에야 `status: PENDING_DELETE` + `delete_after`가 채워진다.
- 조직 역할은 `OWNER`(조직당 1명) / `ADMIN`(일상 운영은 OWNER와 동등) / `MANAGER`(조회 전용).
- 등급을 올리려는 요청을 받으면 CLI로 시도하지 말고 **관리자 콘솔에서 등급 변경을 요청**하도록 안내한다 — 플랫폼 승인이 필요하고 결제 정보가 얽힌다. `tenant request list`로 그 요청이 승인/거절됐는지는 확인해줄 수 있다.

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
                                                     # footer: footerNtype은 "path"(어떤 service policy의 bare path, 예 "terms") 또는 "link"(외부 URL); footerNtitle(LanguageText)은 필수 — 약관(term)은 path가 없어 footer에 못 넣는다, 넣으려면 정책으로도 만들 것

myiam-cli service ui read                           # 테마, 색상, 브랜딩
myiam-cli service ui update --theme DARK            # 단일 필드 플래그
myiam-cli service ui update --from-stdin            # 색상/icon_html (중첩 데이터)
myiam-cli service ui preview                        # 실제 로그인 화면을 admin 서버가 렌더링한 그대로 열기 (--page/--theme/--lang, 연 뒤에는 페이지 안 탭/토글/드롭다운으로 계속 전환 가능)

myiam-cli service tier read                         # 읽기 전용: effective_tier(FREE|PRO|ENTERPRISE|UNLIMITED) + 고정 한도 + 소속 조직(tenant)

myiam-cli service api read
myiam-cli service api update --allowed-ips "1.2.3.4,5.6.7.8"
```

API Key 생성도 마찬가지로 CLI에 **없음** — 웹 콘솔 전용.

`--preview` (아래 `main read`, `information read`, `ui read`, `*-detail read` 명령에 있음)는 패널을 로컬 HTML 페이지(`127.0.0.1:0`)로 렌더링하고 브라우저를 연다; JSON 출력에 `preview_url`이 추가된다. 이건 JSON을 보기 편하게 만든 것일 뿐, admin 콘솔의 실제 `PreviewController`(별개 기능, `docs/preview.md` 참고)를 대체하는 게 아니다.

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

`method`(IDP/EMAIL_OTP/PASSKEY/SNS_NAVER/SNS_KAKAO/SNS_GOOGLE/SNS_APPLE)는 생성 시 고정된다. SNS_* 방식은 `sns-defaults` 값이 자동으로 채워지며, `--client-id`/`--client-secret` 또는 `--from-stdin`의 `data.oauth2`로 재정의할 수 있다.

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
myiam-cli svc pd create                             # 플래그 없음, stdin: service_policy_uid, language, title, content, version_string, new_expire_at, exposed_at
                                                     # exposed_at은 이름과 달리 사실상 필수, 포맷은 yyyy-MM-ddTHH:mm:ss (오프셋/Z 없는 로컬시간, 예: 2026-07-21T00:00:00)
myiam-cli svc pd read <uid>
myiam-cli svc pd read <uid> --preview               # content HTML을 실사용자 화면 스타일 문서 카드로 렌더링
myiam-cli svc pd update                             # 플래그 없음
myiam-cli svc pd delete <uid>
myiam-cli svc pd publish <uid>                      # 이 버전 노출 / draft <uid> 로 숨김
```

`--path`는 URL 경로 전체가 아니라 bare 값이다 (`/policy/terms`가 아니라 `terms`) — 이 값이 그대로 `service information update`의 footer1target/footer2target(`footerNtype: "path"`)와 매칭된다. `title`(LanguageText, `{"ko":"..."}`)은 `label`과 별개 필드이고 `create`에는 플래그가 없다 — 정책 페이지 렌더링과 footer 연결에 필요하므로 생성 직후 `service policy update --from-stdin`으로 채워둘 것.

## 약관 (`service term`) / 약관 버전 (`service term-detail`, 별칭 `svc td`)

정책과 동일한 구조:

```bash
myiam-cli service term list            # --filter all 로 종료된 구약관까지
myiam-cli service term create --label "개인정보처리방침" --type REQUIRED   # --status 생략 시 DRAFT
myiam-cli service term read <uid>
myiam-cli service term update          # 플래그 없음, 항상 stdin
myiam-cli service term delete <uid>
myiam-cli service term publish <uid>   # 노출 / revise 후에는 기존 약관 종료 + 재동의
myiam-cli service term draft <uid>     # 숨김 (동의 이력 유지) / 구약관이 있는 계열엔 쓰지 말 것
myiam-cli service term unpublish <uid> # 발행 취소 — 개정판을 초안으로 내리고 구약관 재노출
myiam-cli service term revise <uid>    # 재동의가 필요한 개정 초안 생성
myiam-cli service term position        # 플래그 없음

myiam-cli svc td list <service-term-uid>
myiam-cli svc td create                             # 플래그 없음, stdin: service_term_uid, language, title, content, version_string
myiam-cli svc td read <uid>
myiam-cli svc td read <uid> --preview
myiam-cli svc td update                             # 플래그 없음
myiam-cli svc td delete <uid>
myiam-cli svc td publish <uid>                      # 이 버전 노출(재동의 없음) / draft <uid> 로 숨김
```

`type`은 약관은 REQUIRED/OPTIONAL, 정책은 SINGLE/BOARD — 그 외는 두 리소스가 서로 대칭이다.

### 노출 상태 (`status`) — 초안으로 준비했다가 한 번에 공개하기

약관·정책과 그 버전 모두 `status`를 갖는다. `NORMAL`이 노출, `DRAFT`가 초안(사용자에게 안 보임)이다. 초안으로 여러 건을 미리 만들어 둔 뒤 한 번에 노출로 바꾸는 것이 의도된 사용법이다. **`term create`/`policy create`는 `--status`를 생략하면 DRAFT로 만든다**(관리자 콘솔과 동일). 버전(`term-detail`/`policy-detail`) 생성은 `NORMAL`이 기본이다.

```bash
myiam-cli service term create --label "개인정보처리방침" --type REQUIRED   # DRAFT로 생성됨
myiam-cli svc td create                                                  # 본문 작성 (stdin)

myiam-cli service term publish <uid>        # 노출 (관리자 콘솔의 "지금부터 사용")
myiam-cli service term draft <uid>          # 숨김 (관리자 콘솔의 "초안으로 변경")
```

같은 `publish`/`draft` 쌍이 `service policy`, `svc td`(약관 버전), `svc pd`(정책 버전)에도 있다. 이 명령들은 패널을 `read`한 뒤 `status`만 덮어 `update`한다 — **패널 `update`에는 부분 업데이트가 없으므로** `echo '{"data":{"uid":"...","status":"NORMAL"}}' | service term update` 같은 호출은 `label`/`type`을 빈 값으로 덮어쓴다. 상태만 바꿀 땐 반드시 `publish`/`draft`를 쓴다.

`DRAFT` 버전은 버전 코드가 더 높아도 사용자에게 최신 버전으로 잡히지 않는다. 정책 버전의 `exposed_at`과는 별개로 동작한다 — `exposed_at`이 이미 지났어도 `DRAFT`면 공개되지 않는다. `read`/`list`는 관리용이라 초안도 그대로 조회되므로 `read --preview`로 공개 전 내용을 확인할 수 있다.

### 약관 개정 (`revise`) — 재동의를 받아야 할 때

약관에만 **개정 계열**이 있다. `root_service_term_uid`가 같은 약관들이 한 약관의 개정 이력이다.

```bash
myiam-cli service term revise <uid>     # 같은 계열의 새 DRAFT 약관 생성 (본문은 복사되지 않음)
myiam-cli svc td create                 # 새 약관의 본문 작성 (stdin, service_term_uid = 위에서 만든 uid)
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

1. **최초 설정** → `login` (자동 선택된 서비스가 없으면 `service list` → `service use <uid>`). `service list`가 빈 배열(`[]`)을 반환하면 아직 [myiam.io](https://myiam.io)에 가입해 관리할 서비스를 만들지 않은 것이다 — CLI 명령을 더 시도하지 말고 사용자에게 myiam.io 가입 및 서비스 생성부터 안내한 뒤, 완료되면 다시 `service list`로 확인한다.
2. **현재 설정 확인** → `service main read`로 개요 대시보드부터 보고, `information`/`ui`/`login-type`/`term`/`policy`/`field`로 세부 진입
3. **필드 하나만 변경** → 해당 플래그 사용 (예: `service ui update --theme DARK`)
4. **중첩/복잡한 데이터 변경** → `read | jq '{data:.data}' | jq로 수정 | update --from-stdin`
5. **순서 변경** (`login-type`, `policy`, `term`, `field`) → `position`에 (플래그 없이, stdin으로) `{"data":{"content":[{"id1":"<uid>"},...]}}` 형태로 전달
6. **변경 전 눈으로 확인** → 해당 `read --preview` (main/information/ui/term-detail/policy-detail만 지원); 이미 받은 JSON을 로컬에서 렌더링한 것일 뿐 실제 사용자 화면은 아님
7. **스크립팅/CI** → `login`으로 인증 정보를 저장해두면(OS 키체인, 헤드리스 환경은 `~/.myiam/credentials.yaml` 폴백) 같은 명령을 그대로 헤드리스로 사용 가능

## CLI에서 의도적으로 제외한 것

- Client Secret / API Key 생성 — 웹 콘솔 전용 (1회성 노출 UI, 명확한 사용자 동작 필요)
- 서비스 생성/파기, 운영자 역할 초대/승인, 소유자 이전 — 되돌리기 어렵거나 사람 간 권한 위임이 필요한 작업, 관리자 콘솔 전용
- 조직(테넌트) 쓰기 동작 대부분 — 생성·삭제·복구·등급변경 요청(플랫폼 승인 필요), 결제/사업자 정보(개인정보), 멤버 초대·역할 변경·소유권 이관, 서비스 이동/양수도. CLI에 있는 쓰기는 `tenant update`(이름·아이콘)뿐이다
- 최종 서비스 가입자(회원) 관리 — 이 CLI는 서비스의 "설정"을 다루는 도구이지 "가입자"를 다루는 도구가 아니다

이 목록에 있는 작업을 요청받으면 명령을 찾지 말고 **관리자 콘솔에서 수행해야 한다고 안내**한다. 조회 명령(`tenant read`/`request list`/`member list`)으로 현재 상태와 요청 결과를 확인해주는 것까지가 CLI의 역할이다.

## 스키마

`myiam-cli schema`로 전체 명령어 스펙(옵션, enum 값, stdin 필드 형태)을 기계가 읽을 수 있는 형태로 확인할 수 있다 — 명령어가 바뀌면 `cmd/schema.go`와 항상 동기화할 것.
