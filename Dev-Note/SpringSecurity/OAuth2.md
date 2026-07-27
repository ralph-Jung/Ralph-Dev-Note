# OAuth2 소셜 로그인 (구글)

> **주제**: OAuth2 소셜 로그인 — 전체 흐름 + state 쿠키 저장소(STATELESS) · 갱신: 2026-07-28 · 상태: 진행중
> **태그**: #OAuth2 #소셜로그인 #google #state #쿠키 #spring-security

## ① 큰 그림 (지도)

핵심 한 줄: **구글은 "누구인지"만 확인해주고, 이후 인증은 우리 JWT를 쓴다.** 소셜 로그인은 "JWT를 발급받는 새 입구"일 뿐. 그리고 OAuth 로그인은 **"시작 요청"과 "구글 콜백"이 서로 다른 HTTP 요청**이라, 그 사이 위조 방지용 `state`를 **STATELESS(세션 0개)** 로 유지하려고 httpOnly 쿠키에 맡긴다.

```
[전체 여정]
① 버튼 → GET /oauth2/authorization/google  (내 서버)
    └ 시작 필터(OAuth2AuthorizationRequestRedirectFilter):
        state를 쿠키에 save() + 구글行 URL 쿼리(?state=)에 심고 → 구글로 redirect
② 구글 로그인/동의
③ 구글 → GET /login/oauth2/code/google?code=..&state=..  (내 서버 콜백)
    └ 콜백 필터(OAuth2LoginAuthenticationFilter):
        쿠키 load() + code/state로 봉투 포장 → authenticate()
④ ProviderManager → OAuth2LoginAuthenticationProvider
    ├ OAuth2AuthorizationCodeAuthenticationProvider = state 대조 + code→token 교환(뒷채널)
    └ CustomOAuth2UserService = userinfo 조회 + DB upsert → OAuth2User
⑤ 되감기: OAuth2User → 인증됨 토큰 → 부모필터가 SecurityContext 저장 + SuccessHandler 호출
⑥ OAuth2LoginSuccessHandler: 우리 JWT 발급 → 프론트로 #token= redirect
⑦ 이후 API: Authorization Bearer → JwtFilter 흐름  → [[TokenAuth]]
```

**state를 쿠키로 지키는 이유 (나가는/돌아오는 반쪽):**
- state = **옷 보관소 번호표**(랜덤, 정보 없음). 시작 필터가 쿠키(내 대조용) + URL(구글 전달용) **두 곳**에 심음.
- 구글은 쿠키를 못 받고(도메인 다름) URL state만 echo → 콜백에 state **두 벌**(쿠키발+URL발)이 모여 대조 → 위조(CSRF) 차단.
- 세션 대신 짧은 HttpOnly 쿠키에 저장하는 커스텀 저장소: `HttpCookieOAuth2AuthorizationRequestRepository` (save/load/remove).

- **분업 3단**: 필터 = 접수 / Provider = 실무(state대조+토큰교환+userinfo) / Handler = 마무리(JWT 발급).
- 로컬 로그인과 같은 뼈대: `authenticationManager.authenticate()` → Provider. 다른 건 "인증 방식"뿐. → [[Authentication]] (인증 흐름), → [[FilterChain]] (필터 계층)

## ② 질문 트리 (본문)

### 2026-07-21

#### Q. HttpCookieOAuth2AuthorizationRequestRepository의 역할이 뭐야?
- **한줄답**: 구글 왕복 사이에 로그인 정보(state 포함)를 쿠키에 맡겼다 꺼내는 ==번호표 보관함==.
- **원리** — 시작 → 과정 → 결과:
    - [시작] `AuthorizationRequestRepository` 인터페이스 = save/load/remove 규격
    - [과정] 스프링 기본 구현은 세션에 저장하는데, 우리는 STATELESS라 세션을 못 씀 → 쿠키 버전을 직접 구현
    - [결과] save = 맡기기 / load = 찾기 / remove = 읽고 즉시 삭제(재사용 방지)
