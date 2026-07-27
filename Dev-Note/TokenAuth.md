# 토큰 기반 인증 (JWT · Refresh)

> **주제**: 토큰 기반 인증 — JWT 서명·검증, access/refresh 전략, 쿠키 전달 · 갱신: 2026-07-26 · 상태: 진행중
> **태그**: #토큰인증 #JWT #refresh #쿠키 #보안

## ① 큰 그림 (지도)

토큰 기반 인증 = 서버가 세션 상태를 안 들고(stateless), **서명된 토큰**으로 신원을 증명. 두 토큰을 역할·전달·수명으로 나눈다.

| | access | refresh |
|---|---|---|
| 역할 | 매 API 요청 인증 | access 재발급용 마스터키 |
| 전달 | 응답 body → `Authorization: Bearer` 헤더 | httpOnly 쿠키(브라우저 자동 전송) |
| 수명 | 짧음(예: 30분) | 김(예: 14일) |
| 왜 그 자리 | JS가 헤더에 붙여야 함 + 짧아서 노출 감수 | JS가 못 읽게(XSS 방어) — 치명적이라 격리 |

```
[JWT 서명]
secret 문자열 --getBytes(UTF-8)--> 바이트 키 --hmacShaKeyFor--> SecretKey (준비만)
Jwts.builder().claim(...).signWith(SecretKey).compact()  ← 여기서 실제 HMAC-SHA256 서명
토큰 = header . payload . signature
                 (Base64,읽힘)  (256비트 지문)

[검증]
같은 SecretKey로 header+payload를 재해싱 → 토큰의 signature와 대조 (같으면 유효)
```

- **대칭키**: 서명과 검증에 같은 secretKey. 그래서 secretKey는 서버 안에만(유출되면 위조 토큰 발급 가능).
- 비유: HMAC-SHA256 = 비밀 레시피(키)를 섞어 토큰 내용의 지문을 찍는 오븐. SHA는 단방향(복호화 불가).
- **토큰 vs 세션**은 인증 전략 축, **쿠키 vs 헤더**는 전송 축 — 서로 직교(refresh는 JWT인데 쿠키로 전송). → [[OAuth2]]

**refresh token 구현 지도** — refresh token 구현은 파일/기능 목록이 아니라 ==토큰 한 개의 생애주기==다.

```
[refresh token 생애주기 = 구현 4단계]

  ① 발급                ② 저장                 ③ 재발급              ④ 폐기
JwtProvider          RefreshTokenStore      AuthService.reissue   AuthService.logout
.createRefreshToken   (Redis, TTL=만료)      + /auth/refresh       + /auth/logout
      │                     │                      │                    │
   토큰 생성  ────────>  서버가 기억  ────────>  대조 후 새로 발급  ──>  기억에서 삭제
                        (무효화용)              (+ 이전 것 폐기)
```

- **부속품**(뼈대에 딸려오는 것): `TokenPair`(두 토큰 동시 반환) / `CookieUtil`(httpOnly 쿠키 전달) / `ErrorCode.INVALID_REFRESH_TOKEN`(401)
- **잊기 쉬운 곳**: `build.gradle`(spring-boot-starter-data-redis) / `application.yaml`(refresh-expiration·redis host·쿠키 secure/sameSite) / `OAuth2LoginSuccessHandler`(소셜 로그인에도 refresh 발급) → [[OAuth2]]

## ② 질문 트리 (본문)

### 2026-07-20

#### Q. Keys.hmacShaKeyFor(secret.getBytes(UTF_8))는 어떻게 동작하나? hmacShaKeyFor가 뭐야?
- **한줄답**: ==문자열 시크릿을 HMAC-SHA용 SecretKey 객체로 "준비"하는 것==. 여기서 해싱/서명은 안 한다.
- **원리**: `getBytes`로 바이트화 → `hmacShaKeyFor`가 SecretKey 생성(+길이 검증 +알고리즘 선택). "For" = HMAC-SHA에 쓸 키를 만들라는 뜻.
- **연결**: → 아래 Q(서명은 signWith)

