# Swagger Trouble Shooting

> **주제**: springdoc / Swagger UI 를 붙이며 막혔던 사건의 기록 · 갱신: 2026-07-31 · 상태: 진행중
> **태그**: #Swagger #springdoc #트러블슈팅 #에러

## 색인

에러 메시지가 있으면 본 그대로, 에러 없이 설계에서 막힌 건은 "하려던 것" 으로 찾는다.

| 증상 / 하려던 것 | 원인 한 줄 | 항목 |
|---|---|---|
| 스웨거에서 Basic 팝업 대신 `{"message":"인증이 필요합니다"}` | 한 체인에 JWT EntryPoint 와 Basic 이 섞여 충돌 | [1](#스웨거-문서에-basic-인증-걸기) |
| 팝업은 뜨는데 뭘 넣어도 401 | 문서 계정과 서비스 사용자는 다른 축의 신원 | [2](#basic-팝업은-뜨는데-어떤-계정도-안-먹힘) |

---

## 스웨거 문서에 Basic 인증 걸기

[동기] 스웨거 문서가 인증 없이 통째로 공개되면 API 전체 지도가 노출된다고 생각해서, 보안을 걸어야겠다고 판단했다.

[시도] 그래서 `SecurityFilterChain` 에서 스웨거 경로를 `authenticated()` 로 막고, `httpBasic()` 으로 비밀번호를 아는 사람만 접속하게 하려고 했다.

[문제] 그런데 스웨거에 접속하니 Basic Auth 팝업이 뜨는 게 아니라 `{"message":"인증이 필요합니다"}` JSON 만 반환됐다.

[원인] 이 프로젝트는 JWT 기반이라 커스텀 `AuthenticationEntryPoint` 가 등록돼 있었고, 인증 실패 시 무조건 이 JSON 을 뱉고 있었다. Basic Auth 가 띄우려던 팝업 응답(`WWW-Authenticate: Basic` 헤더)을 이게 덮어써 버린 것이다. 즉 ,  ==하나의 필터체인에 JWT 인증과 Basic 인증이 섞이면서 충돌한 것==이었다.

[해결] 그래서 필터체인을 2개로 분리했다. 스웨거 전용 체인은 `@Order(1)` 로 두고 커스텀 엔트리포인트 없이 순수 Basic Auth 만 쓰게, 나머지 API 체인은 `@Order(2)` 로 기존 JWT 설정을 유지하게 했다.

```java
@Bean
@Order(1)
public SecurityFilterChain swaggerChain(HttpSecurity http) throws Exception {
    http.securityMatcher("/swagger-ui/**", "/v3/api-docs/**")
        .authorizeHttpRequests(a -> a.anyRequest().authenticated())
        .httpBasic(Customizer.withDefaults());
    return http.build();
}
```

→ [[FilterChain]] (체인을 여러 개 두고 `securityMatcher` 로 범위를 가르는 구조) · [[ExceptionHandling]] (EntryPoint 가 401 응답을 결정하는 지점)

---

## Basic 팝업은 뜨는데 어떤 계정도 안 먹힘

[동기] 체인을 나눠 팝업까지는 띄웠으니 이제 로그인이 되게 해야 했다.

[시도] 이미 `CustomUserDetailsService` 가 있었다. `BasicAuthenticationFilter` 도 결국 `UserDetailsService` 로 사용자를 찾으니 그걸 그대로 쓰면 될 줄 알았다.

[문제] 무슨 id 와 비밀번호를 넣어도 401 이 돌아왔다.

[원인] `CustomUserDetails.getPassword()` 가 `null` 을 반환하고 있었다. 구글 로그인 전용이라 비밀번호를 아예 보관하지 않는 설계였고, 직전 커밋(V3 마이그레이션)에서 `users.password` 컬럼까지 DROP 한 참이었다. ==문서 계정과 서비스 사용자는 다른 축의 신원이다== — `CustomUserDetailsService` 가 답하는 건 "우리 서비스를 쓰는 사람이 누구냐" 이고, 문서 인증이 묻는 건 "우리 팀원이 맞냐" 다. 있는 걸 재사용하려던 게 아니라 성격이 다른 둘을 합치려던 게 문제였다.

[시도] 그래서 `DocsUserDetailsService` 라는 클래스를 새로 만들고 `@Service` 를 붙였다.

[문제] `@Service` 는 컴포넌트 스캔 대상이라 그대로 빈이 되고, 컨텍스트에 `UserDetailsService` 가 둘이 된다. 나중에 다른 체인에서 `formLogin` 이나 `httpBasic` 을 켜면 문서 계정으로도 서비스 로그인이 뚫린다.

[해결] 문서 계정용 `InMemoryUserDetailsManager` 를 `@Bean` 이 아닌 `private` 메서드로 두고, `http.userDetailsService(...)` 로 그 체인에만 꽂았다. 빈이 아니므로 다른 체인은 이 계정의 존재조차 모른다.

```java
.userDetailsService(docsUserDetailsService())

// @Bean 을 붙이지 않는다. 빈이 되면 UserDetailsService 가 둘이 되어
// 문서 계정이 다른 체인의 인증에도 쓰인다. 이 체인 안에만 가둔다.
private UserDetailsService docsUserDetailsService() {
    UserDetails docs = org.springframework.security.core.userdetails.User
            .withUsername(docsUsername)
            .password(docsPassword)
            .roles("DOCS")
            .build();
    return new InMemoryUserDetailsManager(docs);
}
```

→ [[Authentication]] (`UserDetailsService` 와 `DaoAuthenticationProvider`)
