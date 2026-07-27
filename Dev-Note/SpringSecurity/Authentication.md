# 인증 (너 누구냐)

> **주제**: Spring Security 인증 — 로그인 흐름(AuthenticationManager → Provider)과 인증 필터의 부모/자식 구조 · 갱신: 2026-07-28 · 상태: 진행중
> **태그**: #spring-security #인증 #로그인 #SecurityContext

## ① 큰 그림 (지도)

인증은 **"너 누구냐"를 확인하고, 그 결과를 SecurityContext에 걸어두는 것**까지다. 검증(과정)과 인정(등록)은 별개다.

```
[로그인·인증 흐름]
POST /auth/login → AuthController → AuthService.login()
  → authenticationManager.authenticate( unauthenticated(id, pw) )
      → ProviderManager → DaoAuthenticationProvider
          → UserDetailsService.loadUserByUsername(id)   (DB 조회만)
          → PasswordEncoder.matches(입력pw, DB암호화pw)  (비교)
  → 성공: authenticated Authentication → JwtProvider.createAccessToken(getName()) → TokenResponse
```

- **토큰 상태 변화**: `unauthenticated`(입력) → `authenticate()` → `authenticated`(결과). SecurityContext에 저장해야 인정됨.
- **로그인 방식 3가지**: A=컨트롤러+AuthManager 직접(채택) / B=필터 상속(세션 정석, JWT엔 번거로움) / C=서비스 수동 검증.
- 비유: 로그인=경비원이 신분증(id/pw) 확인 후 방문증(authenticated 토큰) 발급 → SecurityContext에 걸어야 인정.

```
[인증 필터의 부모/자식 — 템플릿 메서드]
AbstractAuthenticationProcessingFilter (부모)
  ├ doFilter()                  흐름 전체를 쥠           ← 자식이 건드리지 않음
  ├ attemptAuthentication()     abstract, 자식이 채움    ← 로그인 "방식"만 다름
  ├ successfulAuthentication()  SecurityContext 저장 + SuccessHandler 호출
  └ unsuccessfulAuthentication() 컨텍스트 비우고 FailureHandler 호출
```

필터를 어떤 기반 클래스로 만들고 어디에 끼우는가는 → [[FilterChain]]. 거부됐을 때의 응답은 → [[ExceptionHandling]].

## ② 질문 트리 (본문)

### 2026-07-20

#### Q. 클라이언트 POST /login부터 SecurityContext 저장까지 9단계 흐름이 맞아?
- **한줄답**: 거의 맞고, 딱 한 군데 틀림 — ==`loadUserByUsername()`은 비밀번호를 검증하지 않는다==.
- **원리**: `loadUserByUsername`은 id로 **DB에서 사용자 조회만** 한다(파라미터에 pw도 없음). 비번 대조는 `DaoAuthenticationProvider`가 `PasswordEncoder.matches()`로 별도 수행. 성공 시엔 새 authenticated 토큰이 생성됨(기존 토큰 필드 변경 아님).
- **연결**: → 아래 Q(authenticate 내부), → [[TokenAuth]] Q.JwtFilter는 어떻게 작성하나

#### Q. AuthenticationManager를 "직접 호출/직접 만든다"는 게 무슨 뜻?
- **한줄답**: 필터 대신 컨트롤러/서비스에서 `authenticate()`를 직접 부르는 것(=B 안 씀). =="만든다"는 새로 짜는 게 아니라 이미 있는 걸 빈으로 꺼내는 것==.
- **원리**: `UsernamePasswordAuthenticationFilter`도 내부에서 결국 `authenticate()`를 부른다. 방식 A는 그 호출을 컨트롤러에 드러낼 뿐. AuthManager는 Security가 내부에 이미 만들지만 필터 체인에 숨어있어, 주입하려면 `AuthenticationConfiguration.getAuthenticationManager()`로 노출해야 함.
- **연결**: → 아래 Q(왜 빈 등록)

#### Q. authenticate()는 DB의 사용자를 어떻게 확인하나? DB에서 직접 가져와?
- **한줄답**: ==내가 만든 `UserDetailsService`가 DB에서 꺼내고(조회), Spring이 `PasswordEncoder`로 비교한다==.
- **원리**: Spring은 내 DB 구조를 모른다 → `loadUserByUsername`(내 코드)이 조회 담당. 비번은 `WHERE password=?`가 아니라, id로 꺼낸 뒤 `passwordEncoder.matches(평문, 암호화값)`로 대조(BCrypt라 직접 비교 불가).
- **연결**: → [[TokenAuth]] (createAccessToken은 성공 후)