- **왜 이렇게**: 세션을 안 쓰기로 한 정책을 깨지 않으려고.
- **연결**: → [[Authentication]] (세션 대신 STATELESS를 택한 배경), → 아래 Q(구글로 쿠키 가나)

#### Q. OAuth2AuthorizationRequest는 내가 만든 거야, 라이브러리 거야?
- **한줄답**: ==라이브러리 것==. `spring-boot-starter-oauth2-client`가 딸려오게 한 `spring-security-oauth2-core` 안에 있다.
- **원리**:
    - import가 `org.springframework.security.oauth2.core.endpoint.OAuth2AuthorizationRequest` = core 소속
    - 보관함 인터페이스 `AuthorizationRequestRepository` = `oauth2-client` 소속
    - 둘 다 우리 패키지(com.gachon) 코드가 아님
- **연결**: → 아래 Q(state 정체)

#### Q. state를 발급한다는 게, state 안에 요청 정보를 담는 거야?
- **한줄답**: 아니. state는 ==정보 통이 아니라 랜덤 번호표 하나(CSRF 방어용)==.
- **원리**:
    - 쿠키에 저장되는 건 `OAuth2AuthorizationRequest` 객체 전체(state + redirectUri + clientId + scope)
    - state는 그 객체의 한 필드일 뿐
    - state가 랜덤 → 공격자가 못 맞춤 → 위조 콜백을 걸러냄
- **연결**: → 아래 Q(언제 구글에 보냈어)

#### Q. 프론트에서 /oauth2/authorization/google로 연결하면 내 서버로 들어와?
- **한줄답**: 맞음. 그 URL은 ==구글이 아니라 내 백엔드 서버 주소==다.
- **원리** — 시작 → 과정 → 결과:
    - [시작] `localhost:8080/oauth2/authorization/google` = 앞부분이 내 서버
    - [과정] `OAuth2AuthorizationRequestRedirectFilter`(시작 필터)가 이 URL을 낚아채 save 후 구글로 redirect
    - [결과] 구글과 실제 통신은 내 서버가 대신, 프론트는 브라우저를 이 입구로 보내기만
- **주의**: 프론트는 `fetch/axios` 말고 **브라우저 전체 이동**(`window.location`/`<a>`)이어야 함 — redirect 흐름이라 fetch는 CORS로 깨짐.
- **연결**: → [[ServletFilter]] (save/addCookie가 어떻게 브라우저 쿠키가 되는가 — 쿠키·헤더 메커니즘), → 아래 Q(언제 구글에)

#### Q. state를 언제 구글에 보냈어? 쿠키는 구글이 안 받았잖아?
> 그때 왜 궁금했나: 쿠키가 구글로 안 가는데 콜백 URL에 state가 있는 게 모순처럼 느껴졌다.
- **한줄답**: state는 쿠키가 아니라 ==구글行 redirect URL의 쿼리스트링으로 보냈다(쿠키와 URL 두 곳에 심음)==.
- **원리** — 시작 → 과정 → 결과:
    - [시작] `OAuth2AuthorizationRequestRedirectFilter`가 state를 두 통로에 동시에 심음: ① 쿠키(내 대조용) ② `...&state=xY9`(구글 전달용)
    - [과정] 구글은 쿠키가 아니라 URL로 state를 받음 → OAuth 규칙상 콜백 때 그 값을 그대로 되돌려줌(echo)
    - [결과] 콜백엔 state가 두 벌 = URL발(구글 echo) + 쿠키발(내 저장분) → 대조 가능
- **왜 이렇게**: 공격자는 HttpOnly 쿠키를 못 읽어 URL state를 쿠키값과 못 맞춤 → 위조 차단(CSRF).
- **연결**: → 아래 Q(넣는 주체), → 아래 Q(state 대조+교환)

#### Q. URL 쿼리에 state를 넣는 건 누구 역할이야?
- **한줄답**: ==시작 필터 `OAuth2AuthorizationRequestRedirectFilter`==.
- **원리**:
    - `OAuth2AuthorizationRequestRedirectFilter` = state 생성 + 객체 조립 + 쿠키 save + 구글行 URL 조립(state 삽입) + redirect 담당
    - (엄밀히는 내부 도우미 `OAuth2AuthorizationRequestResolver`가 객체 조립)
