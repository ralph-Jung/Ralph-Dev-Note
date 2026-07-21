# OAuth2 state와 쿠키 저장소 (STATELESS 유지)

> 주제: OAuth2 state와 쿠키 저장소 · 갱신: 2026-07-21 · 상태: 진행중
> 태그: #spring-security #oauth2 #소셜로그인 #csrf #cookie #stateless #state
> 다음에 팔 것: 쿠키 직렬화 보안(역직렬화 취약점), SameSite 옵션, IF_REQUIRED로 바꾸면 이 파일 통째로 삭제 가능한지

## ① 큰 그림 (지도)

OAuth 로그인은 **"시작 요청"과 "구글 콜백"이 서로 다른 HTTP 요청**이다. 그 사이에 위조 방지용 `state`를 기억해둬야 하는데, 우리는 세션을 안 쓰기로(`SessionCreationPolicy.STATELESS`) 했다. 그래서 세션 대신 **짧은 만료의 HttpOnly 쿠키**에 저장하는 커스텀 저장소를 만들었다.

```
【나가는 반쪽】 OAuth2AuthorizationRequestRedirectFilter (= 시작 필터)
① 버튼 클릭 → GET /oauth2/authorization/google  (내 서버 주소!)
② state(랜덤) 생성 + OAuth2AuthorizationRequest 객체 조립
③ state를 "두 곳"에 심는다:
     - 쿠키에 save()      = 내가 나중에 대조하려고 (내 도메인에만 남음)
     - 구글行 URL 쿼리에   = 구글한테 전달하려고 (?state=xY9)
④ 브라우저를 구글로 redirect (Set-Cookie + Location 동시에)

【돌아오는 반쪽】 OAuth2LoginAuthenticationFilter (= 콜백 필터)
⑤ 구글 → GET /login/oauth2/code/google?code=..&state=xY9  (내 서버)
⑥ 브라우저가 쿠키 자동 첨부(같은 도메인) → load()로 꺼냄 → state 대조 → remove()로 삭제
```

- 비유: **state = 옷 보관소 번호표.** 맡길 때(save) 번호표를 쿠키에 넣어두고, 찾을 때(load) 번호표로 대조. 번호표 숫자엔 정보가 없다(랜덤).
- 핵심 파일: `HttpCookieOAuth2AuthorizationRequestRepository` (save/load/remove 3메서드).
- 이 파일이 있어야 **세션 0개로 STATELESS를 지키면서** OAuth가 돈다. → 전체 여정은 [[oauth2-google-login]]

## ② 질문 트리 (본문) ★핵심

### Q. HttpCookieOAuth2AuthorizationRequestRepository의 역할이 뭐야?
- **한줄답**: 구글 왕복 사이에 로그인 정보(state 포함)를 쿠키에 맡겼다 꺼내는 "번호표 보관함".
- **원리** — 시작 → 과정 → 결과:
    - [시작] `AuthorizationRequestRepository` 인터페이스 = save/load/remove 규격
    - [과정] 스프링 기본 구현은 세션에 저장하는데, 우리는 STATELESS라 세션을 못 씀 → 쿠키 버전을 직접 구현
    - [결과] save = 맡기기 / load = 찾기 / remove = 읽고 즉시 삭제(재사용 방지)
- **왜 이렇게**: 세션을 안 쓰기로 한 정책을 깨지 않으려고. → 아래 Q(구글로 쿠키 가나)
- **연결**: → [[spring-security-filter]]

### Q. OAuth2AuthorizationRequest는 내가 만든 거야, 라이브러리 거야?
- **한줄답**: 라이브러리 것. `spring-boot-starter-oauth2-client`가 딸려오게 한 `spring-security-oauth2-core` 안에 있다.
- **원리**:
    - import가 `org.springframework.security.oauth2.core.endpoint.OAuth2AuthorizationRequest` = core 소속
    - 보관함 인터페이스 `AuthorizationRequestRepository` = `oauth2-client` 소속
    - 둘 다 우리 패키지(com.gachon) 코드가 아님
- **연결**: → 아래 Q(state 정체)

### Q. state를 발급한다는 게, state 안에 요청 정보를 담는 거야?
- **한줄답**: 아니. state는 **정보 통이 아니라 랜덤 번호표** 하나(CSRF 방어용).
- **원리**:
    - 쿠키에 저장되는 건 `OAuth2AuthorizationRequest` 객체 전체(state + redirectUri + clientId + scope)
    - state는 그 객체의 한 필드일 뿐
    - state가 랜덤 → 공격자가 못 맞춤 → 위조 콜백을 걸러냄
- **연결**: → 아래 Q(언제 구글에 보냈어)

---

