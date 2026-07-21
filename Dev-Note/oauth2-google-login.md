# 구글 OAuth2 소셜 로그인 전체 흐름 (서버 리다이렉트)

> 주제: 구글 OAuth2 소셜 로그인 전체 흐름 · 갱신: 2026-07-21 · 상태: 진행중
> 태그: #spring-security #oauth2 #소셜로그인 #google #authorization-code #jwt #authenticationmanager #provider #userinfo
> 다음에 팔 것: Refresh 토큰, id_token(OIDC) 검증, HttpOnly 쿠키로 JWT 전달 시 JwtFilter 수정, 카카오/네이버 추가

## ① 큰 그림 (지도)

핵심 한 줄: **구글은 "누구인지"만 확인해주고, 이후 인증은 우리 JWT를 쓴다.** 소셜 로그인은 "JWT를 발급받는 새 입구"일 뿐.

```
① 버튼 → GET /oauth2/authorization/google  (내 서버)
    └ 시작 필터: state 쿠키 저장 + 구글로 redirect          → 상세 [[oauth2-state-cookie]]
② 구글 로그인/동의
③ 구글 → GET /login/oauth2/code/google?code=..&state=..  (내 서버 콜백)
    └ OAuth2LoginAuthenticationFilter(콜백 필터):
         load()로 쿠키 꺼냄 + code/state로 봉투 포장 → authenticate()에 넘김
④ ProviderManager → OAuth2LoginAuthenticationProvider
    ├ OAuth2AuthorizationCodeAuthenticationProvider = state 대조 + code→token 교환(뒷채널)
    └ CustomOAuth2UserService = userinfo 조회 + DB upsert → OAuth2User 반환
⑤ 되감기: OAuth2User → 인증됨 토큰 → 필터로 반환
    └ 부모필터가 SecurityContext 저장 + SuccessHandler 호출  → 상세 [[spring-security-filter]]
⑥ OAuth2LoginSuccessHandler: 우리 JWT 발급 → 프론트로 #token= redirect
⑦ 이후 API: Authorization 헤더 Bearer → 기존 JwtFilter 흐름  → [[spring-security-auth-flow]]
```

- **분업 3단**: 필터 = 접수 / Provider = 실무(state대조+토큰교환+userinfo) / Handler = 마무리(JWT 발급).
- 로컬 로그인과 같은 뼈대: `authenticationManager.authenticate()` → Provider. 다른 건 "인증 방식"뿐. → [[spring-security-auth-flow]]

## ② 질문 트리 (본문) ★핵심

### Q. 구글이 로그인 끝나면 토큰을 (bearer로) 주는 거야?
> 그때 왜 궁금했나: "로그인 성공 = 토큰 받음"이라 생각했는데 흐름이 더 복잡해서.
- **한줄답**: 아니. 구글은 **토큰이 아니라 code(일회용 교환권)**를 URL 쿼리로 준다 (Authorization Code 방식).
- **원리** — 시작 → 과정 → 결과:
    - [시작] 로그인/동의 끝 → 구글이 브라우저를 내 서버로 redirect
    - [과정] `/login/oauth2/code/google?code=..&state=..` = 토큰이 아니라 code를 URL 쿼리로 전달
    - [결과] code는 client_secret 없이는 토큰으로 못 바꿈 → 서버가 뒷채널로 교환
- **왜 이렇게**: 토큰이 브라우저/URL을 거치면 로그·history로 샘 → 쓸모없는 code만 흘리고 진짜 토큰은 서버↔서버로만.
- **연결**: → 아래 Q(code 교환), → [[oauth2-state-cookie]]의 콜백