- **연결**: → [[FilterChain]] (필터 계층)

#### Q. 구글이 내 서버로 redirect할 때, 브라우저가 쿠키를 구글로 보내는 거야?
> 그때 왜 궁금했나: 쿠키가 대체 어느 요청에 붙는지(구글이냐 내 서버냐)가 헷갈렸다.
- **한줄답**: 아니. ==우리 쿠키는 구글엔 안 붙고, 나중에 내 서버로 돌아올 때만 붙는다==.
- **원리** — 시작 → 과정 → 결과:
    - [시작] 쿠키는 "발급한 도메인(localhost:8080)에만" 자동 첨부되는 규칙
    - [과정] 브라우저가 구글(accounts.google.com)로 갈 땐 도메인이 달라 우리 쿠키 안 붙음
    - [결과] 내 서버 콜백(localhost:8080)으로 돌아올 때 같은 도메인이라 `Cookie` 헤더에 자동 첨부 → `OAuth2LoginAuthenticationFilter`가 load로 꺼냄
- **왜 다행**: 이 규칙 덕에 우리 로그인 쿠키가 남의 서버(구글)로 절대 안 샘.
- **연결**: → 아래 Q(콜백 필터가 재료 넘기기)

#### Q. 구글이 로그인 끝나면 토큰을 (bearer로) 주는 거야?
> 그때 왜 궁금했나: "로그인 성공 = 토큰 받음"이라 생각했는데 흐름이 더 복잡해서.
- **한줄답**: 아니. ==구글은 토큰이 아니라 code(일회용 교환권)를 URL 쿼리로 준다== (Authorization Code 방식).
- **원리** — 시작 → 과정 → 결과:
    - [시작] 로그인/동의 끝 → 구글이 브라우저를 내 서버로 redirect
    - [과정] `/login/oauth2/code/google?code=..&state=..` = 토큰이 아니라 code를 URL 쿼리로 전달
    - [결과] code는 client_secret 없이는 토큰으로 못 바꿈 → 서버가 뒷채널로 교환
- **왜 이렇게**: 토큰이 브라우저/URL을 거치면 로그·history로 샘 → 쓸모없는 code만 흘리고 진짜 토큰은 서버↔서버로만.
- **연결**: → 아래 Q(code 교환), → 위 Q(쿠키 load/콜백)

#### Q. 콜백 필터가 받은 재료를 어떻게 넘기나?
- **한줄답**: ==쿠키에서 복원한 요청 + URL의 code/state를 한 봉투에 담아 `authenticate()`에 넘긴다==.
- **원리** — 시작 → 과정 → 결과:
    - [시작] `OAuth2LoginAuthenticationFilter`가 콜백 요청 받음 (state 비교도 토큰 교환도 안 함)
    - [과정] `removeAuthorizationRequest()` = 쿠키의 요청 복원 / `code,state` = `OAuth2AuthorizationResponse` 생성
    - [결과] 둘을 봉투(Exchange)에 묶고 토큰으로 포장해 넘김 (다음 Q)
- **연결**: → 아래 Q(G 3객체), → 아래 Q(Provider 위임)

#### Q. 봉투 만들 때 나오는 객체 3개가 뭔데? (G 단계)
> 그때 왜 궁금했나: 코드에 갑자기 낯선 타입 3개가 한 줄에 나와서 각각이 뭔지 안 잡혔다.
- **한줄답**: ==구글 설정 + 요청·응답 봉투를 묶어 Provider에 넘길 "인증 요청서"를 만든다==.
- **원리** — 세 객체 각각:
    - `ClientRegistration` = application 설정의 구글 설정 뭉치(clientId/clientSecret/tokenUri/userInfoUri/scopes). registrationId("google")로 조회. 뒤에서 토큰 교환·userinfo에 씀.
    - `OAuth2AuthorizationExchange` = Request(쿠키 복원, state 원본) + Response(URL의 code+state)를 한 쌍으로 묶은 상자(DTO). 두 state를 나란히 담아 대조 가능케 함.
    - `OAuth2LoginAuthenticationToken` = Provider에 넘길 "인증 요청서"(authenticated=false). 로컬 로그인의 `UsernamePasswordAuthenticationToken`과 같은 위치.
