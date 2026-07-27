# 방어 계층 (공격을 어떻게 막나)

> **주제**: Spring Security 방어 계층 — CSRF, CORS, 보안 응답 헤더 · 갱신: 2026-07-28 · 상태: 진행중 (아직 질문 없음)
> **태그**: #spring-security #보안 #CSRF #CORS

## ① 큰 그림 (지도)

인증도 인가도 아닌, **공격 기법 자체를 막는** 필터들이다. "누구냐/되냐"와 무관하게 요청 형태를 검사한다.

```
CsrfFilter          남의 사이트가 내 쿠키를 업고 요청을 위조하는 것을 차단
CorsFilter          다른 출처(origin)의 브라우저 요청을 허용/차단. preflight(OPTIONS) 처리
HeaderWriterFilter  X-Frame-Options, X-Content-Type-Options, HSTS 등 보안 응답 헤더 부착
```

현재 프로젝트 상태:
- **CSRF는 꺼져 있다**(`.csrf(disable)`). 다만 refresh 토큰을 쿠키로 전달하므로 실제 표적이 될 수 있고, 지금은 `SameSite=Lax` 가 막고 있다 → [[TokenAuth]]
- **CORS 설정이 없다.** 프론트를 붙이는 순간 필요해진다.
- OAuth2의 `state` 는 CSRF 방어지만 OAuth 흐름과 떼면 이해가 안 되므로 → [[OAuth2]] 에 둔다.

## ② 질문 트리 (본문)

아직 이 주제로 파고든 질문이 없다.