### Q. 콜백 필터가 받은 재료를 어떻게 넘기나?
- **한줄답**: 쿠키에서 복원한 요청 + URL의 code/state를 한 봉투에 담아 `authenticationManager.authenticate()`에 넘긴다.
- **원리** — 시작 → 과정 → 결과:
    - [시작] `OAuth2LoginAuthenticationFilter`가 콜백 요청 받음 (state 비교도 토큰 교환도 안 함)
    - [과정] `removeAuthorizationRequest()` = 쿠키의 요청 복원 / `code,state` = `OAuth2AuthorizationResponse` 생성
    - [결과] 둘을 봉투(Exchange)에 묶고 토큰으로 포장해 넘김 (다음 Q)
- **연결**: → 아래 Q(G 3객체), → 아래 Q(Provider 위임)

### Q. ★ 봉투 만들 때 나오는 객체 3개가 뭔데? (G 단계)
> 그때 왜 궁금했나: 코드에 갑자기 낯선 타입 3개가 한 줄에 나와서 각각이 뭔지 안 잡혔다.

- **한줄답**: 구글 설정 + 요청·응답 봉투를 묶어 Provider에 넘길 "인증 요청서"를 만든다.
- **원리** — 세 객체 각각:
    - `ClientRegistration` = application.properties의 구글 설정 뭉치(clientId/clientSecret/tokenUri/userInfoUri/scopes). registrationId("google")로 조회. 뒤에서 토큰 교환·userinfo에 씀.
    - `OAuth2AuthorizationExchange` = Request(쿠키 복원, state 원본) + Response(URL의 code+state)를 한 쌍으로 묶은 상자(DTO). 두 state를 나란히 담아 대조 가능케 함.
    - `OAuth2LoginAuthenticationToken` = Provider에 넘길 "인증 요청서"(authenticated=false). 로컬 로그인의 `UsernamePasswordAuthenticationToken`과 같은 위치.
- **Exchange 관계**: Response가 Exchange 안 한 칸에 **중첩(포장)**된 것. 타입이 Response→Exchange로 "변한" 게 아님.
- **왜 묶나**: state를 대조하려면 Request(내 원본)와 Response(구글 echo) 둘 다 필요해서.
- **연결**: → 아래 Q(Provider 위임), → 아래 Q(state 대조)

---

### Q. AuthenticationManager가 봉투를 받은 다음 Provider로 어떻게 가나?
- **한줄답**: ProviderManager가 OAuth2LoginAuthenticationProvider에 위임하고, 그게 다시 code용 Provider에 "재포장"해 넘긴다.
- **원리** — 시작 → 과정 → 결과:
    - [시작] `AuthenticationManager`(구현체 `ProviderManager`) = Provider가 아니라 관리자. 결과를 전달만.
    - [과정] `OAuth2LoginAuthenticationProvider`가 봉투를 풀어 `clientRegistration + exchange`를 꺼냄 → **새 타입** `OAuth2AuthorizationCodeAuthenticationToken`으로 재포장
    - [결과] `OAuth2AuthorizationCodeAuthenticationProvider.authenticate(새 토큰)` 호출
- **왜 재포장**: code Provider는 로그인 전용이 아닌 범용 부품이라 자기 입력 타입을 요구 → 어댑터 역할.
- **연결**: → 아래 Q(state 대조+교환)

### Q. state 대조와 code→token 교환은 누가 해?
- **한줄답**: 필터가 아니라 `OAuth2AuthorizationCodeAuthenticationProvider`.
- **원리** — 시작 → 과정 → 결과:
    - [시작] 봉투 안 exchange에서 두 state 꺼냄: `resp.getState()`(URL) vs `req.getState()`(쿠키)
    - [과정] 같으면 통과, 다르면 위조로 차단
    - [과정] `accessTokenResponseClient.getTokenResponse()` = 구글 토큰 엔드포인트에 직접 POST(code+client_id+secret)
    - [결과] 구글이 `access_token`(+id_token) 응답 → 토큰에 담아 반환
- **연결**: → 아래 Q(userinfo), → [[oauth2-state-cookie]]의 state 두 벌

