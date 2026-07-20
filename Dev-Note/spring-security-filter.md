# Spring Security 필터 구조

> 주제: Spring Security 필터 구조 · 갱신: 2026-07-20 · 상태: 진행중
> 태그: #spring-security #filter #jwt #서블릿필터
> 다음에 팔 것: SecurityContextHolderFilter/ExceptionTranslationFilter 등 인프라 필터, 필터 실행 순서 전체 목록

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

## ② 질문 트리 (본문) ★핵심

### Q. OncePerRequestFilter는 spring-web 소속이라 Security 필터로 쓰면 안 된다던데?
- **한줄답**: 틀린 조언. spring-web 소속인 건 맞지만, JWT 검증 필터를 이걸로 만드는 게 정석이다.
- **원리**: Spring Security의 인증 필터 인프라 자체가 spring-web의 `GenericFilterBean`/`OncePerRequestFilter`를 뿌리로 삼는다. Security는 필터 기반 클래스를 자기 모듈에서 새로 안 만들고 spring-web 걸 상속해 쓴다.
- **연결**: → 아래 Q(계층), → [[spring-security-auth-flow]]

### Q. AbstractAuthenticationProcessingFilter는 뭐고 무엇을 상속하나?
- **한줄답**: 폼/자격증명 인증의 공통 뼈대(추상 클래스)이고, **`GenericFilterBean`을 직접 상속**한다 (OncePerRequestFilter 아님).
- **원리**: `attemptAuthentication()`만 하위(UsernamePasswordAuthenticationFilter)가 채우고, 성공/실패 후처리는 부모가 담당. 특정 URL(`/login`)에서만 동작하도록 스스로 걸러서 중복 실행이 문제 안 됨 → OncePerRequestFilter가 필요 없었던 것.
- **연결**: → 아래 Q(왜 JWT엔 OncePer)

### Q. 그럼 왜 JWT 필터는 OncePerRequestFilter를 쓰나?
- **한줄답**: JWT 검증은 거의 모든 요청에 걸리므로 "요청당 정확히 한 번"이 중요해서.
- **원리**: 그냥 `Filter`면 forward/include/async 재디스패치에서 같은 요청에 여러 번 실행될 수 있다. OncePerRequestFilter가 이걸 막아준다(토큰 검증/SecurityContext 세팅 중복 방지).
- **연결**: → 아래 Q(JwtFilter 작성)

### Q. JwtFilter는 어떻게 작성하나?
- **한줄답**: `extends OncePerRequestFilter` → `doFilterInternal` 오버라이드: 헤더 토큰 추출 → 검증 → SecurityContext 등록 → `filterChain.doFilter()`.
- **원리**: 헤더가 `Bearer `로 시작 안 하면 그냥 통과. 유효하면 `JwtProvider.isValid/getLoginId`로 검증(secretKey는 JwtProvider 내부 캡슐화)하고 authenticated 토큰을 SecurityContext에 저장. 마지막 `doFilter`를 빼먹으면 요청이 멈춘다.
- **연결**: → [[spring-security-auth-flow]]의 authenticated 토큰, → [[jwt-signing]]의 검증

### Q. JwtFilter에 @Component를 붙이면 되나?
- **한줄답**: 안 됨. `@Component` + SecurityConfig의 `new JwtFilter(...)` 조합이면 이중 등록된다.
- **원리**: Spring Boot는 `Filter` 타입 빈을 발견하면 서블릿 컨테이너에 자동 등록한다(모든 요청에 실행). 여기에 보안 체인 등록까지 겹치면 필터가 두 번 돈다. → `@Component` 떼고 SecurityConfig에서 `new`로 만들어 `addFilterBefore`로만 등록.
- **연결**: → [[spring-security-auth-flow]]의 SecurityConfig 배선

## ③ 용어 카드 (역참조)
- **GenericFilterBean**: spring-web의 필터 기반 클래스. 모든 Security 필터의 뿌리. → Q.계층
- **OncePerRequestFilter**: "요청당 1회" 보장 필터(spring-web). JWT 검증용 정석. → Q.왜 JWT엔
- **AbstractAuthenticationProcessingFilter**: 폼 인증 공통 뼈대. GenericFilterBean 직접 상속. → Q.이게 뭐
- **doFilterInternal**: OncePerRequestFilter에서 오버라이드하는 실제 필터 로직 메서드. → Q.JwtFilter 작성
- **addFilterBefore**: 보안 체인의 특정 필터 앞에 커스텀 필터를 끼우는 메서드. → Q.@Component

## ④ 내가 틀렸던 것 (오개념 로그) ★가치 높음
| 내가 생각했던 것 | 실제 |
|---|---|
| OncePerRequestFilter는 security 소속이 아니라 쓰면 안 됨 | spring-web 소속 맞지만 JWT 필터엔 정석 |
| AbstractAuthenticationProcessingFilter가 OncePerRequestFilter를 상속 | GenericFilterBean을 직접 상속 (OncePer 안 거침) |
| 필터에 @Component 붙이면 편함 | Filter 빈은 서블릿에 자동 등록돼 이중 실행 → 떼고 new로 |
