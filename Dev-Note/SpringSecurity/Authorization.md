# 인가 (그거 해도 되냐)

> **주제**: Spring Security 인가 — 규칙 표현, 권한 모델, 메서드 시큐리티 · 갱신: 2026-07-28 · 상태: 진행중 (아직 질문 없음)
> **태그**: #spring-security #인가 #권한 #메서드시큐리티

## ① 큰 그림 (지도)

인가는 **"이 요청자가 요구 조건을 만족하나"** 를 검사하는 단계다. 신원 확인(인증)이 끝난 뒤에 온다.

```
인증 (너 누구냐)        →  인가 (그거 해도 되냐)
[[Authentication]]          AuthorizationFilter (체인 맨 끝)
SecurityContext 채움          그 컨텍스트를 읽어 규칙과 대조
                              └ 위반 → AccessDeniedException → [[ExceptionHandling]]
```

- **"인증됨"도 하나의 자격 조건**이다. `.authenticated()` 와 `.hasRole("ADMIN")` 은 `AuthorizationFilter` 에겐 같은 종류의 검사다.
- 검사 지점이 두 군데다 — **URL 기반**(`requestMatchers`, 필터 단계)과 **메서드 기반**(`@PreAuthorize`, AOP). 후자는 필터 밖에서 터지므로 예외 전파 경로가 다르다 → [[ExceptionHandling]]

현재 프로젝트는 `.anyRequest().authenticated()` 하나뿐이라 403 경로가 아직 실행되지 않는다.

## ② 질문 트리 (본문)

아직 이 주제로 파고든 질문이 없다.