### Q. access token으로 어떻게 사용자를 찾아?
> 그때 왜 궁금했나: access token으로 우리 DB를 뒤지는 줄 알았다.
- **한줄답**: 우리가 찾는 게 아니라, 토큰을 들고 **구글 userinfo에 물어보면 구글이 찾아준다.**
- **원리** — 시작 → 과정 → 결과:
    - [시작] `OAuth2LoginAuthenticationProvider`가 access_token으로 `userService.loadUser()` 호출
    - [과정] `CustomOAuth2UserService`의 `delegate.loadUser` = 구글 userinfo에 `Authorization: Bearer <access_token>`로 GET
    - [결과] access_token = 구글에 내미는 출입증. 구글이 token↔user 매핑을 조회해 `{sub, email, email_verified}` 응답
- **연결**: → 아래 Q(우리 서비스 실행), → 아래 Q(upsert)

### Q. {sub,email} 받은 다음에 우리 서비스가 실행되는 거야?
- **한줄답**: 반대. 우리 서비스가 실행돼서 그 **안에서** {sub,email}을 받아온다.
- **원리** — 시작 → 과정 → 결과:
    - [시작] `OAuth2LoginAuthenticationProvider`가 `this.userService.loadUser(access_token)` 호출 = 방아쇠
    - [과정] 우리 `CustomOAuth2UserService.loadUser` 시작 → 내부 `delegate.loadUser`가 구글 userinfo GET → 값 도착
    - [결과] 값 받는 일 자체가 우리 서비스 안에서 일어남
- **왜 우리 게 실행**: SecurityConfig `.userService(customOAuth2UserService)`가 Provider 필드에 우리 빈을 주입해서.
- **연결**: → 아래 Q(upsert)

### Q. 받은 사용자 정보를 DB에 어떻게 저장해? 중복이면 합쳐?
> 그때 왜 궁금했나: 같은 사람이 구글로 또 로그인하면 계정이 새로 생기나 걱정됐다.
- **한줄답**: 3분기 — ① sub로 기존 구글계정 찾으면 그대로 ② (검증된)이메일로 기존계정 찾으면 합치기 ③ 없으면 신규 저장.
- **원리** — `OAuth2UserUpsertService.upsertGoogleUser` (`@Transactional`):
    - `findByProviderAndProviderId(GOOGLE, sub)` 히트 = 재로그인, 그대로 반환 (sub는 안 변하는 안정 식별자)
    - 미스 + emailVerified → `findByEmail` 히트 = `existing.linkGoogle(sub)`로 기존 계정에 provider/providerId만 채움(통합). save() 없이 JPA dirty checking이 커밋 시 UPDATE
    - 다 미스 = `save(User.ofGoogle(...))` 신규 INSERT
- **왜 emailVerified 체크**: 미검증 이메일로 연동하면 남의 이메일로 가짜 구글계정 만들어 계정 탈취 가능 → 검증된 것만 연동.
- **트랜잭션 경계**: userinfo(외부 HTTP)는 트랜잭션 밖(delegate)에서 끝내고 DB만 @Transactional = 커넥션 풀 고갈 방지. self-invocation 프록시 함정 피하려 upsert를 별도 빈으로 분리.
- **연결**: → 아래 Q(OAuth2User 반환)

---

### Q. OAuth2User 반환된 다음엔? 성공 핸들러는 어떻게 연결돼?
- **한줄답**: Provider가 "인증됨 토큰"으로 포장 → 필터로 되감김 → 부모필터가 SecurityContext 저장 + SuccessHandler 호출.
- **원리** — 시작 → 과정 → 결과:
    - [시작] `OAuth2LoginAuthenticationProvider`가 OAuth2User를 principal로 담은 authenticated 토큰 생성
    - [과정] manager가 그대로 필터에 반환 → `attemptAuthentication`이 `OAuth2AuthenticationToken`으로 변환해 return
    - [결과] 부모 `AbstractAuthenticationProcessingFilter`가 성공으로 판정해 successHandler 호출