- **Exchange 관계**: Response가 Exchange 안 한 칸에 **중첩(포장)**된 것. 타입이 Response→Exchange로 "변한" 게 아님.
- **왜 묶나**: state를 대조하려면 Request(내 원본)와 Response(구글 echo) 둘 다 필요해서.
- **연결**: → 아래 Q(Provider 위임), → 아래 Q(state 대조)

#### Q. AuthenticationManager가 봉투를 받은 다음 Provider로 어떻게 가나?
- **한줄답**: ==ProviderManager가 OAuth2LoginAuthenticationProvider에 위임하고, 그게 다시 code용 Provider에 "재포장"해 넘긴다==.
- **원리** — 시작 → 과정 → 결과:
    - [시작] `AuthenticationManager`(구현체 `ProviderManager`) = Provider가 아니라 관리자. 결과를 전달만.
    - [과정] `OAuth2LoginAuthenticationProvider`가 봉투를 풀어 `clientRegistration + exchange`를 꺼냄 → **새 타입** `OAuth2AuthorizationCodeAuthenticationToken`으로 재포장
    - [결과] `OAuth2AuthorizationCodeAuthenticationProvider.authenticate(새 토큰)` 호출
- **왜 재포장**: code Provider는 로그인 전용이 아닌 범용 부품이라 자기 입력 타입을 요구 → 어댑터 역할.
- **연결**: → 아래 Q(state 대조+교환)

#### Q. state 대조와 code→token 교환은 누가 해?
- **한줄답**: 필터가 아니라 ==`OAuth2AuthorizationCodeAuthenticationProvider`==.
- **원리** — 시작 → 과정 → 결과:
    - [시작] 봉투 안 exchange에서 두 state 꺼냄: `resp.getState()`(URL) vs `req.getState()`(쿠키)
    - [과정] 같으면 통과, 다르면 위조로 차단
    - [과정] `accessTokenResponseClient.getTokenResponse()` = 구글 토큰 엔드포인트에 직접 POST(code+client_id+secret)
    - [결과] 구글이 `access_token`(+id_token) 응답 → 토큰에 담아 반환
- **연결**: → 아래 Q(userinfo), → 위 Q(state 두 벌)

#### Q. access token으로 어떻게 사용자를 찾아?
> 그때 왜 궁금했나: access token으로 우리 DB를 뒤지는 줄 알았다.
- **한줄답**: 우리가 찾는 게 아니라, ==토큰을 들고 구글 userinfo에 물어보면 구글이 찾아준다==.
- **원리** — 시작 → 과정 → 결과:
    - [시작] `OAuth2LoginAuthenticationProvider`가 access_token으로 `userService.loadUser()` 호출
    - [과정] `CustomOAuth2UserService`의 `delegate.loadUser` = 구글 userinfo에 `Authorization: Bearer <access_token>`로 GET
    - [결과] access_token = 구글에 내미는 출입증. 구글이 token↔user 매핑을 조회해 `{sub, email, email_verified}` 응답
- **연결**: → 아래 Q(우리 서비스 실행), → 아래 Q(upsert)

#### Q. {sub,email} 받은 다음에 우리 서비스가 실행되는 거야?
- **한줄답**: 반대. ==우리 서비스가 실행돼서 그 안에서 {sub,email}을 받아온다==.
- **원리** — 시작 → 과정 → 결과:
    - [시작] `OAuth2LoginAuthenticationProvider`가 `this.userService.loadUser(access_token)` 호출 = 방아쇠
    - [과정] 우리 `CustomOAuth2UserService.loadUser` 시작 → 내부 `delegate.loadUser`가 구글 userinfo GET → 값 도착
    - [결과] 값 받는 일 자체가 우리 서비스 안에서 일어남