#### Q. UserDetailsService는 Spring Security 안 쓰면 없어도 됐던 거네?
- **한줄답**: 맞다. ==AuthManager에 위임하니 필요한 것==. 직접 검증(방식 C)하면 UserRepository+PasswordEncoder만으로 되고 UserDetailsService 불필요.
- **원리**: `UserDetailsService`는 Security 인증 엔진의 규격. A/B는 AuthManager를 쓰므로 필수, C는 서비스에서 `findByLoginId` + `matches`를 직접 해서 생략 가능.
- **연결**: → 아래 Q(10년차 선택)

#### Q. 10년차라면 로그인을 어떻게? 로그인은 signup이랑 다른 service야?
- **한줄답**: ==방식 A(컨트롤러+AuthManager) 추천==. signup은 평범한 CRUD 서비스, login은 인증 엔진을 태우는 별개 흐름.
- **원리**: 필터(B)는 세션 폼로그인용이라 JWT엔 JSON 파싱/토큰 삽입 등 우회가 많다. 로그인도 하나의 API라 컨트롤러가 테스트/문서화에 유리. signup=`userRepository.save()`(Security 무관), login=AuthManager 위임.
- **연결**: → 아래 2026-07-21 Q(AbstractAuthenticationProcessingFilter = 방식 B의 실체)

#### Q. unauthenticated는 왜 쓰고, 왜 id/pw 2개 인자를 넣나?
- **한줄답**: 로그인 입력은 "아직 검증 안 된" 상태라서. ==인자는 `(principal, credentials)` = (loginId, password)==.
- **원리**: Spring Security는 인증을 "누구(principal)+증명(credentials)"로 모델링. `authenticate()`는 unauthenticated 입력을 받아 authenticated를 반환하는 함수. `.unauthenticated()`는 "검증 전" 의도를 이름에 드러낸 팩토리(2-arg 생성자와 동일).
- **연결**: → 아래 Q(authenticated)

#### Q. authenticated 토큰은 어디서 만들고, 왜 3개 인자여야 하나?
- **한줄답**: JwtFilter가 검증 후 직접 만들고(그리고 authenticate() 반환값도 그것). ==authenticated는 권한(authorities)이 필수라 3-arg 생성자로만 생성==.
- **원리**: `setAuthenticated(true)`를 외부에서 부르면 예외(오버라이드로 막음) → 권한을 넘기는 3-arg 생성자 내부에서 `super.setAuthenticated(true)`로 우회. 검증 없이 "인증됨"으로 위조하는 권한 상승을 차단하는 설계. 대안: `.authenticated(principal, creds, authorities)` 팩토리.
- **연결**: → [[TokenAuth]] Q.JwtFilter는 어떻게 작성하나, → 아래 Q(SecurityContext)

#### Q. authentication.getName()이 어떻게 바로 loginId를 뽑아?
- **한줄답**: 마법 아님. ==`getName()` 내부에 "principal이 UserDetails면 `getUsername()`을 꺼낸다"는 코드가 있다==.
- **원리**: 검증 후 principal은 문자열 id에서 `UserDetails` 객체로 승격됨. `AbstractAuthenticationToken.getName()`이 타입을 보고 UserDetails면 `getUsername()`, 문자열이면 `toString()`을 반환하는 편의 메서드.
- **연결**: → [[TokenAuth]] createAccessToken(loginId)

#### Q. authenticated 토큰을 만들어 SecurityContext에 저장한다는 게 무슨 의미?
- **한줄답**: =="이 요청은 이 사용자로 검증됨"이라는 신뢰 표시(방문증)를 걸어두는 것==. 등록해야 인가를 통과.
- **원리**: `SecurityContextHolder`는 요청별(ThreadLocal) 인증 보관함. 저장 안 하면 검증했어도 익명 취급 → `.authenticated()`에서 거부. 뒤의 AuthorizationFilter가 이걸 읽어 통과 판단. 검증(과정)과 인정(등록)은 별개.
- **연결**: → [[FilterChain]] (필터 계층), → [[ExceptionHandling]] (저장 안 됐을 때 401/403이 갈리는 지점)

