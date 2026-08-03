# Trouble Shooting

> **주제**: 실제로 겪은 에러의 증상·원인·해결 기록 · 갱신: 2026-08-04 · 상태: 진행중
> **태그**: #트러블슈팅 #에러 #디버깅

## 증상 색인

에러 메시지로 찾는다. 어느 프레임워크 문제인지는 원인을 알고 난 뒤에나 알 수 있으므로 색인 기준으로 삼지 않는다.

| 증상 (본 그대로) | 원인 한 줄 | 항목 |
|---|---|---|
| `400: redirect_uri_mismatch` (구글 화면) | 구글 콘솔 등록 URI가 글자 단위로 불일치 | [1](#redirect_uri_mismatch-구글-400-액세스-차단됨) |
| `#error=[invalid_request]` | 콜백 URL을 code/state 없이 직접 호출 | [2](#invalid_request-콜백-url-직접-접속-시) |
| `#error=[authorization_request_not_found]` | state 쿠키 미생성·만료·소비됨 | [3](#authorization_request_not_found) |
| 프론트가 `/login#token=...` 으로 감 | 백엔드는 정상, 프론트 SPA가 재리다이렉트 | [4](#프론트가-oauth2redirect-아닌-login-으로-감-redirect-uri-혼동) |
| `Schema-validation: missing table [users]` | Boot 4 오토컨피그 모듈(`spring-boot-flyway`) 누락 | [5](#flyway-가-안-돎--schema-validation-missing-table-users) |
| `null value in column "password" ... not-null` | `ddl-auto=update` 는 기존 컬럼 제약을 안 바꿈 | [6](#null-value-in-column-password--violates-not-null-constraint) |
| `JWT string cannot be null or empty.` | 형제 예외라 `catch (JwtException)` 이 못 잡음 | [7](#illegalargumentexception-jwt-string-cannot-be-null-or-empty-isvalid-가-못-잡음) |
| 로그인 성공했는데 화면은 실패 / `#error=...` 로 튕김 | redirect-uri 포트(3000)를 Grafana가 점유 | [8](#redirect-uri-포트를-다른-서비스가-점유-로그인-성공이-실패로-보임) |
| `java.io.EOFException` at `ObjectInputStream.<init>` | 지운 쿠키를 같은 요청에서 다시 읽음 | [9](#eofexception--값길이0-예외를-삼키는-catch-때문에-원인-파악-불가) |
| `HV000030` / `HV000271` (제약을 List 에 붙임) | 제약 대상이 컨테이너인지 원소인지는 위치로 정해진다 | [10](#제약을-list-에-붙였다) |
| `TransientPropertyValueException` at `em.flush()` | DB 의 `ON DELETE CASCADE` 를 Hibernate 가 모른다 | [11](#문제를-지우니-flush-에서-터졌다-transientpropertyvalueexception) |

---

## `redirect_uri_mismatch` (구글 400: 액세스 차단됨)

- **발생일**: 2026-07-23
- **맥락**: 구글 소셜 로그인 최초 연동, `/oauth2/authorization/google` 접속 시
- **증상**: 구글 화면에서 "액세스 차단됨: 이 앱의 요청이 잘못되었습니다 / 400 오류: redirect_uri_mismatch"
- **원인**: ==Spring이 구글에 보낸 `redirect_uri` 와 구글 콘솔에 등록된 승인 리디렉션 URI가 글자 단위로 불일치.==
  실제 등록값을 `http://localhost:8080/login/oauth2` 로 넣어 `/code/google` 이 빠져 있었다.
- **진단**: Spring 기본 콜백 경로는 `{baseUrl}/login/oauth2/code/{registrationId}` 형식.
  - `/login/oauth2/code/*` → `OAuth2LoginAuthenticationFilter.DEFAULT_FILTER_PROCESSES_URI` (하드코딩)
  - `{registrationId}` = `application.properties`의 `...registration.google...` 키 → `google`
- **해결**: 구글 콘솔 승인 리디렉션 URI에 `http://localhost:8080/login/oauth2/code/google` 등록
- **개념**: → [[OAuth2]] (콜백 경로가 어떻게 정해지는가)

---

## `[invalid_request]` (콜백 URL 직접 접속 시)

- **발생일**: 2026-07-23
- **맥락**: `/login/oauth2/code/google` 을 브라우저 주소창에 직접 입력
- **증상**: 프론트로 `#error=[invalid_request]` 리다이렉트
- **원인**: ==콜백 URL을 맨몸으로(=`code`/`state` 없이) 직접 호출.== 저장된 authorization request도 매칭 안 됨 → 필터가 잘못된 요청으로 판단
- **진단**: 콜백은 구글이 `?code=...&state=...` 를 붙여 호출하는 주소인데 그게 없었음. 필터가 정상 작동한다는 증거
- **해결**: 콜백이 아니라 **로그인 시작 URL** `/oauth2/authorization/google` 로 접속
- **개념**: → [[OAuth2]] Q.프론트에서 /oauth2/authorization/google로 연결하면 내 서버로 들어와?

---

## `authorization_request_not_found`

- **발생일**: 2026-07-23
- **맥락**: 구글 로그인 완료 후 콜백 처리 단계
- **증상**: 프론트로 `#error=[authorization_request_not_found]` 리다이렉트
- **원인**: ==콜백 시점에 저장해둔 authorization request(state 담긴 쿠키)를 못 찾음 / 역직렬화 실패==
- **STATELESS + 쿠키 저장 메커니즘**: 세션을 안 쓰므로(`SessionCreationPolicy.STATELESS`) state를
  `HttpCookieOAuth2AuthorizationRequestRepository` 로 **쿠키(`oauth2_auth_request`, TTL 180초)** 에 실어나른다.
  콜백 필터가 `removeAuthorizationRequest` → `loadAuthorizationRequest` 로 쿠키를 복원하는데 이게 null이면 이 에러.
- **유력 원인 3가지**:
  1. 시작점(`/oauth2/authorization/google`)을 안 밟고 콜백만 재요청(새로고침/뒤로가기) → 쿠키 미생성
  2. 쿠키 TTL 180초 초과 (구글 화면에 오래 머묾)
  3. 이전 실패로 쿠키가 이미 소비/삭제됨
- **진단 함정**: `deserialize()` 의 `catch → return null` 이 "쿠키 없음"과 "역직렬화 실패"를 구분 못하게 삼킴.
  → 원인 특정이 필요하면 `loadAuthorizationRequest` 에 "쿠키 유무 + 역직렬화 성공 여부" 임시 로그.
  실제로 2026-07-30에 그 임시 로그를 심어 원인을 잡아냈다 — 쿠키를 지우는 코드가 요청 객체를 오염시키고 있었다 → 항목 [9](#eofexception--값길이0-예외를-삼키는-catch-때문에-원인-파악-불가)
- **해결**: 시크릿 창에서 **시작 URL부터** 시작해 **3분 내** 로그인 완료
- **개념**: → [[OAuth2]] (state를 쿠키로 지키는 이유), → [[ServletFilter]] (addCookie가 어떻게 브라우저 쿠키가 되는가)

---

## 프론트가 `/oauth2/redirect` 아닌 `/login` 으로 감 (redirect-uri 혼동)

- **발생일**: 2026-07-23
- **맥락**: 로그인 성공, 프론트 URL이 `http://localhost:3000/login#token=...` 로 뜸
- **증상**: 백엔드 기본값은 `/oauth2/redirect` 인데 실제로는 `/login` 으로 감
- **원인**: 백엔드는 정상적으로 `/oauth2/redirect` 로 302 전송. ==프론트 SPA가 그 주소를 받아 다시 `/login` 으로 client-side 리다이렉트==(라우트 미정의 등). `#token=` fragment는 주소창에 그대로 남아 따라옴
- **진단**: DevTools Network 탭에서 `:8080` 302 응답의 `Location` 헤더 확인 → `.../oauth2/redirect#token=...` 이면 백엔드는 정상
- **해결**: (1) 프론트에 `/oauth2/redirect` 라우트를 만들어 토큰 파싱, 또는 (2) `OAUTH2_REDIRECT_URI` env를 프론트 실제 경로로 맞춤
- **개념**: → [[OAuth2]] Q.성공 핸들러가 실행된 다음엔? (왜 fragment로 토큰을 넘기는가)
- **관련**: 증상은 비슷하나 원인이 다른 사례 → 항목 [8](#redirect-uri-포트를-다른-서비스가-점유-로그인-성공이-실패로-보임). 여기서는 **프론트 SPA가 스스로 재리다이렉트**한 것이고, 항목 8은 **그 포트에 프론트가 아예 없고 다른 서비스가 떠 있던** 경우다

---

## Flyway 가 안 돎 → `Schema validation: missing table [users]`

- **발생일**: 2026-07-23
- **맥락**: Flyway 도입 후 첫 부팅 (Spring Boot 4.1)
- **증상**: 부팅 로그에 Flyway 흔적 0줄, Hibernate가 `Schema-validation: missing table [users]` 로 부팅 실패(앱이 스스로 꺼짐)
- **원인**: ==Spring Boot 4부터 오토컨피그가 기술별 모듈로 분리됨.== `flyway-core`(Redgate 라이브러리)만 넣으면 라이브러리는 있지만 **Boot가 시작 시 `flyway.migrate()` 를 실행해주는 배선(`spring-boot-flyway` 오토컨피그 모듈)이 없음** → 마이그레이션 미실행 → 테이블 없음 → validate 실패
- **진단**: `./gradlew dependencies` 에 `spring-boot-flyway` 없음 확인 / 로그에 flyway 0줄
- **해결**: `implementation 'org.springframework.boot:spring-boot-flyway'` 추가 (flyway-core 를 전이 의존성으로 포함). Postgres는 `flyway-database-postgresql` 별도 필요
- **개념**: → [[DbBasics]] (Flyway 마이그레이션), → [[Persistence]] (`ddl-auto=validate` 가 무엇을 검사하는가)

---

## `null value in column "password" ... violates not-null constraint`

- **발생일**: 2026-07-23
- **맥락**: 구글 로그인 사용자 DB 저장 시 (`User.ofGoogle` → password=null)
- **증상**: `DataIntegrityViolationException` — password NOT NULL 제약 위반
- **원인**: 엔티티는 password 를 nullable로 바꿨지만, ==`ddl-auto=update` 는 기존 컬럼의 제약 변경을 안 함.== 예전 스키마의 `NOT NULL` 이 DB에 그대로 남아 있었음
- **해결**: Flyway 도입 + `V1__init.sql` 에서 password nullable 로 정의, stale 테이블 드롭 후 재생성
- **개념**: → [[Persistence]] (`ddl-auto` 옵션별 한계), → [[DbBasics]] (NOT NULL 제약)

---

## `IllegalArgumentException: JWT string cannot be null or empty.` (isValid 가 못 잡음)

- **발견일**: 2026-07-26 (코드 리뷰로 발견, **미재현**)
- **맥락**: `Authorization: Bearer ` 처럼 접두사만 있고 토큰이 빈 문자열인 요청이 `JwtFilter` 를 통과할 때
- **증상(추정)**: `GlobalExceptionHandler` 가 못 잡아 `BasicErrorController` 의 기본 500 JSON 이 나갈 것으로 보임. 401이어야 할 상황에 500. 실제 재현은 아직 안 했으므로 테스트로 확인 필요
- **원인**: 인자 검증 단계에서 터지는 예외라 `JwtException` 이 아니다. jjwt 내부는 대략 아래와 같이 토큰 형식을 뜯어보기 **전에** 인자 자체를 확인한다.

  ```java
  // DefaultJwtParser (대략)
  if (!Strings.hasText(jwt)) {
      throw new IllegalArgumentException("JWT string cannot be null or empty.");
  }
  ```

  그래서 ==“잘못된 JWT”가 아니라 **“잘못된 메서드 호출”** 로 취급된다.==
- **진단**:
  - API 시그니처가 이미 명시: `Jws<Claims> parseSignedClaims(CharSequence jws) throws JwtException, IllegalArgumentException;`
  - `parseClaims()` 에 try-catch 없음 → 그대로 위로
  - `isValid()` 는 `catch (JwtException e)` 만 있는데, ==두 예외는 부모-자식이 아니라 **형제**(둘 다 `RuntimeException` 직계) → 안 잡힘==
  - unchecked 라 컴파일러 경고도 없음
  - 필터는 DispatcherServlet 앞단이라 `@RestControllerAdvice` 가 개입 불가 → [[ExceptionHandling]]
- **해결(예정)**: `isValid` 에 guard clause + `catch (JwtException | IllegalArgumentException)` 이중 방어. 자세한 원리는 [[JavaBasics]] / [[ExceptionHandling]] 참고
- **남은 확인**: `JwtProviderTest` 가 현재 컴파일되지 않는다(refresh 커밋에서 `JwtProvider` 생성자가 2→3 인자로 바뀌었는데 테스트 4곳이 그대로). 테스트를 복구하면서 `isValid("")` 재현 케이스를 추가해 실제 예외 타입·메시지를 확인하고, 확인되면 이 항목의 `발견일(미재현)` 을 `발생일` + 실제 증상으로 갱신할 것

---

## redirect-uri 포트를 다른 서비스가 점유 (로그인 성공이 실패로 보임)

- **발생일**: 2026-07-30
- **맥락**: 구글 소셜 로그인 전체 흐름을 처음 끝까지 검증하던 중 (Spring Boot 4.1.0 / Spring Security 7.1.0 / Java 21)
- **증상**: 구글 로그인 후 계속 실패로 보임. `http://localhost:3000/login#error=%255Bauthorization_request_not_found%255D` 로 튕김. 브라우저 화면만 보면 완전한 실패
- **원인**: `app.oauth2.redirect-uri` 가 `http://localhost:3000/oauth2/redirect` 인데 ==3000번 포트에 프론트가 아니라 Grafana 컨테이너가 떠 있었다.== Grafana가 모르는 경로를 받고 자기 `/login` 으로 302 리다이렉트 → 토큰이 프론트에 도달하지 못하고 사라짐
- **진단**:
  - `docker ps` → `grafana → 127.0.0.1:3000->3000/tcp` 확인
  - `curl -i http://localhost:3000/oauth2/redirect` → `302 Location: /login`, `Set-Cookie: redirect_to=%2Foauth2%2Fredirect`
  - 이 `redirect_to` 쿠키가 브라우저 쿠키 목록에서 본 것과 **정확히 일치** → Grafana가 범인임이 확정
  - 반대로 서버 로그는 이미 성공이었다: `Authenticated=true`, `Granted Authorities=[ROLE_USER]`, `Redirecting to ...#token=eyJ...`
  - DB 확인 시 `users` 테이블에 행이 정상 생성되어 있었고, 그 토큰으로 `GET /auth/me` 호출하니 200 + 정상 JSON
- **해결**: `OAUTH2_REDIRECT_URI` 를 비어 있는 포트(`http://localhost:5173/oauth2/redirect`)로 변경. 프론트 포트가 확정되면 그 값으로. **3000은 Grafana가 점유 중이라 사용 불가**
  - 빈 포트로 두면 브라우저는 "연결할 수 없음"을 띄우지만 ==주소창에 `#token=` 이 그대로 남아 복사할 수 있다== — 프론트 없이 테스트할 때는 오히려 이쪽이 낫다
- **곁다리로 알게 된 것**: ==`localhost` 쿠키는 포트를 구분하지 않는다.== 개발자도구 Domain 칸에 포트가 안 붙고, 3000(Grafana)·8080(우리 앱)·IntelliJ 쿠키가 전부 같은 `localhost` 저장소를 공유한다. 그래서 우리 서버 로그의 "전체쿠키수"에 남의 쿠키가 섞여 보였다
- **교훈**: ==「실패로 보인다」와 「실패했다」는 다르다.== 브라우저 화면보다 **서버 로그와 DB를 먼저** 확인할 것. 이번엔 인증·회원가입·토큰 발급이 전부 성공한 상태에서 마지막 리다이렉트 목적지만 잘못돼 있었다
- **개념**: → [[OAuth2]] (성공 핸들러가 fragment로 토큰을 넘기는 흐름)

---

## `EOFException` / 값길이=0 (예외를 삼키는 catch 때문에 원인 파악 불가)

- **발생일**: 2026-07-30
- **맥락**: 항목 [3](#authorization_request_not_found)(`authorization_request_not_found`)의 원인을 좁히려고 `loadAuthorizationRequest`/`deserialize` 에 임시 로그를 심은 직후. 관련 파일은 `HttpCookieOAuth2AuthorizationRequestRepository.java`, `OAuth2LoginSuccessHandler.java`, `OAuth2LoginFailureHandler.java`
- **증상**: 쿠키는 존재하는데 **값이 빈 문자열**

  ```
  [OAUTH2] 역직렬화 실패 (값길이=0)
  java.io.EOFException
      at java.io.ObjectInputStream.readStreamHeader(ObjectInputStream.java:985)
      at java.io.ObjectInputStream.<init>(ObjectInputStream.java:416)
      at HttpCookieOAuth2AuthorizationRequestRepository.deserialize(...)
  ```

- **원인 A — `deleteCookie` 가 request 객체를 오염시킴**:

  ```java
  getCookie(request, name).ifPresent(cookie -> {
      cookie.setValue("");   // request 가 들고 있는 실제 Cookie 객체를 수정
      cookie.setMaxAge(0);
      response.addCookie(cookie);
  });
  ```

  ==`request.getCookies()` 는 복사본이 아니라 톰캣이 파싱해 캐시해둔 실제 배열이다.== 여기 담긴 객체를 수정하면 그 요청이 끝날 때까지 이후 모든 `getCookie` 호출이 빈 값을 본다. 응답용 쿠키를 만들려고 **요청 쿠키를 재활용한 것**이 실수
- **원인 B — `removeAuthorizationRequest()` 중복 호출**:
  - `OAuth2LoginAuthenticationFilter.attemptAuthentication()` 이 이미 호출한다 (프레임워크)
  - 그런데 `OAuth2LoginSuccessHandler` / `OAuth2LoginFailureHandler` 에서 또 호출했다
  - 첫 호출: 정상 복원 + `deleteCookie` 로 request 오염 → 두 번째 호출: 빈 값을 읽고 터짐
  - ==`removeAuthorizationRequest` 는 「읽고 지운다」라서 멱등하지 않다.== 이름이 `remove` 라 두 번 불러도 안전할 것 같지만 결과가 다르다
- **왜 하필 EOFException 인가**: `""` → Base64 디코딩 → `new byte[0]`(예외 없음) → `new ObjectInputStream(...)` **생성자가 스트림 헤더 매직넘버 2바이트(`AC ED`)를 읽으려다 즉시 EOF**. `readObject()` 까지 가지도 못하고 객체 생성 시점에 실패한다. 스택트레이스의 `readStreamHeader` / `<init>` 이 그 증거
- **진단 함정 (이 항목의 핵심)**:
  - `loadAuthorizationRequest` 가 null 을 반환하는 경로가 2개인데(① 쿠키 자체가 없음 ② 쿠키는 있는데 역직렬화 실패) `catch (Exception e) { return null; }` 로 예외를 삼켜 **둘을 구분할 수단이 없었다.** 둘 다 `authorization_request_not_found` 로 귀결되지만 원인도 해결책도 완전히 다르다
  - 더 나쁜 건, ==`값길이=0` 로그는 **두 번째** 호출이 찍은 것이고 첫 번째 호출은 이미 `복원성공 state=...` 이었다.== 인증 흐름 자체는 멀쩡했는데 이 노이즈를 원인으로 착각해 엉뚱한 곳을 오래 팠다
- **해결**:
  1. 핸들러의 중복 `removeAuthorizationRequest()` 호출 제거 (프레임워크가 이미 함) **(적용됨)**
  2. `deleteCookie` 가 새 `ResponseCookie` 를 만들어 응답에만 싣도록 변경 → `request` 파라미터 자체가 불필요해짐. 없는 쿠키를 지우라고 해도 무해하므로 존재 확인도 필요 없음 **(미적용)**
  3. `deserialize` 에 빈 값 가드(`StringUtils.hasText`) + `catch` 에 `log.warn(..., e)` **(미적용)**
- **교훈**:
  - ==예외를 삼키는 `catch` 는 디버깅을 불가능하게 만든다.== 최소한 `log.warn(..., e)` 한 줄
  - `request` 에서 꺼낸 객체를 수정하지 마라 — 요청 상태가 오염돼 같은 요청 내 다른 코드가 잘못된 값을 본다
  - 프레임워크가 이미 호출하는 **멱등하지 않은** 훅을 중복 호출하지 마라
  - 가장 시끄러운 예외가 진짜 범인이 아닐 수 있다. 시간순 **첫 번째** 이상 징후부터 볼 것
- **개념**: → [[OAuth2]] (state 쿠키 저장소 동작), → [[ServletFilter]] (request/response 쿠키 다루기), → [[JavaBasics]] (Java 직렬화와 ObjectInputStream)

---

## 제약을 List 에 붙였다 

[동기] `POST /problems` 요청 DTO 에 태그 key 목록과 예제 목록을 받아야 했다. 둘 다 `List` 인데 비어 있으면 안 되고 원소도 검증돼야 했다.

[시도] 평소 문자열에 쓰던 대로 `@NotBlank` 를 리스트에 붙이고, 원소까지 타고 들어가라고 `@Valid` 를 함께 달았다.

[문제] Swagger 예시에 `"tagKeys": [0]` 이라는 이상한 값이 찍혔고, 요청을 보내면 400 이 아니라 500 이 났다.

```
HV000030: No validator could be found for constraint '@NotBlank'
validating type 'java.util.List<java.lang.String>'
```

[원인] `@NotBlank` 를 처리하는 validator 는 `NotBlankValidator implements ConstraintValidator<NotBlank, CharSequence>` 하나뿐이라 `List` 에 맞는 후보가 없다. (`@NotEmpty` 는 CharSequence·Collection·Map·배열용으로 12개가 들어 있다.) ==`@Target` 은 "어디에(필드·파라미터) 붙일 수 있나"만 제한할 뿐 "어떤 타입에 붙일 수 있나"는 표현할 방법이 없다.== 그래서 컴파일은 통과하고 첫 검증 시점에 `UnexpectedTypeException` 으로 터진다 — 검증 실패(400)가 아니라 설정 오류(500)다.

[시도] 컬렉션의 "비어 있지 않음" 은 `@NotEmpty` 소관이므로 그걸로 바꿨다.

[문제] 500 은 사라졌는데 이번엔 경고가 떴다.

```
HV000271: Using `@Valid` on a container (java.util.List) is deprecated.
You should apply the annotation on the type argument(s).
Affected element: PostProblemRequest#samples()
```

[원인] ==`@Valid List<SampleRequest>` 는 문자 그대로 읽으면 "List **객체 자체**를 검증하라" 다.== 원소로 내려가라는 뜻이 아니다. 옛날 Hibernate Validator 가 관례적으로 원소까지 타고 들어가 줬을 뿐이고, 애노테이션의 **위치와 실제 의미가 어긋난** 상태였다. Bean Validation 2.0 에서 컨테이너 원소 제약이 생기며 이걸 정확히 쓸 문법이 마련됐고(`@Valid` 의 `@Target` 에 `TYPE_USE` 추가), Hibernate Validator 9 가 옛 방식을 deprecated 로 표시했다. 지금은 동작하지만 다음 메이저에서 빠지면 **사라지는 게 기능이 아니라 검증이라 조용히 뚫린다.**

[해결] 애노테이션을 꺾쇠 안으로 옮겼다. "리스트 자체"가 대상인 것과 "원소 각각"이 대상인 것을 위치로 갈랐다.

```java
// @NotEmpty 는 밖 = 리스트 자체가 비면 안 된다
// @NotBlank / @Valid 는 꺾쇠 안 = 원소 각각을 검증한다
@NotEmpty List<@NotBlank String> tagKeys,
@NotEmpty List<@Valid SampleRequest> samples
```

→ [[JavaBasics]] (애노테이션 `@Target` 과 `TYPE_USE`)

---

## 문제를 지우니 flush 에서 터졌다 (`TransientPropertyValueException`)

[동기] `DELETE /problems/{problemNum}` 이 FK 의 `ON DELETE CASCADE` 에 기대어 자식 행까지 정말 지우는지 확인하고 싶었다. 그때까지는 FK 정의만 보고 "걸려 있으니 될 것"이라 믿고 있었다.

[시도] `@SpringBootTest @Transactional` 통합 테스트를 짰다. `postProblem` 으로 문제 하나를 만들고, 자식 행 개수를 센 뒤, `deleteEachProblem` 을 부르고 다시 세는 흐름이다.

[문제] 삭제 직후 `em.flush()` 에서 터졌다.

```
TransientPropertyValueException: Persistent instance of
'com.gachon.judge.domain.entity.TestCase' references an unsaved transient
instance of 'com.gachon.judge.domain.entity.Problem'
(persist the transient instance before flushing)
```

[원인] ==`ON DELETE CASCADE` 는 DB 의 규칙이라 Hibernate 가 모른다.== 같은 트랜잭션 안에서 `postProblem` 이 만든 `TestCase` · `ProblemTag` 가 영속성 컨텍스트에 그대로 남아 `Problem` 을 참조하고 있었는데, 그 `Problem` 을 `em.remove` 하자 Hibernate 는 **"살아 있는 자식이 사라질 부모를 가리킨다"** 로 판단했다. DB 는 곧 자식을 치울 예정이지만 flush 시점에는 DELETE 가 나가기도 전이라, Hibernate 가 먼저 제동을 건 것이다.

[해결] 삭제 전에 영속성 컨텍스트를 비웠다(`em.clear()`). 자식 엔티티가 준영속으로 떨어지면 Hibernate 의 추적 대상에서 빠지므로, flush 할 때 `Problem` 하나만 지우면 되고 나머지는 DB 가 처리한다. `flush()` 를 먼저 두는 이유는 `clear()` 가 아직 안 내보낸 변경을 그냥 버리기 때문이다.

```java
em.flush();
em.clear();   // postProblem 이 남긴 TestCase / ProblemTag 를 세션에서 떼어낸다

problemService.deleteEachProblem(99001);
em.flush();
em.clear();
```

==운영 경로에는 이 문제가 없다.== `DELETE` 요청은 자기 트랜잭션에서 혼자 돌고 자식을 로딩한 적이 없어 세션이 비어 있다. 같은 트랜잭션에서 등록과 삭제를 연달아 한 **테스트 특유의 상황**이었다. 다만 "문제를 수정하면서 예제도 손보는" API 를 만들면 같은 함정을 만난다.

→ [[Persistence]] (cascade 를 DB 와 JPA 중 어느 층에 둘 것인가)