- **왜 우리 게 실행**: SecurityConfig `.userService(customOAuth2UserService)`가 Provider 필드에 우리 빈을 주입해서.
- **연결**: → 아래 Q(upsert)

#### Q. 받은 사용자 정보를 DB에 어떻게 저장해? 중복이면 합쳐?
> 그때 왜 궁금했나: 같은 사람이 구글로 또 로그인하면 계정이 새로 생기나 걱정됐다.
- **한줄답**: 3분기 — ==① sub로 기존 구글계정 찾으면 그대로 ② (검증된)이메일로 기존계정 찾으면 합치기 ③ 없으면 신규 저장==.
- **원리** — `OAuth2UserUpsertService.upsertGoogleUser` (`@Transactional`):
    - `findByProviderAndProviderId(GOOGLE, sub)` 히트 = 재로그인, 그대로 반환 (sub는 안 변하는 안정 식별자)
    - 미스 + emailVerified → `findByEmail` 히트 = `existing.linkGoogle(sub)`로 기존 계정에 provider/providerId만 채움(통합). save() 없이 JPA dirty checking이 커밋 시 UPDATE
    - 다 미스 = `save(User.ofGoogle(...))` 신규 INSERT
- **왜 emailVerified 체크**: 미검증 이메일로 연동하면 남의 이메일로 가짜 구글계정 만들어 계정 탈취 가능 → 검증된 것만 연동.
- **트랜잭션 경계**: userinfo(외부 HTTP)는 트랜잭션 밖(delegate)에서 끝내고 DB만 @Transactional = 커넥션 풀 고갈 방지. self-invocation 프록시 함정 피하려 upsert를 별도 빈으로 분리.
- **연결**: → 아래 Q(OAuth2User 반환)

#### Q. OAuth2User 반환된 다음엔? 성공 핸들러는 어떻게 연결돼?
- **한줄답**: ==Provider가 "인증됨 토큰"으로 포장 → 필터로 되감김 → 부모필터가 SecurityContext 저장 + SuccessHandler 호출==.
- **원리** — 시작 → 과정 → 결과:
    - [시작] `OAuth2LoginAuthenticationProvider`가 OAuth2User를 principal로 담은 authenticated 토큰 생성
    - [과정] manager가 그대로 필터에 반환 → `attemptAuthentication`이 `OAuth2AuthenticationToken`으로 변환해 return
    - [결과] 부모 `AbstractAuthenticationProcessingFilter`가 성공으로 판정해 successHandler 호출
- **되감기란**: 자식(필터)의 인증 결과가 부모의 try 블록으로 돌아와 후처리(저장+핸들러)로 이어지는 것. 상세 → [[Authentication]] Q.doFilter랑 successfulAuthentication은 부모 메서드야
- **연결**: → 아래 Q(핸들러 내부)

