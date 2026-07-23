# Spring Security 필터 구조

> **주제**: Spring Security 필터 구조 · 갱신: 2026-07-21 · 상태: 진행중
> **태그**: #spring-security #필터 #필터체인

## ① 큰 그림 (지도)

요청은 컨트롤러 도착 전에 **필터 체인(공항 검색대)**을 통과한다. 대부분의 필터는 Spring이 자동 등록하고, **내가 직접 만드는 건 보통 JWT 검증 필터 하나뿐.**

```
요청 → [필터1] → [필터2] → ... → [컨트롤러]

필터 계층(상속):
jakarta.servlet.Filter                 (서블릿 표준)
  └ GenericFilterBean                  (spring-web)
      ├ OncePerRequestFilter           (spring-web)
      │   ├ BasicAuthenticationFilter        (spring-security-web)
      │   ├ BearerTokenAuthenticationFilter  (spring-security-web)
      │   └ [내 JwtFilter]                    ← 여기
      └ AbstractAuthenticationProcessingFilter (spring-security-web)  ★GenericFilterBean 직접 상속
          └ UsernamePasswordAuthenticationFilter
```

**필터 3분류**: (1)인증 "너 누구야" (2)인가 "여기 들어가도 돼"(AuthorizationFilter) (3)인프라(뒤에서 돕는). 내가 신경 쓸 건 사실상 (1)뿐.

## ② 질문 트리 (본문)

### 2026-07-21

#### Q. OncePerRequestFilter는 spring-web 소속이라 Security 필터로 쓰면 안 된다던데?
- **한줄답**: 틀린 조언. spring-web 소속인 건 맞지만, JWT 검증 필터를 이걸로 만드는 게 정석이다.
- **원리**: Spring Security의 인증 필터 인프라 자체가 spring-web의 `GenericFilterBean`/`OncePerRequestFilter`를 뿌리로 삼는다. Security는 필터 기반 클래스를 자기 모듈에서 새로 안 만들고 spring-web 걸 상속해 쓴다.
- **연결**: → 아래 Q(계층), → [[spring-security-auth-flow]]

#### Q. AbstractAuthenticationProcessingFilter는 뭐고 무엇을 상속하나?
- **한줄답**: 폼/자격증명 인증의 공통 뼈대(추상 클래스)이고, **`GenericFilterBean`을 직접 상속**한다 (OncePerRequestFilter 아님).
- **원리**: `attemptAuthentication()`만 하위(UsernamePasswordAuthenticationFilter)가 채우고, 성공/실패 후처리는 부모가 담당. 특정 URL(`/login`)에서만 동작하도록 스스로 걸러서 중복 실행이 문제 안 됨 → OncePerRequestFilter가 필요 없었던 것.
- **연결**: → 아래 Q(왜 JWT엔 OncePer)

#### Q. 그럼 왜 JWT 필터는 OncePerRequestFilter를 쓰나?
- **한줄답**: JWT 검증은 거의 모든 요청에 걸리므로 "요청당 정확히 한 번"이 중요해서.
- **원리**: 그냥 `Filter`면 forward/include/async 재디스패치에서 같은 요청에 여러 번 실행될 수 있다. OncePerRequestFilter가 이걸 막아준다(토큰 검증/SecurityContext 세팅 중복 방지).
- **연결**: → 아래 Q(JwtFilter 작성)

#### Q. JwtFilter는 어떻게 작성하나?
- **한줄답**: `extends OncePerRequestFilter` → `doFilterInternal` 오버라이드: 헤더 토큰 추출 → 검증 → SecurityContext 등록 → `filterChain.doFilter()`.
- **원리**: 헤더가 `Bearer `로 시작 안 하면 그냥 통과. 유효하면 `JwtProvider.isValid/getLoginId`로 검증(secretKey는 JwtProvider 내부 캡슐화)하고 authenticated 토큰을 SecurityContext에 저장. 마지막 `doFilter`를 빼먹으면 요청이 멈춘다.
- **연결**: → [[spring-security-auth-flow]]의 authenticated 토큰, → [[jwt-signing]]의 검증