- **되감기란**: 자식(필터)의 인증 결과가 부모의 try 블록으로 돌아와 후처리(저장+핸들러)로 이어지는 것. 상세 → [[spring-security-filter]]
- **연결**: → 아래 Q(핸들러 내부)

### Q. 성공 핸들러가 실행된 다음엔? (OAuth 흐름의 끝)
- **한줄답**: email로 우리 JWT 발급 → 쿠키 정리 → 토큰을 URL fragment(#token=)에 실어 프론트로 redirect.
- **원리** — 시작 → 과정 → 결과:
    - [시작] `OAuth2LoginSuccessHandler`가 principal(OAuth2User)에서 email(=loginId) 추출
    - [과정] `jwtProvider.createAccessToken(loginId)` = 로컬 로그인과 동일한 JWT 발급 + removeAuthorizationRequest로 쿠키 정리
    - [결과] `#token=`으로 프론트에 redirect → 프론트가 `window.location.hash`로 토큰 꺼내 저장
- **왜 fragment**: 쿼리스트링(?)은 서버 로그·Referer에 남아 유출 위험, fragment(#)는 서버로 안 감.
- **연결**: → 이후 API는 [[spring-security-auth-flow]]의 JwtFilter 흐름, → [[jwt-signing]]

## ③ 용어 카드 (역참조)
- **Authorization Code 방식**: 구글은 code만 주고, 서버가 뒷채널로 token 교환. 토큰 유출 방지. → Q.토큰 주나
- **ClientRegistration**: application.properties의 구글 설정 뭉치(id/secret/tokenUri/userInfoUri). → Q.G 3객체
- **OAuth2AuthorizationExchange**: Request(쿠키)+Response(URL code/state)를 묶은 봉투. → Q.G 3객체
- **OAuth2LoginAuthenticationToken**: Provider에 넘길 인증 요청서. UsernamePasswordAuthenticationToken과 동급. → Q.G 3객체
- **ProviderManager**: AuthenticationManager 구현체. Provider들의 관리자(Provider 아님). → Q.Provider 위임
- **OAuth2AuthorizationCodeAuthenticationProvider**: state 대조 + code→token 교환 담당. → Q.대조 교환
- **CustomOAuth2UserService**: 구글 userinfo 조회 + DB upsert. delegate가 실제 HTTP. → Q.우리 서비스
- **linkGoogle / dirty checking**: 기존 계정에 provider/providerId만 채움. save 없이 UPDATE 자동. → Q.upsert
- **되감기**: 자식 필터 결과가 부모 try 블록으로 돌아와 후처리로 이어지는 구조. → Q.OAuth2User 반환
- **URL 값 전달 3방식**: path variable(식별) / query string(조건) / fragment(#token 클라전용·서버 안감). → Q.핸들러 끝

## ④ 내가 틀렸던 것 (오개념 로그) ★가치 높음
| 내가 생각했던 것 | 실제 |
|---|---|
| 구글이 로그인 끝나면 토큰을 (bearer로) 준다 | code(교환권)를 URL 쿼리로 줌. 토큰은 서버가 뒷채널로 받아옴 |
| 콜백 응답에 token이 들어있다 | code + state만. token 아님 |
| code용 Provider가 원래 토큰을 통째로 받는다 | 원래 토큰을 풀어 새 타입(CodeAuthToken)으로 재포장해 넘김 |
| Response 타입이 Exchange로 바뀐다 | 안 바뀜. Response가 Exchange 안에 중첩(포장)됨 |
| access token으로 우리 DB를 검색한다 | 구글 userinfo에 내미는 출입증. 찾는 건 구글 |
| {sub,email} 받은 뒤 우리 서비스 실행 | 우리 서비스가 실행돼서 그 안에서 받아옴 |
| AuthenticationManager가 첫 Provider다 | Provider 아님. Provider들의 관리자(ProviderManager) |
| "Bearer 헤더"가 맞는 명칭 | Authorization 헤더 + Bearer 스킴. "Bearer 토큰"은 토큰 지칭으로 맞음 |