### Q. 프론트에서 /oauth2/authorization/google로 연결하면 내 서버로 들어와?
- **한줄답**: 맞음. 그 URL은 **구글이 아니라 내 백엔드 서버 주소**다.
- **원리** — 시작 → 과정 → 결과:
    - [시작] `localhost:8080/oauth2/authorization/google` = 앞부분이 내 서버
    - [과정] `OAuth2AuthorizationRequestRedirectFilter`(시작 필터)가 이 URL을 낚아채 save 후 구글로 redirect
    - [결과] 구글과 실제 통신은 내 서버가 대신, 프론트는 브라우저를 이 입구로 보내기만
- **주의**: 프론트는 `fetch/axios` 말고 **브라우저 전체 이동**(`window.location`/`<a>`)이어야 함 — redirect 흐름이라 fetch는 CORS로 깨짐.
- **연결**: → 아래 Q(save가 어떻게)

### Q. save 메서드가 이해가 안 돼. addCookie가 어떻게 브라우저에 저장돼?
> 그때 왜 궁금했나: 자바 메서드 하나 부른 게 어떻게 브라우저 안 쿠키가 되는지가 안 그려졌다.
- **한줄답**: 객체를 문자열로 바꿔 response에 Set-Cookie로 얹으면, 브라우저가 HTTP 규칙상 자동 저장한다.
- **원리** — 시작 → 과정 → 결과:
    - [시작] `OAuth2AuthorizationRequestRedirectFilter`가 만든 `OAuth2AuthorizationRequest` 객체를 `saveAuthorizationRequest()`가 받음
    - [과정] `SerializationUtils.serialize` + Base64 = 객체를 문자열로 압축
    - [과정] `addCookie(response, ...)` = response에 `Set-Cookie` 헤더 등록
    - [결과] 브라우저가 `Set-Cookie`를 받으면 자동 저장 (우리가 "저장해"라고 명령 안 함)
- **연결**: → 아래 Q(addCookie 인자)

### Q. addCookie의 인자 4개가 다 저장되는 거야?
- **한줄답**: 아니. `response`는 배달부(저장 안 됨), 나머지 3개(이름/값/수명)가 쿠키의 각 부분이 됨.
- **원리**:
    - `response` = 쿠키 실어보낼 통로 (저장 안 됨)
    - `OAUTH2_AUTH_REQUEST_COOKIE` = 쿠키 이름(키)
    - `value` = 내용물 (여기 OAuth2AuthorizationRequest 객체가 문자열로 압축돼 들어감)
    - `180` = 수명(Max-Age)
- **결과물**: `Set-Cookie: oauth2_auth_request=...; Max-Age=180; HttpOnly` 헤더 한 줄
- **연결**: → 아래 Q(header에 담기)

### Q. addCookie가 response 객체 안에 header로 쿠키를 담는 거지?
- **한줄답**: 맞음. `response`에 `Set-Cookie` 헤더 형태로 등록하는 것.
- **원리** — 시작 → 과정 → 결과:
    - [시작] `addCookie(response, ...)` 호출 = Cookie 객체를 Set-Cookie 헤더 문자열로 변환
    - [과정] `response.addCookie()`가 그 헤더를 응답 객체에 얹음 (호출 직후엔 response 안에만 있음)
    - [결과] 응답이 실제 전송될 때 네트워크로 나감
- **왜 순서 중요**: `sendRedirect`(응답 커밋)보다 **먼저** addCookie 해야 함. 커밋 후엔 헤더 추가 불가.
- **연결**: → 아래 Q(sendRedirect 헤더)

### Q. sendRedirect 할 때 쿠키를 header에 담아 보내는 거야?
- **한줄답**: 쿠키는 `Set-Cookie` 응답 헤더에 실려 나간다. 단, sendRedirect가 담는 게 아니라 이미 등록된 게 같이 나가는 것.
- **원리**:
    - `addCookie` = `Set-Cookie` 헤더 등록 / `sendRedirect` = `Location` 헤더 + 302 등록
    - 둘은 같은 response의 다른 헤더 → 응답 전송 시 함께 나감
- **헷갈림 주의**: `Set-Cookie`(서버→브라우저 "저장해") vs `Cookie`(브라우저→서버 "가져왔어") = 방향 반대.
- **연결**: → 아래 Q(언제 구글에)

---