#### Q. bCryptPasswordEncoder / AuthenticationManager는 왜 직접 빈으로 등록해야 해? 자동으로 하면 안 돼?
- **한줄답**: ==PasswordEncoder는 "어떤 알고리즘"인지 개발자만 아는 선택이라 명시 필요==. AuthManager는 이미 있고 "꺼내는" 것.
- **원리**: 인코더는 DB 저장 형식과 정확히 일치해야 하고 보안 결정이라 Spring이 임의로 못 고름. 등록해두면 `DaoAuthenticationProvider`가 자동으로 `matches()`에 사용(그래서 AuthService에 호출 코드가 없어도 쓰임). 명시 사용은 JudgeApplication의 테스트 사용자 `encode()`. AuthManager는 배관이라 이미 생성되지만 주입하려면 노출만.
- **연결**: → 위 Q(AuthManager 꺼내기)

---

### 2026-07-21

*OAuth 로그인 뜯어보다 파생 — AbstractAuthenticationProcessingFilter 내부*

#### Q. AbstractAuthenticationProcessingFilter는 뭐고 무엇을 상속하나?
- **한줄답**: 폼/자격증명 인증의 공통 뼈대(추상 클래스)이고, ==`GenericFilterBean`을 직접 상속==한다 (OncePerRequestFilter 아님).
- **원리**: `attemptAuthentication()`만 하위(UsernamePasswordAuthenticationFilter)가 채우고, 성공/실패 후처리는 부모가 담당. 특정 URL(`/login`)에서만 동작하도록 스스로 걸러서 중복 실행이 문제 안 됨 → OncePerRequestFilter가 필요 없었던 것.
- **연결**: → [[FilterChain]] Q.그럼 왜 JWT 필터는 OncePerRequestFilter를 쓰나

#### Q. doFilter랑 successfulAuthentication은 부모 메서드야?
> 그때 왜 궁금했나: 내가 만든 SuccessHandler가 대체 어디서 불리는지 추적하다가.
- **한줄답**: 맞음. ==부모가 흐름을 쥐고, 자식은 "인증 방법"만 채운다 (템플릿 메서드 패턴)==.
- **원리** — 시작 → 과정 → 결과:
    - [시작] `AbstractAuthenticationProcessingFilter.doFilter()` = 로그인 처리 전체 순서를 쥔 뼈대(부모)
    - [과정] `OAuth2LoginAuthenticationFilter.attemptAuthentication()` = 실제 인증 시도(자식이 채움: OAuth면 code검증)
    - [과정] 성공(값 반환) → 부모 `successfulAuthentication()` = SecurityContext 저장 + SuccessHandler 호출
    - [과정] 실패(예외) → 부모 `unsuccessfulAuthentication()` = SecurityContext 비우고 FailureHandler 호출
    - [결과] 어떤 로그인 방식이든 후처리는 부모가 공통 담당
- **되감기란**: 자식에서 예외가 나도 부모 try-catch 안에서 돌기 때문에, 흐름이 결국 부모의 실패 처리로 "되돌아온다". 그래서 되감기.
- **왜 이렇게**: 저장·핸들러 호출은 모든 로그인이 똑같음 → 부모에 한 번만. 매번 다른 "인증 방법"만 자식이.
- **연결**: → 아래 Q(오버라이드), → [[OAuth2]]의 되감기

#### Q. doFilter는 오버라이드 안 하는 거야?
- **한줄답**: 안 함. 부모 것 그대로 사용. ==자식이 오버라이드하는 건 `attemptAuthentication` 하나뿐==.
- **원리**:
    - 공통 흐름(저장·성공/실패 분기) = 부모 `doFilter`에 완성해둠 → 상속만
    - "인증 방식" = 부모가 `abstract attemptAuthentication`로 비워둠 → 자식이 채움
    - 그래서 폼 로그인이든 OAuth든 attempt만 다르고 후처리는 공유
- **왜 이렇게**: 후처리를 필터마다 복붙하면 중복·누락 위험 → 부모에 한 번만 두는 게 템플릿 메서드 패턴의 이점. 패턴 자체는 → [[JavaBasics]]
- **연결**: → 아래 Q(핸들러 실행 원리)

#### Q. 내가 등록한 successHandler가 실행되는 원리는?
> 그때 왜 궁금했나: config에 한 줄 등록했을 뿐인데 그게 어떻게 실행까지 이어지는지 마법 같았다.
- **한줄답**: ==다형성 + DI. 부모의 `successHandler` 필드가 인터페이스 타입이라, 거기 꽂힌 실제 객체(우리 것)의 메서드가 실행==된다.
- **원리** — 시작 → 과정 → 결과:
    - [시작] 부모에 `private AuthenticationSuccessHandler successHandler` = 인터페이스 타입 필드(무엇이 담길지 부모는 모름)
    - [과정] SecurityConfig `.successHandler(우리핸들러)` = 그 필드에 우리 구현체 주입(DI)
    - [과정] `successfulAuthentication()`이 `this.successHandler.onAuthenticationSuccess()` 호출
    - [결과] 자바 동적 디스패치가 "필드에 실제 담긴 객체"의 메서드 실행 = 우리 것 실행
