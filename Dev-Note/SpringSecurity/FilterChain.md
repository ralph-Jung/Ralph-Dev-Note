# 필터 체인 구조 (어디서 도는가)

> **주제**: Spring Security 필터 체인 — 상속 계층, 커스텀 필터의 기반 클래스 선택과 등록 방식 · 갱신: 2026-07-28 · 상태: 진행중
> **태그**: #spring-security #필터 #체인 #필터등록

## ① 큰 그림 (지도)

요청은 컨트롤러에 닿기 전 **필터 체인(공항 검색대)** 을 통과한다. 이 노트는 **"어디서 도는가"** 만 다룬다 — 개별 필터가 무슨 일을 하는지는 각 관심사 노트로.

```
[체인]
요청 → [필터1] → [필터2] → ... → [컨트롤러]

[상속 계층]
jakarta.servlet.Filter                 (서블릿 표준)
  └ GenericFilterBean                  (spring-web)
      ├ OncePerRequestFilter           (spring-web)
      │   ├ BasicAuthenticationFilter        (spring-security-web)
      │   ├ BearerTokenAuthenticationFilter  (spring-security-web)
      │   └ [내 JwtFilter]                    ← 여기
      └ AbstractAuthenticationProcessingFilter (spring-security-web)  ★GenericFilterBean 직접 상속
          └ UsernamePasswordAuthenticationFilter
```

- **뿌리가 spring-web**: Security는 필터 기반 클래스를 자기 모듈에서 새로 만들지 않고 ==spring-web 것을 상속해 쓴다==. 그래서 "spring-web 소속이라 Security에 쓰면 안 된다"는 말은 성립하지 않는다.
- **필터 3분류**: (1) 인증 "너 누구야" → [[Authentication]] (2) 인가 `AuthorizationFilter` (3) 인프라. 내가 만드는 건 보통 JWT 검증 필터 하나뿐.

거부됐을 때의 응답 결정은 → [[ExceptionHandling]]. 서블릿 필터 일반 규약(`chain.doFilter`, 체인 중단)은 → [[ServletFilter]].

## ② 질문 트리 (본문)

### 2026-07-21

#### Q. OncePerRequestFilter는 spring-web 소속이라 Security 필터로 쓰면 안 된다던데?
- **한줄답**: 틀린 조언. ==spring-web 소속인 건 맞지만, JWT 검증 필터를 이걸로 만드는 게 정석==이다.
- **원리**: Spring Security의 인증 필터 인프라 자체가 spring-web의 `GenericFilterBean`/`OncePerRequestFilter`를 뿌리로 삼는다. Security는 필터 기반 클래스를 자기 모듈에서 새로 안 만들고 spring-web 걸 상속해 쓴다.
- **연결**: → 아래 Q(왜 JWT엔 OncePer), → [[Authentication]] Q.AbstractAuthenticationProcessingFilter는 무엇을 상속하나

#### Q. 그럼 왜 JWT 필터는 OncePerRequestFilter를 쓰나?
- **한줄답**: ==JWT 검증은 거의 모든 요청에 걸리므로 "요청당 정확히 한 번"이 중요해서==.
- **원리**: 그냥 `Filter`면 forward/include/async 재디스패치에서 같은 요청에 여러 번 실행될 수 있다. OncePerRequestFilter가 이걸 막아준다(토큰 검증/SecurityContext 세팅 중복 방지).
- **대비**: `AbstractAuthenticationProcessingFilter`는 특정 URL(`/login`)에서만 동작하도록 **스스로 걸러서** 중복 실행이 문제가 안 됐다 → OncePerRequestFilter가 필요 없었다. 즉 ==기반 클래스 선택은 "이 필터가 몇 개의 경로에 걸리는가"로 갈린다==.
- **연결**: → [[TokenAuth]] Q.JwtFilter는 어떻게 작성하나

#### Q. JwtFilter에 @Component를 붙이면 되나?
- **한줄답**: 안 됨. ==`@Component` + SecurityConfig의 `new JwtFilter(...)` 조합이면 이중 등록==된다.
- **원리**: Spring Boot는 `Filter` 타입 빈을 발견하면 서블릿 컨테이너에 자동 등록한다(모든 요청에 실행). 여기에 보안 체인 등록까지 겹치면 필터가 두 번 돈다. → `@Component` 떼고 SecurityConfig에서 `new`로 만들어 `addFilterBefore`로만 등록.
- **연결**: → [[OAuth2]]의 SecurityConfig 배선

## ③ 용어 카드 (역참조)

> [!quote]- 용어 4개
> - **GenericFilterBean**: spring-web의 필터 기반 클래스. 모든 Security 필터의 뿌리. → Q.OncePerRequestFilter 소속
> - **OncePerRequestFilter**: "요청당 1회" 보장 필터(spring-web). JWT 검증용 정석. → Q.왜 JWT엔
> - **doFilterInternal**: OncePerRequestFilter에서 오버라이드하는 실제 필터 로직 메서드. → [[TokenAuth]] Q.JwtFilter 작성
> - **addFilterBefore**: 보안 체인의 특정 필터 앞에 커스텀 필터를 끼우는 메서드. → Q.@Component

## ④ 내가 틀렸던 것 (오개념 로그)

> [!quote]- 오개념 3건
> | 내가 생각했던 것 | 실제 |
> |---|---|
> | OncePerRequestFilter는 security 소속이 아니라 쓰면 안 됨 | spring-web 소속 맞지만 JWT 필터엔 정석 |
> | AbstractAuthenticationProcessingFilter가 OncePerRequestFilter를 상속 | GenericFilterBean을 직접 상속 (OncePer 안 거침) |
> | 필터에 @Component 붙이면 편함 | Filter 빈은 서블릿에 자동 등록돼 이중 실행 → 떼고 new로 |
