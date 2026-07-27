# Trouble Shooting

> **주제**: 실제로 겪은 에러의 증상·원인·해결 기록 · 갱신: 2026-07-28 · 상태: 진행중
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
  → 원인 특정이 필요하면 `loadAuthorizationRequest` 에 "쿠키 유무 + 역직렬화 성공 여부" 임시 로그
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
