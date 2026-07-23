# Spring Security 로그인·인증 흐름

> **주제**: Spring Security 로그인·인증 흐름 · 갱신: 2026-07-20 · 상태: 진행중
> **태그**: #spring-security #인증 #로그인

## ① 큰 그림 (지도)

```
[로그인] POST /auth/login
  → AuthController → AuthService.login()
      → authenticationManager.authenticate( unauthenticated(id, pw) )
          → ProviderManager
             → DaoAuthenticationProvider
                → UserDetailsService.loadUserByUsername(id)   (DB 조회만)
                → PasswordEncoder.matches(입력pw, DB의암호화pw) (비교)
      → 성공: authenticated Authentication 반환
      → JwtProvider.createAccessToken( authentication.getName() )
  → TokenResponse
```

- **토큰 상태 변화**: `unauthenticated`(입력, 검증 전) → `authenticate()` → `authenticated`(결과, 검증 후).
- **로그인 방식 3가지**: A=컨트롤러+AuthManager 직접 호출(채택) / B=필터 상속(세션용 정석, JWT엔 번거로움) / C=서비스 수동 검증(AuthManager 안 씀).
- 비유: 로그인=경비원이 신분증(id/pw) 확인 후 방문증 발급. 방문증(authenticated 토큰)을 SecurityContext에 걸어야 인정.

## ② 질문 트리 (본문)

### 2026-07-20

#### Q. 클라이언트 POST /login부터 SecurityContext 저장까지 9단계 흐름이 맞아?
- **한줄답**: 거의 맞고, 딱 한 군데 틀림 — `loadUserByUsername()`은 비밀번호를 검증하지 않는다.
- **원리**: `loadUserByUsername`은 id로 **DB에서 사용자 조회만** 한다(파라미터에 pw도 없음). 비번 대조는 `DaoAuthenticationProvider`가 `PasswordEncoder.matches()`로 별도 수행. 성공 시엔 새 authenticated 토큰이 생성됨(기존 토큰 필드 변경 아님).
- **연결**: → 아래 Q(authenticate 내부), → [[spring-security-filter]]

#### Q. AuthenticationManager를 "직접 호출/직접 만든다"는 게 무슨 뜻?
- **한줄답**: 필터 대신 컨트롤러/서비스에서 `authenticate()`를 직접 부르는 것(=B 안 씀). "만든다"는 새로 짜는 게 아니라 **이미 있는 걸 빈으로 꺼내는 것**.
- **원리**: `UsernamePasswordAuthenticationFilter`도 내부에서 결국 `authenticate()`를 부른다. 방식 A는 그 호출을 컨트롤러에 드러낼 뿐. AuthManager는 Security가 내부에 이미 만들지만 필터 체인에 숨어있어, 주입하려면 `AuthenticationConfiguration.getAuthenticationManager()`로 노출해야 함.
- **연결**: → 아래 Q(왜 빈 등록)

#### Q. authenticate()는 DB의 사용자를 어떻게 확인하나? DB에서 직접 가져와?
- **한줄답**: 내가 만든 `UserDetailsService`가 DB에서 꺼내고(조회), Spring이 `PasswordEncoder`로 비교한다.
- **원리**: Spring은 내 DB 구조를 모른다 → `loadUserByUsername`(내 코드)이 조회 담당. 비번은 `WHERE password=?`가 아니라, id로 꺼낸 뒤 `passwordEncoder.matches(평문, 암호화값)`로 대조(BCrypt라 직접 비교 불가).
- **연결**: → [[jwt-signing]] (createAccessToken은 성공 후)

#### Q. UserDetailsService는 Spring Security 안 쓰면 없어도 됐던 거네?
- **한줄답**: 맞다. AuthManager에 위임하니 필요한 것. 직접 검증(방식 C)하면 UserRepository+PasswordEncoder만으로 되고 UserDetailsService 불필요.
- **원리**: `UserDetailsService`는 Security 인증 엔진의 규격. A/B는 AuthManager를 쓰므로 필수, C는 서비스에서 `findByLoginId` + `matches`를 직접 해서 생략 가능.
- **연결**: → 아래 Q(10년차 선택)

#### Q. 10년차라면 로그인을 어떻게? 로그인은 signup이랑 다른 service야?
- **한줄답**: 방식 A(컨트롤러+AuthManager) 추천. signup은 평범한 CRUD 서비스, login은 인증 엔진을 태우는 별개 흐름.
- **원리**: 필터(B)는 세션 폼로그인용이라 JWT엔 JSON 파싱/토큰 삽입 등 우회가 많다. 로그인도 하나의 API라 컨트롤러가 테스트/문서화에 유리. signup=`userRepository.save()`(Security 무관), login=AuthManager 위임.
- **연결**: → [[spring-security-filter]]의 필터 방식

#### Q. `return ResponseEntity.ok(authService.login(req))`면 service로 안 넘어가는 거 아냐?
- **한줄답**: 넘어간다. 안쪽 `authService.login(req)`이 먼저 실행되고 그 결과를 감싼다.
- **원리**: 자바는 인자를 먼저 평가 → `login()` 실행 후 `ResponseEntity.ok(결과)`. 단, `authService`가 주입되려면 생성자(또는 `@RequiredArgsConstructor`)가 있어야 함(final 필드).
- **연결**: → 없음(자바 평가순서)