### Q. state를 언제 구글에 보냈어? 쿠키는 구글이 안 받았잖아?
> 그때 왜 궁금했나: 쿠키가 구글로 안 가는데 콜백 URL에 state가 있는 게 모순처럼 느껴졌다.
- **한줄답**: state는 쿠키가 아니라 **구글行 redirect URL의 쿼리스트링**으로 보냈다(쿠키와 URL 두 곳에 심음).
- **원리** — 시작 → 과정 → 결과:
    - [시작] `OAuth2AuthorizationRequestRedirectFilter`가 state를 두 통로에 동시에 심음: ① 쿠키(내 대조용) ② `...&state=xY9`(구글 전달용)
    - [과정] 구글은 쿠키가 아니라 URL로 state를 받음 → OAuth 규칙상 콜백 때 그 값을 그대로 되돌려줌(echo)
    - [결과] 콜백엔 state가 두 벌 = URL발(구글 echo) + 쿠키발(내 저장분) → 대조 가능
- **왜 이렇게**: 공격자는 HttpOnly 쿠키를 못 읽어 URL state를 쿠키값과 못 맞춤 → 위조 차단(CSRF).
- **연결**: → 아래 Q(넣는 주체), → [[oauth2-google-login]]의 state 대조

### Q. URL 쿼리에 state를 넣는 건 누구 역할이야?
- **한줄답**: 시작 필터 `OAuth2AuthorizationRequestRedirectFilter`.
- **원리**:
    - `OAuth2AuthorizationRequestRedirectFilter` = state 생성 + 객체 조립 + 쿠키 save + 구글行 URL 조립(state 삽입) + redirect 담당
    - (엄밀히는 내부 도우미 `OAuth2AuthorizationRequestResolver`가 객체 조립)
- **연결**: → [[spring-security-filter]]의 필터 계층

### Q. 구글이 내 서버로 redirect할 때, 브라우저가 쿠키를 구글로 보내는 거야?
> 그때 왜 궁금했나: 쿠키가 대체 어느 요청에 붙는지(구글이냐 내 서버냐)가 헷갈렸다.
- **한줄답**: 아니. 우리 쿠키는 **구글엔 안 붙고**, 나중에 내 서버로 돌아올 때만 붙는다.
- **원리** — 시작 → 과정 → 결과:
    - [시작] 쿠키는 "발급한 도메인(localhost:8080)에만" 자동 첨부되는 규칙
    - [과정] 브라우저가 구글(accounts.google.com)로 갈 땐 도메인이 달라 우리 쿠키 안 붙음
    - [결과] 내 서버 콜백(localhost:8080)으로 돌아올 때 같은 도메인이라 `Cookie` 헤더에 자동 첨부 → `OAuth2LoginAuthenticationFilter`가 load로 꺼냄
- **왜 다행**: 이 규칙 덕에 우리 로그인 쿠키가 남의 서버(구글)로 절대 안 샘.
- **연결**: → [[oauth2-google-login]]의 콜백 처리

## ③ 용어 카드 (역참조)
- **state**: OAuth 위조 방지용 랜덤 문자열(번호표). 정보 없음. 쿠키+URL 두 곳에 심음. → Q.state 정체
- **OAuth2AuthorizationRequest**: state/redirectUri/clientId/scope를 담은 라이브러리 객체. 쿠키에 통째로 저장. → Q.라이브러리 거야
- **AuthorizationRequestRepository**: save/load/remove 규격 인터페이스. 기본=세션, 우리=쿠키. → Q.역할
- **Set-Cookie / Cookie**: 전자=서버→브라우저(저장해, 응답헤더), 후자=브라우저→서버(가져왔어, 요청헤더). → Q.sendRedirect
- **쿠키 도메인 규칙**: 쿠키는 발급 도메인에만 자동 첨부. 구글엔 안 붙는 이유. → Q.구글로 보내나
- **직렬화/역직렬화**: 객체↔문자열 변환(+Base64). 쿠키는 문자열만 담아서 필요. → Q.save 이해

## ④ 내가 틀렸던 것 (오개념 로그) ★가치 높음
| 내가 생각했던 것 | 실제 |
|---|---|
| state 안에 요청 정보가 담겨 있다 | state는 랜덤 번호표. 정보는 OAuth2AuthorizationRequest 객체에 |
| save는 구글로 보내려고 저장하는 것 | 지금 쓰려는 게 아니라 "나중에 돌아올 콜백"에서 대조하려고 |
| state를 쿠키로 구글에 보냈다 | 쿠키 아니라 구글行 URL 쿼리로 보냄. 구글이 그걸 echo |
| 브라우저가 쿠키를 구글로 보낸다 | 도메인 달라서 구글엔 안 붙음. 내 서버로 돌아올 때만 |
| addCookie의 response도 쿠키에 저장된다 | response는 배달부. 이름/값/수명만 쿠키가 됨 |
| sendRedirect가 쿠키를 담아 보낸다 | addCookie가 미리 Set-Cookie 등록, redirect 응답에 같이 실림 |