#### Q. getBytes(UTF-8)는 인코딩인가? 32바이트가 256비트가 되는 거야?
- **한줄답**: 맞다. `getBytes(UTF-8)`는 문자→바이트 인코딩. ==32바이트=256비트는 인코딩이 아니라 단위 환산(1byte=8bit)==.
- **원리**: 암호 연산은 바이트 단위 → UTF-8로 문자를 바이트로 표현(ASCII 1글자=1바이트). 그 바이트가 곧 HMAC의 비밀키. "바이트→비트 인코딩" 같은 건 없고, 32바이트를 비트로 세면 256비트일 뿐.
- **연결**: → 아래 Q(예시 키)

#### Q. Q}a1L80b/... 같은 키가 어떻게 256비트가 되는 거야?
- **한줄답**: 그 키는 43글자=43바이트=344비트. ==256비트가 "되는" 게 아니라 256비트 최소치를 넘긴 것==.
- **원리**: 256비트는 고정 크기가 아니라 HS256의 **하한**(32바이트). 이상이면 다 유효. 주의: 글자수=바이트수는 ASCII만 성립(한글 3바이트, 이모지 4바이트).
- **연결**: → 아래 Q(내 코드 HS256?)

#### Q. 같은 secretKey로 서명하고 검증도 이걸로? 그래서 대칭키야?
- **한줄답**: 맞다. ==서명·검증에 같은 키 → 대칭키==. 검증은 "같은 키로 서명을 다시 계산해 대조".
- **원리**: 검증기가 header+payload를 secretKey로 재해싱해 토큰의 signature와 비교. 같으면 유효(위조 없음), 다르면 예외. 키를 모르면 유효한 서명을 못 만들어 위조 불가. 대칭키라 secretKey는 절대 외부 노출 금지.
- **연결**: → 아래 Q(HMAC vs SHA)

#### Q. hmac-sha는 sha와 다른 거네? sha256이 뭘 하는 거고 hmac-sha256은 뭐지?
- **한줄답**: ==SHA-256은 단방향 해시(임의 입력 → 256비트 지문). HMAC-SHA256은 거기에 비밀키를 섞은 버전==.
- **원리**: SHA는 암호화(복호화 가능)가 아니라 되돌릴 수 없는 해싱. 그냥 SHA는 누구나 계산 가능(키 없음) → 위조 가능. HMAC은 키를 섞어(SHA를 키와 함께 2번) "키 아는 사람만 만들 수 있는 해시"로 만든 것.
- **연결**: → 아래 Q(무엇이 해싱되나)

#### Q. SHA256이 secretKey를 256비트로 해싱한다는 거야?
- **한줄답**: 아니다. ==해싱 대상은 secretKey가 아니라 "토큰 내용(header+payload)"==. secretKey는 거기 섞는 재료.
- **원리**: SHA-256 입력은 1개(메시지)지만 HMAC은 입력 2개(키+메시지). 키를 256비트로 바꾸는 게 아니라, 키로 메시지를 해싱한 결과(서명)가 256비트. 256이 되는 건 서명이지 키가 아니다.
- **연결**: → 아래 Q(해싱되는 내용)

#### Q. 256비트로 해싱되는 내용은 뭐야? header+sign을 합쳐서 256? payload는 그대로 둔다는 게 무슨 말?
- **한줄답**: ==해싱 입력 = header+payload, 결과 = signature(256비트)==. payload 원본은 그대로 남고 해싱은 별도 지문을 만들 뿐(원본 소비 안 함).
- **원리**: 토큰은 `header.payload.signature` 3조각. signature = HMAC-SHA256(secretKey, header+payload) → 세 번째 조각. payload는 Base64로 읽히게 그대로 저장되고, 그걸 해싱한 256비트 지문이 옆에 붙는다(복사기처럼 원본 유지). 토큰 전체가 256비트인 게 아니라 signature만 256비트. header를 줄여 256 맞추는 일도 없다.
- **주의**: payload는 Base64일 뿐 암호화가 아니라 ==누구나 디코드해 읽을 수 있다== → claim에 민감정보를 넣지 않는다.