- **비유**: 리모컨 전원버튼(인터페이스 호출)은 하나지만, 페어링된 기기(주입된 구현체)에 따라 켜지는 게 달라짐.
- **주의**: 등록 안 하면 필드에 기본 핸들러가 있어 그게 실행됨(그냥 "/"로 redirect, JWT 안 나옴).
- **연결**: → 아래 Q(필드 주입), → [[OAuth2]]의 SuccessHandler, → [[JavaBasics]] (동적 디스패치)

#### Q. .successHandler(...)는 부모의 successHandler 필드에 넣는 거야?
- **한줄답**: 맞음. ==(필터 세터를 통해) `AbstractAuthenticationProcessingFilter`의 `successHandler` 필드에 우리 빈을 주입==하는 것.
- **원리** — 시작 → 과정 → 결과:
    - [시작] `.successHandler(x)` = config DSL 호출
    - [과정] 내부적으로 `filter.setAuthenticationSuccessHandler(x)` → `this.successHandler = x`
    - [결과] 필드도 부모 소유, 그 필드를 쓰는 `successfulAuthentication`도 부모 소유 → config가 채우고 부모 코드가 호출
- **패턴**: `.userService(...)`, `.authorizationRequestRepository(...)`도 전부 같은 "config로 부품(필드) 꽂기".
- **연결**: → [[OAuth2]]의 SecurityConfig 배선

## ③ 용어 카드 (역참조)

> [!quote]- 용어 9개
> - **AuthenticationManager**: 인증 진입점(인터페이스). 구현체 `ProviderManager`. → Q.직접 호출
> - **DaoAuthenticationProvider**: DB 기반 인증 제공자. UserDetailsService 조회 + PasswordEncoder 비교. → Q.authenticate 내부
> - **UserDetailsService**: id로 UserDetails를 조회하는 규격(내가 구현). 비번 비교 안 함. → Q.DB 확인
> - **UsernamePasswordAuthenticationToken**: Authentication 구현체. unauthenticated(2-arg)/authenticated(3-arg). → Q.unauthenticated, Q.authenticated
> - **SecurityContextHolder**: 요청별 인증 보관함(ThreadLocal). 저장해야 인정됨. → Q.SecurityContext 저장
> - **getName()**: principal에서 식별자를 꺼내는 편의 메서드. UserDetails면 getUsername(). → Q.getName
> - **AbstractAuthenticationProcessingFilter**: 폼 인증 공통 뼈대. GenericFilterBean 직접 상속. → Q.이게 뭐
> - **successfulAuthentication**: 부모 메서드. SecurityContext 저장 + successHandler 호출 담당. → Q.부모 메서드
> - **config로 부품 꽂기**: .successHandler/.userService/.authorizationRequestRepository 모두 필터·Provider 필드에 우리 빈 주입. → Q.필드 주입

## ④ 내가 틀렸던 것 (오개념 로그)

> [!quote]- 오개념 8건
> | 내가 생각했던 것 | 실제 |
> |---|---|
> | loadUserByUsername이 비번도 검증 | 조회만 함. 비교는 DaoAuthenticationProvider가 PasswordEncoder로 |
> | UsernamePasswordAuthenticationToken이 AuthManager 내부에 있음 | 밖에서 만들어 authenticate()에 넘기는 입력 객체 |
> | getName()이 마법으로 뽑힘 | 내부에 UserDetails면 getUsername()을 꺼내는 코드가 있음 |
> | unauthenticated(request) 처럼 1개 인자 | (principal, credentials) 2개 인자 |
> | bCryptPasswordEncoder는 안 쓰임 | DaoAuthenticationProvider가 자동으로 matches에 사용 |
> | 다 자동으로 빈 생성하면 됨 | PasswordEncoder는 선택이라 명시, AuthManager는 이미 있고 꺼내는 것 |
> | doFilter도 자식이 오버라이드한다 | 부모 것 그대로 상속. 자식은 attemptAuthentication만 오버라이드 |
> | 핸들러가 어떻게 우리 것이 되는지 마법 같다 | 인터페이스 필드 + config 주입 + 다형성. 등록 안 하면 기본 핸들러 |