#### Q. unauthenticated는 왜 쓰고, 왜 id/pw 2개 인자를 넣나?
- **한줄답**: 로그인 입력은 "아직 검증 안 된" 상태라서. 인자는 `(principal, credentials)` = (loginId, password).
- **원리**: Spring Security는 인증을 "누구(principal)+증명(credentials)"로 모델링. `authenticate()`는 unauthenticated 입력을 받아 authenticated를 반환하는 함수. `.unauthenticated()`는 "검증 전" 의도를 이름에 드러낸 팩토리(2-arg 생성자와 동일).
- **연결**: → 아래 Q(authenticated)

#### Q. authenticated 토큰은 어디서 만들고, 왜 3개 인자여야 하나?
- **한줄답**: JwtFilter가 검증 후 직접 만들고(그리고 authenticate() 반환값도 그것). authenticated는 권한(authorities)이 필수라 **3-arg 생성자로만** 생성.
- **원리**: `setAuthenticated(true)`를 외부에서 부르면 예외(오버라이드로 막음) → 권한을 넘기는 3-arg 생성자 내부에서 `super.setAuthenticated(true)`로 우회. 검증 없이 "인증됨"으로 위조하는 권한 상승을 차단하는 설계. 대안: `.authenticated(principal, creds, authorities)` 팩토리.
- **연결**: → [[spring-security-filter]]의 JwtFilter, → 아래 Q(SecurityContext)

#### Q. authentication.getName()이 어떻게 바로 loginId를 뽑아?
- **한줄답**: 마법 아님. `getName()` 내부에 "principal이 UserDetails면 `getUsername()`을 꺼낸다"는 코드가 있다.
- **원리**: 검증 후 principal은 문자열 id에서 `UserDetails` 객체로 승격됨. `AbstractAuthenticationToken.getName()`이 타입을 보고 UserDetails면 `getUsername()`, 문자열이면 `toString()`을 반환하는 편의 메서드.
- **연결**: → [[jwt-signing]] createAccessToken(loginId)

#### Q. authenticated 토큰을 만들어 SecurityContext에 저장한다는 게 무슨 의미?
- **한줄답**: "이 요청은 이 사용자로 검증됨"이라는 신뢰 표시(방문증)를 걸어두는 것. 등록해야 인가를 통과.
- **원리**: `SecurityContextHolder`는 요청별(ThreadLocal) 인증 보관함. 저장 안 하면 검증했어도 익명 취급 → `.authenticated()`에서 거부. 뒤의 AuthorizationFilter가 이걸 읽어 통과 판단. 검증(과정)과 인정(등록)은 별개.
- **연결**: → [[spring-security-filter]] 인가 필터

#### Q. bCryptPasswordEncoder / AuthenticationManager는 왜 직접 빈으로 등록해야 해? 자동으로 하면 안 돼?
- **한줄답**: PasswordEncoder는 "어떤 알고리즘"인지 개발자만 아는 선택이라 명시 필요. AuthManager는 이미 있고 "꺼내는" 것.
- **원리**: 인코더는 DB 저장 형식과 정확히 일치해야 하고 보안 결정이라 Spring이 임의로 못 고름. 등록해두면 `DaoAuthenticationProvider`가 자동으로 `matches()`에 사용(그래서 AuthService에 호출 코드가 없어도 쓰임). 명시 사용은 JudgeApplication의 테스트 사용자 `encode()`. AuthManager는 배관이라 이미 생성되지만 주입하려면 노출만.
- **연결**: → 위 Q(AuthManager 꺼내기)

## ③ 용어 카드 (역참조)

> [!quote]- 용어 6개
> - **AuthenticationManager**: 인증 진입점(인터페이스). 구현체 `ProviderManager`. → Q.직접 호출
> - **DaoAuthenticationProvider**: DB 기반 인증 제공자. UserDetailsService 조회 + PasswordEncoder 비교. → Q.authenticate 내부
> - **UserDetailsService**: id로 UserDetails를 조회하는 규격(내가 구현). 비번 비교 안 함. → Q.DB 확인
> - **UsernamePasswordAuthenticationToken**: Authentication 구현체. unauthenticated(2-arg)/authenticated(3-arg). → Q.unauthenticated, Q.authenticated
> - **SecurityContextHolder**: 요청별 인증 보관함(ThreadLocal). → Q.SecurityContext 저장
> - **getName()**: principal에서 식별자를 꺼내는 편의 메서드. → Q.getName

## ④ 내가 틀렸던 것 (오개념 로그)

> [!quote]- 오개념 6건
> | 내가 생각했던 것 | 실제 |
> |---|---|
> | loadUserByUsername이 비번도 검증 | 조회만 함. 비교는 DaoAuthenticationProvider가 PasswordEncoder로 |
> | UsernamePasswordAuthenticationToken이 AuthManager 내부에 있음 | 밖에서 만들어 authenticate()에 넘기는 입력 객체 |
> | getName()이 마법으로 뽑힘 | 내부에 UserDetails면 getUsername()을 꺼내는 코드가 있음 |
> | unauthenticated(request) 처럼 1개 인자 | (principal, credentials) 2개 인자 |
> | bCryptPasswordEncoder는 안 쓰임 | DaoAuthenticationProvider가 자동으로 matches에 사용 |
> | 다 자동으로 빈 생성하면 됨 | PasswordEncoder는 선택이라 명시, AuthManager는 이미 있고 꺼내는 것 |