#### Q. hmacShaKeyFor에서 이미 hmac-sha를 한 거 아냐? 정의만 한 거야?
- **한줄답**: 정의(준비)만. ==실제 HMAC-SHA 서명은 `signWith(secretKey)` + `compact()`에서 일어난다==.
- **원리**: 생성자의 hmacShaKeyFor는 열쇠를 깎아 필드에 저장(앱 시작 시 1회). 토큰 만들 때마다 `signWith`가 header+payload를 그 열쇠로 실제 해싱해 signature 생성. 검증 시엔 `verifyWith(secretKey)`가 재해싱.
- **연결**: → 위 Q(서명 대상)

#### Q. 그러면 지금 내 코드는 HMAC-SHA256인 거야?
- **한줄답**: 맞다. ==`hmacShaKeyFor`+`signWith` 조합은 HMAC-SHA. 키가 32~47바이트면 HS256==.
- **원리**: jjwt가 키 길이로 알고리즘 자동 선택(32~47B→HS256, 48~63B→HS384, 64B~→HS512). 확인은 토큰 header 디코드 → `{"alg":"HS256"}`. 키 32바이트 미만이면 WeakKeyException.
- **연결**: → 위 Q(예시 키 344비트)

---

### 2026-07-21

#### Q. JwtFilter는 어떻게 작성하나?
- **한줄답**: ==`extends OncePerRequestFilter` → `doFilterInternal` 오버라이드: 헤더 토큰 추출 → 검증 → SecurityContext 등록 → `filterChain.doFilter()`==.
- **원리**: 헤더가 `Bearer ` 로 시작 안 하면 그냥 통과. 유효하면 `JwtProvider.isValid/getLoginId` 로 검증(secretKey는 JwtProvider 내부 캡슐화)하고 authenticated 토큰을 SecurityContext에 저장. ==마지막 `doFilter` 를 빼먹으면 요청이 멈춘다.==
- **왜 OncePerRequestFilter 인가**: → [[FilterChain]] Q.그럼 왜 JWT 필터는 OncePerRequestFilter를 쓰나
- **연결**: → [[Authentication]] Q.authenticated 토큰은 어디서 만들고 왜 3개 인자인가, → 위 Q(서명 검증)

---

### 2026-07-25

#### Q. refreshToken도 쿠키 말고 응답 body에 담아 주면 되는 거 아냐? 클라이언트가 값을 몰라야 해서 쿠키야?
- **한줄답**: body로 줘도 기능은 되지만, ==refresh는 JS가 못 읽는 httpOnly 쿠키에 격리==한다. "클라가 값을 몰라야"가 아니라 ==클라의 JavaScript가 못 읽어야(XSS 방어)==가 정확.
- **원리**: body로 주면 프론트 JS가 어딘가(localStorage/변수/일반 쿠키)에 저장해야 하고 전부 JS 접근 가능 → XSS 스크립트가 훔쳐감. refresh는 14일짜리 마스터키(계속 access를 찍음)라 털리면 계정 탈취. httpOnly 쿠키는 브라우저는 값을 갖고 `/auth`로 자동 전송하지만 `document.cookie`로 JS가 못 읽음 → XSS로도 유출 불가. access는 30분이라 body 노출을 감수(헤더에 붙이려면 JS가 들고 있어야 함). 보너스: 쿠키는 자동 전송이라 프론트가 저장·첨부를 안 해도 됨.
- **연결**: → [[OAuth2]] (httpOnly 쿠키·Set-Cookie 메커니즘), 토큰/세션·쿠키/헤더 축 구분 → 위 ① 큰 그림

---

### 2026-07-26