#### Q. 성공 핸들러가 실행된 다음엔? (OAuth 흐름의 끝)
- **한줄답**: ==email로 우리 JWT 발급 → 쿠키 정리 → 토큰을 URL fragment(#token=)에 실어 프론트로 redirect==.
- **원리** — 시작 → 과정 → 결과:
    - [시작] `OAuth2LoginSuccessHandler`가 principal(OAuth2User)에서 email(=loginId) 추출
    - [과정] `jwtProvider.createAccessToken(loginId)` = 로컬 로그인과 동일한 JWT 발급 + removeAuthorizationRequest로 쿠키 정리
    - [결과] `#token=`으로 프론트에 redirect → 프론트가 `window.location.hash`로 토큰 꺼내 저장
- **왜 fragment**: 쿼리스트링(?)은 서버 로그·Referer에 남아 유출 위험, fragment(#)는 서버로 안 감.
- **연결**: → 이후 API는 [[TokenAuth]] Q.JwtFilter는 어떻게 작성하나

## ③ 용어 카드 (역참조)

> [!quote]- 용어 14개
> - **state**: OAuth 위조 방지용 랜덤 문자열(번호표). 정보 없음. 쿠키+URL 두 곳에 심음. → Q.state 정체
> - **OAuth2AuthorizationRequest**: state/redirectUri/clientId/scope를 담은 라이브러리 객체. 쿠키에 통째로 저장. → Q.라이브러리 거야
> - **AuthorizationRequestRepository**: save/load/remove 규격 인터페이스. 기본=세션, 우리=쿠키. → Q.역할
> - **쿠키 도메인 규칙**: 쿠키는 발급 도메인에만 자동 첨부. 구글엔 안 붙는 이유. → Q.구글로 보내나
> - **Authorization Code 방식**: 구글은 code만 주고, 서버가 뒷채널로 token 교환. 토큰 유출 방지. → Q.토큰 주나
> - **ClientRegistration**: 구글 설정 뭉치(id/secret/tokenUri/userInfoUri). → Q.G 3객체
> - **OAuth2AuthorizationExchange**: Request(쿠키)+Response(URL code/state)를 묶은 봉투. → Q.G 3객체
> - **OAuth2LoginAuthenticationToken**: Provider에 넘길 인증 요청서. UsernamePasswordAuthenticationToken과 동급. → Q.G 3객체
> - **ProviderManager**: AuthenticationManager 구현체. Provider들의 관리자(Provider 아님). → Q.Provider 위임
> - **OAuth2AuthorizationCodeAuthenticationProvider**: state 대조 + code→token 교환 담당. → Q.대조 교환
> - **CustomOAuth2UserService**: 구글 userinfo 조회 + DB upsert. delegate가 실제 HTTP. → Q.우리 서비스
> - **linkGoogle / dirty checking**: 기존 계정에 provider/providerId만 채움. save 없이 UPDATE 자동. → Q.upsert
> - **되감기**: 자식 필터 결과가 부모 try 블록으로 돌아와 후처리로 이어지는 구조. → Q.OAuth2User 반환
> - **URL 값 전달 3방식**: path variable(식별) / query string(조건) / fragment(#token 클라전용·서버 안감). → Q.핸들러 끝

## ④ 내가 틀렸던 것 (오개념 로그)

> [!quote]- 오개념 12건
> | 내가 생각했던 것 | 실제 |
> |---|---|
> | state 안에 요청 정보가 담겨 있다 | state는 랜덤 번호표. 정보는 OAuth2AuthorizationRequest 객체에 |
> | save는 구글로 보내려고 저장하는 것 | 지금 쓰려는 게 아니라 "나중에 돌아올 콜백"에서 대조하려고 |
> | state를 쿠키로 구글에 보냈다 | 쿠키 아니라 구글行 URL 쿼리로 보냄. 구글이 그걸 echo |
> | 브라우저가 쿠키를 구글로 보낸다 | 도메인 달라서 구글엔 안 붙음. 내 서버로 돌아올 때만 |
> | 구글이 로그인 끝나면 토큰을 (bearer로) 준다 | code(교환권)를 URL 쿼리로 줌. 토큰은 서버가 뒷채널로 받아옴 |
> | 콜백 응답에 token이 들어있다 | code + state만. token 아님 |
> | code용 Provider가 원래 토큰을 통째로 받는다 | 원래 토큰을 풀어 새 타입(CodeAuthToken)으로 재포장해 넘김 |
> | Response 타입이 Exchange로 바뀐다 | 안 바뀜. Response가 Exchange 안에 중첩(포장)됨 |
> | access token으로 우리 DB를 검색한다 | 구글 userinfo에 내미는 출입증. 찾는 건 구글 |
> | {sub,email} 받은 뒤 우리 서비스 실행 | 우리 서비스가 실행돼서 그 안에서 받아옴 |
> | AuthenticationManager가 첫 Provider다 | Provider 아님. Provider들의 관리자(ProviderManager) |
> | "Bearer 헤더"가 맞는 명칭 | Authorization 헤더 + Bearer 스킴. "Bearer 토큰"은 토큰 지칭으로 맞음 |