#### Q. JwtFilter에 @Component를 붙이면 되나?
- **한줄답**: 안 됨. `@Component` + SecurityConfig의 `new JwtFilter(...)` 조합이면 이중 등록된다.
- **원리**: Spring Boot는 `Filter` 타입 빈을 발견하면 서블릿 컨테이너에 자동 등록한다(모든 요청에 실행). 여기에 보안 체인 등록까지 겹치면 필터가 두 번 돈다. → `@Component` 떼고 SecurityConfig에서 `new`로 만들어 `addFilterBefore`로만 등록.
- **연결**: → [[spring-security-auth-flow]]의 SecurityConfig 배선

*OAuth 로그인 뜯어보다 파생 — AbstractAuthenticationProcessingFilter 내부*

#### Q. doFilter랑 successfulAuthentication은 부모 메서드야?
> 그때 왜 궁금했나: 내가 만든 SuccessHandler가 대체 어디서 불리는지 추적하다가.
- **한줄답**: 맞음. 부모가 흐름을 쥐고, 자식은 "인증 방법"만 채운다 (템플릿 메서드 패턴).
- **원리** — 시작 → 과정 → 결과:
    - [시작] `AbstractAuthenticationProcessingFilter.doFilter()` = 로그인 처리 전체 순서를 쥔 뼈대(부모)
    - [과정] `OAuth2LoginAuthenticationFilter.attemptAuthentication()` = 실제 인증 시도(자식이 채움: OAuth면 code검증)
    - [과정] 성공(값 반환) → 부모 `successfulAuthentication()` = SecurityContext 저장 + SuccessHandler 호출
    - [과정] 실패(예외) → 부모 `unsuccessfulAuthentication()` = SecurityContext 비우고 FailureHandler 호출
    - [결과] 어떤 로그인 방식이든 후처리는 부모가 공통 담당
- **되감기란**: 자식에서 예외가 나도 부모 try-catch 안에서 돌기 때문에, 흐름이 결국 부모의 실패 처리로 "되돌아온다". 그래서 되감기.
- **왜 이렇게**: 저장·핸들러 호출은 모든 로그인이 똑같음 → 부모에 한 번만. 매번 다른 "인증 방법"만 자식이.
- **연결**: → 아래 Q(오버라이드), → [[oauth2-google-login]]의 되감기

#### Q. doFilter는 오버라이드 안 하는 거야?
- **한줄답**: 안 함. 부모 것 그대로 사용. 자식이 오버라이드하는 건 `attemptAuthentication` 하나뿐.
- **원리**:
    - 공통 흐름(저장·성공/실패 분기) = 부모 `doFilter`에 완성해둠 → 상속만
    - "인증 방식" = 부모가 `abstract attemptAuthentication`로 비워둠 → 자식이 채움
    - 그래서 폼 로그인이든 OAuth든 attempt만 다르고 후처리는 공유
- **왜 이렇게**: 후처리를 필터마다 복붙하면 중복·누락 위험 → 부모에 한 번만 두는 게 템플릿 메서드 패턴의 이점.
- **연결**: → 아래 Q(핸들러 실행 원리)

#### Q. 내가 등록한 successHandler가 실행되는 원리는?
> 그때 왜 궁금했나: config에 한 줄 등록했을 뿐인데 그게 어떻게 실행까지 이어지는지 마법 같았다.
- **한줄답**: 다형성 + DI. 부모의 `successHandler` 필드가 인터페이스 타입이라, 거기 꽂힌 실제 객체(우리 것)의 메서드가 실행된다.
- **원리** — 시작 → 과정 → 결과:
    - [시작] 부모에 `private AuthenticationSuccessHandler successHandler` = 인터페이스 타입 필드(무엇이 담길지 부모는 모름)
    - [과정] SecurityConfig `.successHandler(우리핸들러)` = 그 필드에 우리 구현체 주입(DI)
    - [과정] `successfulAuthentication()`이 `this.successHandler.onAuthenticationSuccess()` 호출
    - [결과] 자바 동적 디스패치가 "필드에 실제 담긴 객체"의 메서드 실행 = 우리 것 실행