#### Q. refreshToken을 만들려면 ErrorCode, ErrorResponse, TokenPair, RefreshTokenStore, AuthService의 reissue, JwtProvider의 createRefreshToken, CookieUtil 이런 게 필요한데 맞아? 추가로 필요한 게 있나?
- **한줄답**: 7개 다 맞고 이미 구현돼 있다. ==다만 그 목록이 완전하지 않다== — 실제 커밋(f705321)은 14개 파일을 건드렸다.
- **원리** — 시작 → 과정 → 결과:
    - [시작] 목록의 7개를 실제 커밋과 대조해 본다
    - [과정] 7개는 전부 존재 확인 — 기능 자체는 이미 동작하는 상태
    - [과정] 목록에 없는데 실제로 필요했던 것: `AuthController`의 `/auth/refresh`·`/auth/logout` 엔드포인트(서비스 메서드만 있으면 호출할 방법이 없다), `AuthService.logout`(서버에 저장했으면 지우는 로직이 반드시 짝으로 필요), `build.gradle`의 Redis 의존성, `application.yaml`의 refresh-expiration·redis·쿠키 설정
    - [결과] 특히 `OAuth2LoginSuccessHandler` — ==소셜 로그인에도 refresh를 발급해야 한다==. 빠뜨리면 소셜 로그인 유저만 access 만료 시 재로그인해야 하는 반쪽 상태가 된다
- **연결**: → [[OAuth2]] (소셜 로그인 성공 핸들러) / → 위 Q.refreshToken도 쿠키 말고 응답 body에 (2026-07-25)

#### Q. 그러면 refresh token을 추가하려면 위의 7개 파일·메서드를 모두 해야 한다는 거야?
- **한줄답**: 아니다. ==7개는 성격이 3가지로 갈린다== — 진짜 필수 / 설계 선택 / refresh와 무관.
- **원리**: 진짜 필수는 `createRefreshToken`(토큰을 만들어야 하니까)과 `reissue`(재발급이 refresh token의 존재 이유)뿐이고, 여기에 목록에 빠져 있던 `/auth/refresh` 엔드포인트가 더해진다. `RefreshTokenStore`·`CookieUtil`·`TokenPair`는 **설계 선택**이다 — ==RefreshTokenStore는 재발급이 아니라 무효화를 위한 것==이라, 서버 저장 없이 서명만 검증하는 stateless 방식도 동작하고, refresh를 응답 body에 담아도 동작한다(대신 XSS에 취약). `ErrorCode`는 enum 상수 한 줄 추가라 "파일 작업"이라 부르기 과하고, `ErrorResponse`는 아예 무관하다.
- **왜 이렇게**: `ErrorResponse`·`CustomException`·`GlobalExceptionHandler`가 같은 커밋에 들어간 건 그때 전역 예외 처리를 통째로 세팅했기 때문이지 refresh token 때문이 아니다. ==같은 커밋에 있다고 같은 기능이 아니다==.
- **연결**: → [[ExceptionHandling]] (전역 예외 처리), → [[FilterChain]] (필터 계층)

#### Q. 내가 외우려고 그래 — 어떤 게 필요한지
- **한줄답**: ==발급 → 저장 → 재발급 → 폐기==. 이 4단계만 외우면 다른 프로젝트에도 그대로 적용된다.
- **원리** — 시작 → 과정 → 결과:
    - [시작] **발급** — `JwtProvider.createRefreshToken`. `createAccessToken`과 코드가 거의 같아서 `createToken(loginId, type, expMs)` 하나로 합칠 수도 있다
    - [과정] **저장** — `RefreshTokenStore`(Redis, TTL = 토큰 만료 시각). 키는 `refresh:{loginId}`
    - [과정] **재발급** — `AuthService.reissue` + `AuthController`의 `/auth/refresh`. 서비스 메서드와 엔드포인트는 항상 한 쌍
    - [과정] **폐기** — `AuthService.logout` + `/auth/logout`. ==저장했으면 지우는 짝이 반드시 있어야 한다==
    - [결과] 부속품 3개는 뼈대에 딸려온다: `TokenPair`(access+refresh를 한 번에 반환), `CookieUtil`(httpOnly 쿠키로 내보냄), `ErrorCode.INVALID_REFRESH_TOKEN`(재발급 실패 시 401)
- **주의**: 자바 파일 밖에서 잊기 쉬운 3곳 — `build.gradle`(spring-boot-starter-data-redis), `application.yaml`(refresh-expiration·redis host/port·쿠키 secure/sameSite), `OAuth2LoginSuccessHandler`(소셜 로그인 refresh 발급)
- **연결**: → 위 ① 큰 그림의 refresh token 생애주기 다이어그램

## ③ 용어 카드 (역참조)