- **비유**: 리모컨 전원버튼(인터페이스 호출)은 하나지만, 페어링된 기기(주입된 구현체)에 따라 켜지는 게 달라짐.
- **주의**: 등록 안 하면 필드에 기본 핸들러가 있어 그게 실행됨(그냥 "/"로 redirect, JWT 안 나옴).
- **연결**: → 아래 Q(필드 주입), → [[oauth2-google-login]]의 SuccessHandler

#### Q. .successHandler(...)는 부모의 successHandler 필드에 넣는 거야?
- **한줄답**: 맞음. (필터 세터를 통해) `AbstractAuthenticationProcessingFilter`의 `successHandler` 필드에 우리 빈을 주입하는 것.
- **원리** — 시작 → 과정 → 결과:
    - [시작] `.successHandler(x)` = config DSL 호출
    - [과정] 내부적으로 `filter.setAuthenticationSuccessHandler(x)` → `this.successHandler = x`
    - [결과] 필드도 부모 소유, 그 필드를 쓰는 `successfulAuthentication`도 부모 소유 → config가 채우고 부모 코드가 호출
- **패턴**: `.userService(...)`, `.authorizationRequestRepository(...)`도 전부 같은 "config로 부품(필드) 꽂기".
- **연결**: → [[oauth2-google-login]]의 SecurityConfig 배선

## ③ 용어 카드 (역참조)

> [!quote]- 용어 9개
> - **GenericFilterBean**: spring-web의 필터 기반 클래스. 모든 Security 필터의 뿌리. → Q.계층
> - **OncePerRequestFilter**: "요청당 1회" 보장 필터(spring-web). JWT 검증용 정석. → Q.왜 JWT엔
> - **AbstractAuthenticationProcessingFilter**: 폼 인증 공통 뼈대. GenericFilterBean 직접 상속. → Q.이게 뭐
> - **doFilterInternal**: OncePerRequestFilter에서 오버라이드하는 실제 필터 로직 메서드. → Q.JwtFilter 작성
> - **addFilterBefore**: 보안 체인의 특정 필터 앞에 커스텀 필터를 끼우는 메서드. → Q.@Component
> - **템플릿 메서드 패턴**: 공통 흐름은 부모(doFilter), 가변 부분만 자식(attemptAuthentication)이 채움. → Q.오버라이드
> - **동적 디스패치(다형성)**: 인터페이스 타입 필드 호출 시 실제 담긴 객체의 메서드 실행. 커스텀 핸들러 실행 원리. → Q.핸들러 실행
> - **successfulAuthentication**: 부모 메서드. SecurityContext 저장 + successHandler 호출 담당. → Q.부모 메서드
> - **config로 부품 꽂기**: .successHandler/.userService/.authorizationRequestRepository 모두 필터·Provider 필드에 우리 빈 주입. → Q.필드 주입

## ④ 내가 틀렸던 것 (오개념 로그)

> [!quote]- 오개념 6건
> | 내가 생각했던 것 | 실제 |
> |---|---|
> | OncePerRequestFilter는 security 소속이 아니라 쓰면 안 됨 | spring-web 소속 맞지만 JWT 필터엔 정석 |
> | AbstractAuthenticationProcessingFilter가 OncePerRequestFilter를 상속 | GenericFilterBean을 직접 상속 (OncePer 안 거침) |
> | 필터에 @Component 붙이면 편함 | Filter 빈은 서블릿에 자동 등록돼 이중 실행 → 떼고 new로 |
> | doFilter도 자식이 오버라이드한다 | 부모 것 그대로 상속. 자식은 attemptAuthentication만 오버라이드 |
> | 필드 접근에 this 안 붙이는 게 맞다 | successHandler는 필드라 this 맞음 (지역변수면 this 없음) |
> | 핸들러가 어떻게 우리 것이 되는지 마법 같다 | 인터페이스 필드 + config 주입 + 다형성. 등록 안 하면 기본 핸들러 |