> [!quote]- 용어 11개
> - **HMAC-SHA256**: 비밀키를 섞어 header+payload의 256비트 서명을 만드는 대칭키 방식. → Q.HMAC vs SHA
> - **SHA-256**: 단방향 해시(임의 입력→256비트). 암호화 아님(복호화 불가). → Q.sha256이 뭘
> - **hmacShaKeyFor**: 바이트 키로 SecretKey를 준비하는 유틸(해싱 아님, 길이 검증+알고리즘 선택). → Q.동작
> - **signWith / verifyWith**: 실제 서명 / 검증(재해싱)이 일어나는 지점. → Q.정의만?
> - **signature**: header+payload를 secretKey로 해싱한 256비트 지문(토큰 3번째 조각). → Q.해싱되는 내용
> - **대칭키**: 서명·검증에 같은 키. secretKey 유출 시 위조 토큰 발급 가능. → Q.대칭키
> - **access / refresh 토큰**: access=매 요청 인증(짧음, body/헤더) / refresh=access 재발급 마스터키(김, httpOnly 쿠키). → Q.refresh 쿠키
> - **httpOnly 쿠키**: 브라우저는 값을 갖고 자동 전송하되 JS(`document.cookie`)는 못 읽음 → XSS로 탈취 불가. → Q.refresh 쿠키
> - **발급→저장→재발급→폐기**: refresh token 구현의 4단계 뼈대. 기능 목록이 아니라 토큰 한 개의 생애주기로 외운다. → Q.내가 외우려고 그래
> - **revocation(무효화)**: 발급된 토큰을 만료 전에 강제로 못 쓰게 만드는 것. `RefreshTokenStore`가 존재하는 진짜 이유. → Q.7개를 모두 해야 하나
> - **stateless refresh**: 서버 저장 없이 서명 검증만으로 재발급하는 방식. 동작은 하지만 로그아웃 즉시 반영과 재사용 탐지가 불가능하다. → Q.7개를 모두 해야 하나

## ④ 내가 틀렸던 것 (오개념 로그)

> [!quote]- 오개념 12건
> | 내가 생각했던 것 | 실제 |
> |---|---|
> | SHA는 암호화다 | 단방향 해싱이다(복호화 불가) |
> | SHA가 secretKey를 256비트로 바꾼다 | 해싱 대상은 header+payload, 256은 서명(키는 재료) |
> | header+signature를 합쳐서 256 | header+payload를 해싱한 결과가 signature(256), sign은 결과지 입력 아님 |
> | payload가 해싱되면 사라진다 | 원본은 그대로 남음(복사기), 256비트 서명은 별도로 붙음 |
> | 키가 정확히 256비트여야 한다 | 최소 256비트(그 이상 OK, 예시는 344비트) |
> | hmacShaKeyFor가 서명을 실행한다 | 준비만. 실제 서명은 signWith |
> | 글자 수 = 바이트 수 | ASCII만 성립. 한글 3바이트, 이모지 4바이트 |
> | refresh를 body로 줘도 됨 / 클라가 값을 몰라야 해서 쿠키 | 브라우저는 값을 갖고 자동 전송, 핵심은 JS가 못 읽게(XSS 방어) |
> | `RefreshTokenStore`가 없으면 refresh token이 동작하지 않는다 | stateless로도 재발급은 된다. 저장은 **무효화(로그아웃 즉시 반영·재사용 탐지)** 를 위한 것이지 재발급을 위한 게 아니다 |
> | `ErrorResponse`도 refresh token 때문에 필요하다 | 무관하다. 공통 에러 응답 DTO가 같은 커밋에 우연히 함께 들어갔을 뿐이다 |
> | `CookieUtil`은 refresh token의 필수 요소다 | 응답 body에 담아도 동작한다. **httpOnly 쿠키로 보내기로 한 설계 선택**의 결과물이다 |
> | 7개 파일 목록이면 refresh token 구현이 끝난다 | 실제 커밋은 14개 파일을 건드렸다. 엔드포인트·logout·Redis 의존성·yaml 설정·소셜 로그인 refresh 발급이 빠져 있었다 |
