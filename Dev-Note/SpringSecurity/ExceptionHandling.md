# 예외 처리 (거부되면 뭘 돌려주나)

> **주제**: 보안 예외의 발생·전파·응답 결정 — 필터 예외가 @RestControllerAdvice에 안 잡히는 이유, 그리고 401 vs 403이 갈리는 지점 · 갱신: 2026-07-28 · 상태: 진행중
> **태그**: #spring-security #필터 #예외처리 #ErrorResponse
> **확인 필요**: `ExceptionHandlingConfigurer` 가 명시된 EntryPoint를 우선하는 내부 구조와, ClientRegistration이 1개일 때 로그인 페이지를 건너뛰는 `OAuth2LoginConfigurer` 분기는 소스를 직접 열어 확인하지 않았다. 동작(302 → 401 전환) 자체는 curl로 확인됨.

## ① 큰 그림 (지도)

핵심은 하나 — **범위의 문제가 아니라 위치의 문제다.**

```
[예외 처리기들의 사정권]

[요청] → Filter1 → JwtFilter → ... → 31.ExceptionTranslationFilter → 32.AuthorizationFilter → DispatcherServlet → Controller
                       ↑                        │                                                      │
                  여기서 예외                    └ 자기보다 "뒤"에서 난                                    └ 여기서부터 advice 사정권
                       │                          AuthenticationException /
                       │                          AccessDeniedException 만 번역
                       │
                  둘 다의 사각지대

[못 잡힌 예외가 가는 곳]
필터에서 throw → 컨테이너 도달 → /error 로 ERROR 디스패치 → BasicErrorController → 기본 JSON
{"timestamp":"...","status":500,"error":"Internal Server Error","path":"/..."}
→ 내 ErrorResponse 포맷도 아니고, 401이어야 할 게 500으로 나간다
```

비유: `@ExceptionHandler` 는 소방서에 걸어둔 대응 매뉴얼이다. "어떤 화재든 출동한다"고 아무리 넓게 써놔도 **신고 전화가 안 오면** 출동하지 않는다. 신고 전화 = DispatcherServlet이 예외를 받고 "처리할 핸들러 있나?" 뒤지는 동작.

사정권 안에서는 다시 **누가 무엇을 판정하는가**로 나뉜다. 거부를 판정하는 곳과 그 사유를 아는 곳이 서로 다르다.

```
[사정권 안의 역할 분담]

JwtFilter                  사유는 안다(만료/위조). 막을 권한이 없다 → 통과시킴
AnonymousAuthenticationFilter   빈 SecurityContext에 익명 토큰 → 401/403 판정의 근거가 여기서 생김
ExceptionTranslationFilter      try { 다음 필터 호출 }
AuthorizationFilter        막을 권한은 있다. 사유는 모른다 → AccessDeniedException "하나만"
        ↑ 예외가 거슬러 올라옴 (필터는 줄이 아니라 중첩된 함수 호출)
ExceptionTranslationFilter 의 catch → 익명? EntryPoint(401) : AccessDeniedHandler(403)
```

비유2: JwtFilter는 목격자, AuthorizationFilter는 경비원, ExceptionTranslationFilter는 둘을 부르는 관리자. 경비원은 "안 됩니다"만 알고, 왜 안 되는지는 목격자만 안다.

필터가 응답을 왜 직접 써야 하는지(체인 중단·response 공유)는 → [[ServletFilter]].

## ② 질문 트리 (본문)

### 2026-07-26

#### Q. commence 이게 뭔데?
- **한줄답**: `AuthenticationEntryPoint` 인터페이스에 딱 하나 있는 메서드. 이름 뜻은 "인증 절차를 개시(commence)한다".
- **원리** — 시작 → 과정 → 결과:
    - [시작] Spring Security가 원래 **폼 로그인 기준**으로 설계됐다. 인증 안 된 사용자가 보호된 페이지에 오면 "로그인 화면으로 보내 인증을 시작시켜라"가 이 메서드의 원래 역할
    - [과정] 그래서 폼 로그인 기본 구현은 `response.sendRedirect("/login")` — 인증 시작 = 로그인 페이지로
    - [과정] REST API에는 리다이렉트할 화면이 없다 → 의미가 바뀐다. 401 + JSON을 내려주고 "인증을 시작하라"는 신호는 **응답 코드로** 전달한다
    - [결과] ==그래서 이름이 "인증 실패 처리"가 아니라 "인증 개시"다==
- **시그니처**: `void commence(HttpServletRequest, HttpServletResponse, AuthenticationException) throws IOException, ServletException`
- **주의**: 반환 타입이 `void` 인 게 포인트 — ==값을 리턴하는 게 아니라 response 객체에 직접 써넣는다.== 세 번째 파라미터 `AuthenticationException` 이 실패 사유이고, 이 **타입을 보고 만료인지 위조인지 구분**할 수 있다.
- **호출 주체**: 보통 `ExceptionTranslationFilter`. 필터가 직접 부르는 방식도 있다.
- **연결**: → 아래 Q.401과 403은 어디서 갈리나 (`handle` 과의 대비), → [[ServletFilter]] Q.commence에서 response객체를 만들었잖아 (체인이 거기서 끝나는 이유)

#### Q. @ExceptionHandler(Throwable.class)로 넓혀도 필터 예외는 못 잡아? 이거는 위치의 문제인 거야?
- **한줄답**: 맞다. ==범위가 아니라 위치의 문제==라 모든 예외의 부모인 `Throwable` 로 잡아도 결과가 같다.
- **원리** — 시작 → 과정 → 결과:
    - [시작] `@ExceptionHandler` 가 정하는 건 "어떤 타입을 잡을까" 하나뿐이다. "언제 실행될까"는 DispatcherServlet이 정한다
    - [과정] advice는 능동적으로 예외를 감시하는 게 아니라, ==DispatcherServlet이 예외를 받았을 때 "이걸 처리할 핸들러 있나?" 하고 뒤져서 호출해주는 수동적 구조==다
    - [과정] 필터는 DispatcherServlet 앞단이라 예외가 거기까지 도달하지 못한다 → advice를 실행시켜 줄 주체가 없다
    - [과정] 톰캣이 받아서 `/error` 로 ERROR 디스패치 → `BasicErrorController` 가 기본 JSON을 낸다
    - [결과] `/error` 로 재진입할 때도 원래 예외는 다시 던져지는 게 아니라 ==`jakarta.servlet.error.exception` 이라는 request attribute로만 전달==돼서, advice가 반응할 대상 자체가 없다
- **증명**: 같은 `IllegalArgumentException` 이라도 **컨트롤러·서비스에서 던지면 advice가 정상적으로 잡는다.** 타입은 동일한데 결과가 다르다는 건, 변수가 타입이 아니라 위치라는 뜻이다.
- **연결**: → [[ServletFilter]] (필터가 응답을 직접 써야 하는 이유), → [[JavaBasics]] (컨트롤러·서비스 예외가 advice까지 가는 흐름)

#### Q. 아 그러면 ExceptionTranslationFilter가 필터 내에서 발생하는 예외를 잡는 역할이야?
- **한줄답**: 아니다. ==`AuthenticationException` 과 `AccessDeniedException` 두 타입만 처리한다.== `IllegalArgumentException` 은 그냥 통과시켜 위로 올려보낸다.
- **원리**: `ExceptionTranslationFilter` 는 `chain.doFilter()` 를 try-catch로 감싼 필터일 뿐이다. try 블록 안 = 자기 뒤쪽 필터들이므로, 거기서 난 두 타입만 번역한다. 타입 제한이 1차 이유이고, 순서가 2차 조건이다 — `addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)` 로 등록한 내 필터는 **인증 필터들이 놓이는 구간**에 들어가는데, 그 구간은 ==`ExceptionTranslationFilter` 보다 앞==이다. 순서를 고쳐 뒤에 놓더라도 타입이 안 맞으면 여전히 못 잡는다. (순서가 어떻게 정해지는지는 → [[FilterChain]])
- **401 vs 403 판정**: `AnonymousAuthenticationFilter` 가 비어 있는 SecurityContext에 익명 토큰을 채워둔다. 그보다 **뒤**의 `AuthorizationFilter` 가 막으면, 그 **바로 앞**의 `ExceptionTranslationFilter` 가 잡아서 — 익명이면 `AuthenticationEntryPoint`(401), 인증은 됐는데 권한이 없으면 `AccessDeniedHandler`(403).
- **주의**: 그래서 필터가 `ExceptionTranslationFilter` 보다 앞에 있다면 ==응답을 스스로 마무리 짓는 것 말고 선택지가 없다== — "직접 처리하거나, 아예 예외를 안 내보내거나" 둘 중 하나다.
- **연결**: → 위 Q.commence 이게 뭔데, → [[FilterChain]] (필터 체인 순서)

#### Q. 그러면 필터에서 예외가 났을 때 내가 원하는 ErrorResponse로 응답을 전달할 방법은 없어? 직접 new ErrorResponse로 만들어서 return 해야 하나?
- **한줄답**: 방법은 있고, ==advice를 포기할 필요도 없다.== 직접 만드는 건 세 가지 중 하나일 뿐이다.
- **원리**: 
	- ① **필터에서 직접 JSON 작성** — `ObjectMapper` 로 직렬화해 `response.getWriter()` 에 쓴다. 간단하지만 상태코드·인코딩·직렬화를 필터가 직접 하게 되어 응답 포맷 로직이 `GlobalExceptionHandler`(이는 컴트롤러에서 발생하는 예외일 경우에 해당) 와 필터 두 군데로 갈라진다. 나중에 응답 스펙에 필드를 추가할 때 한 쪽만 고치는 사고가 난다. 
	- ② **HandlerExceptionResolver 위임** — `HandlerExceptionResolver` 는 DispatcherServlet이 예외 처리에 쓰는 바로 그 컴포넌트다. 필터에 직접 주입해 `resolver.resolveException(request, response, null, e)` 를 호출하면 ==DispatcherServlet을 안 거치고도 advice의 @ExceptionHandler를 태울 수 있다.== 우회로를 하나 뚫는 셈이다. 
	- ③ **애초에 예외를 안 던지기** — 토큰이 이상하면 인증 안 된 상태로 통과시키고 `AuthenticationEntryPoint` 가 401을 내게 둔다.
- **주의(②의 함정)**: `HandlerExceptionResolver` 타입 빈이 여러 개라 `@Qualifier("handlerExceptionResolver")` 가 필수다. 그런데 ==Lombok의 `@RequiredArgsConstructor` 는 필드에 붙은 `@Qualifier` 를 생성자 파라미터로 복사하지 않는다== → qualifier가 유실돼 `NoUniqueBeanDefinitionException` 이 난다. 생성자를 직접 쓰거나 `lombok.config` 에 `lombok.copyableAnnotations += org.springframework.beans.factory.annotation.Qualifier` 를 넣어야 한다.
- **공통점**: 어느 방식이든 응답 작성 코드는 EntryPoint(또는 advice) 안에만 있다 — 그게 책임 분리가 지켜지는 지점이다.
- **연결**: → [[JavaBasics]] (HandlerExceptionResolver가 advice를 찾는 상세)

#### Q. isValid에서 return false만 하고 토큰 없이 다음 필터로 넘기면, 어차피 뒤에서 토큰 없다는 이유로 잡히는 거 아닌가?
- **한줄답**: 맞다. 다만 "그 다음 필터"가 아니라 ==체인 거의 끝의 `AuthorizationFilter`==이고, 그 **바로 앞**의 `ExceptionTranslationFilter` 가 잡아서 **EntryPoint**를 부른다.
- **원리** — 시작 → 과정 → 결과:
    - [시작] `JwtFilter` 가 SecurityContext를 안 채우고 `chain.doFilter()` 로 통과시킨다
    - [과정] 뒤쪽의 `AnonymousAuthenticationFilter` 가 비어 있는 컨텍스트에 익명 토큰을 채운다 → 그래서 그 뒤에서 보는 건 "인증 정보 없음"이 아니라 "익명 사용자"다
    - [과정] 체인 맨 끝의 `AuthorizationFilter` 가 인가 검사에서 막고 `AccessDeniedException` 을 던진다
    - [과정] 그 바로 앞의 `ExceptionTranslationFilter` 가 그걸 잡아 익명 여부를 판정 → 익명이므로 `AuthenticationEntryPoint` 호출
    - [결과] 401이 나간다. 다만 ==EntryPoint를 등록 안 하면 기본값이 302 리다이렉트==다 (oauth2Login이 켜져 있으면 구글 로그인으로). 프론트가 401을 못 받으면 refresh 분기 자체가 안 돈다
- **연결**: → [[TokenAuth]] (refresh 재발급 흐름), → [[OAuth2]]

#### Q. 안 던지고 EntryPoint에 맡기는 게 원래 정석이야?
- **한줄답**: 순수한 "전부 통과"는 정석이 아니다. Spring Security 자체 필터는 ==없으면 통과, 있는데 틀리면 즉시 거절==하는 하이브리드다.
- **원리**: `BearerTokenAuthenticationFilter`(JWT용 공식 필터)와 `BasicAuthenticationFilter` 는 둘 다 같은 모양이다 — 토큰이 없으면 `chain.doFilter()` 로 조용히 통과시키고, 토큰이 있는데 인증에 실패하면 그 자리에서 `AuthenticationEntryPoint` 를 부르고 체인을 끊는다. 헤더가 없는 건 `permitAll` 경로(로그인·회원가입)에서 정상이므로 예외로 다룰 일이 아니고, 헤더를 보냈는데 틀린 건 클라이언트 오류의 명확한 신호라 즉시 알려주는 게 맞다.
- **왜 이렇게**: 전부 통과시키면 ==실패 사유(만료/서명불일치/타입불일치)가 EntryPoint까지 전달되지 않는다.== 그래서 request attribute에 사유를 심는 우회책이 필요해진다. 하이브리드는 실패 지점이 곧 필터 안이라 예외 객체를 그대로 손에 쥐고 있어 그 트릭이 불필요하다.
- **주의**: 정리하면 세 방식의 성격 — 직접 작성(포맷 중복) / Resolver 위임(advice 재사용) / 안 던지기 + EntryPoint(Security 표준 흐름, 401 의미에 부합).
- **연결**: → [[ServletFilter]] (체인 중단 메커니즘), → [[TroubleShooting]]

---

### 2026-07-27

#### Q. 인증 없이 API를 호출했더니 401이 아니라 구글 로그인으로 302가 나갔다. 왜?
- **한줄답**: ==302를 보낸 건 구글이 아니라 우리 서버==다. `oauth2Login()` 이 EntryPoint 슬롯에 몰래 심어둔 기본 구현이 한 일이다.
- **원리** — 시작 → 과정 → 결과:
    - [시작] `.exceptionHandling()` 을 설정하지 않으면 EntryPoint 슬롯이 비어 있고, `oauth2Login()` 이 자기 기본값을 후보로 등록해둔다
    - [과정] 미인증 요청이 거부되면 그 기본 구현(`LoginUrlAuthenticationEntryPoint`)의 `commence()` 가 `sendRedirect("/oauth2/authorization/google")` 을 호출한다 → **302 ①**
    - [과정] 브라우저가 그 경로를 재요청하면 `OAuth2AuthorizationRequestRedirectFilter` 가 state를 만들고 `accounts.google.com` 으로 보낸다 → **302 ②**
    - [결과] 302는 두 번 일어난다. ①이 문제이고 ②는 정상적인 로그인 시작 과정이다
- **왜 하필 구글이었나**: 스프링이 "로그인 수단"으로 인지하는 건 `.formLogin()` 과 `.oauth2Login()` 뿐이다. 우리 로컬 로그인은 `@PostMapping("/auth/login")` 평범한 컨트롤러라 ==시큐리티는 그 존재를 모른다.== 등록된 ClientRegistration이 google 하나뿐이라 "선택지가 없으니 직행"한 것이고, 둘 이상이면 선택 페이지를 생성한다. 제멋대로 고른 게 아니라 자기가 아는 유일한 답을 고른 셈이다.
- **왜 REST API에서 치명적인가**: 클라이언트가 주소창이 아니라 `fetch` 다. ==fetch도 302를 자동으로 따라가는데 남의 도메인이라 CORS에 막히고, 프론트는 "Network Error"만 받는다.== 401이었다는 사실조차 모르니 재발급도 로그인 유도도 못 한다.
- **어떻게 없앴나**: `.exceptionHandling()` 에 EntryPoint를 명시하면 기본값을 덮어쓰는 게 아니라 **기본값을 조립할 기회 자체가 사라진다** — 명시값이 있으면 그 조립 로직을 타지 않는다. 구글 로그인은 프론트가 `/oauth2/authorization/google`(permitAll)로 직접 이동시켜 시작하므로 EntryPoint를 거치지 않아 멀쩡하다.
- **연결**: → [[OAuth2]] (state·콜백 흐름), → 아래 Q.어느 필터에서 거부되나

#### Q. 그 거부는 어느 필터에서, 어떻게 일어나나?
- **한줄답**: ==체인 맨 마지막 `AuthorizationFilter` 에서 일어난다.== 앞의 필터들은 아무도 막지 않는다.
- **원리** — 시작 → 과정 → 결과 (토큰 없이 `GET /problems`):
    - [시작] `JwtFilter` — Authorization 헤더가 없으니 아무것도 하지 않고 통과시킨다. 여기서 막지 않는 이유는 `permitAll` 경로도 이 필터를 지나기 때문이다
    - [과정] `AnonymousAuthenticationFilter` — 비어 있는 SecurityContext에 익명 토큰을 채운다
    - [과정] `AuthorizationFilter` — `.anyRequest().authenticated()` 규칙을 검사한다. 익명은 인증된 게 아니므로 거부
    - [결과] `AccessDeniedException` 을 던진다. ==401이든 403이든 이 필터는 항상 이 예외 하나만 던진다== — 사유를 모르기 때문이다
- **인증이 아니라 인가**: 이름이 `AuthorizationFilter` 인 게 핵심이다. "토큰이 없다"를 판정하는 게 아니라 `.authenticated()` 라는 **자격 조건**을 만족하는지 검사한다. `.hasRole("ADMIN")` 과 같은 종류의 검사이고, 인증 상태는 그중 한 조건일 뿐이다.
- **예외가 거슬러 올라가는 이유**: 필터는 줄 서 있는 게 아니라 **중첩된 함수 호출**이다. `ExceptionTranslationFilter` 가 `try { chain.doFilter() } catch { ... }` 로 `AuthorizationFilter` 를 감싸고 있어서, 거기서 던지면 자연히 catch로 올라온다. ==잡으러 가는 게 아니라 자기가 부른 놈이 던진 걸 받는 것==이고, 그래서 `ExceptionTranslationFilter` 는 `AuthorizationFilter` 바로 앞에 있어야만 한다.
- **결과**: 예외가 `ExceptionTranslationFilter` 에서 응답으로 바뀌므로 DispatcherServlet도 컨트롤러도 실행되지 않는다 → 이 노트 첫 Q의 "위치의 문제"가 여기서 다시 확인된다.
- **연결**: → 위 Q.@ExceptionHandler(Throwable)로 넓혀도, → [[ServletFilter]] (체인이 중첩 호출이라는 것)

#### Q. 그럼 401과 403은 어디서 갈리나?
- **한줄답**: ==`ExceptionTranslationFilter` 가 인증 객체가 익명인지 보고 가른다.== 우리가 만든 분기가 아니라 원래부터 있던 것이다.
- **원리** — 시작 → 과정 → 결과:
    - [시작] catch 블록에서 `AuthenticationTrustResolver.isAnonymous(authentication)` 로 판정한다
    - [과정] 익명이면 → `AuthenticationEntryPoint.commence()` → **401**
    - [과정] 익명이 아니면(= `JwtFilter` 가 컨텍스트를 채우는 데 성공한 사람) → `AccessDeniedHandler.handle()` → **403**
    - [결과] 두 슬롯은 원래부터 뚫려 있었다. ==우리가 한 건 분기를 만든 게 아니라 그 안의 구현체를 갈아끼운 것뿐이다==
- **이름이 다른 이유**: 401은 "다시 인증해봐라"고 길을 열어주는 것이라 `commence`(개시)이고, 403은 같은 계정으로 다시 와도 결과가 같아 열어줄 길이 없으니 `handle`(뒤처리)다. HTTP 스펙 그대로 — 401은 재시도 가능(그래서 `WWW-Authenticate` 헤더가 스펙상 필수), 403은 재시도 무의미.
- **용어 주의**: ==401 Unauthorized는 이름과 달리 "미인증"이고, "비인가"는 403이다.== HTTP 스펙의 유명한 작명 실수다.
- **함정**: `AnonymousAuthenticationToken` 도 `isAuthenticated()` 가 `true` 를 반환한다. 시큐리티 내부도 그걸 안 보고 타입으로 판정한다. 컨트롤러에서 `if (auth.isAuthenticated())` 로 로그인 여부를 가리면 익명도 통과한다. 로그인 여부는 `@AuthenticationPrincipal` 이 null인지 보거나, 애초에 SecurityConfig 규칙으로 거르는 게 맞다.
- **현재 상태**: `.anyRequest().authenticated()` 뿐이라 403 경로는 아직 실행되지 않는다. 관리자 API에 `hasRole("ADMIN")` 을 걸면 그때부터 동작한다.
- **실측**: 토큰 없음 / 쓰레기 토큰 / 만료 토큰 / refresh를 access 자리에 — 네 경우 모두 401 JSON이 나가는 것을 curl로 확인했다.
- **연결**: → [[FilterChain]] (필터 체인 구조), → [[TokenAuth]] (만료 시 재발급 분기)

## ③ 용어 카드 (역참조)

> [!quote]- 용어 9개
> - **commence()**: `AuthenticationEntryPoint` 의 유일한 메서드. "인증을 개시하라". REST에선 401 JSON 작성 역할. → Q.commence 이게 뭔데
> - **DispatcherServlet 경계**: @ExceptionHandler가 개입 가능한 안쪽과 필터가 도는 바깥쪽을 가르는 선. → Q.Throwable로 넓혀도
> - **`jakarta.servlet.error.exception`**: ERROR 디스패치 때 원래 예외가 담기는 request attribute. 다시 던져지는 게 아니라 여기 실려서 전달된다. → Q.Throwable로 넓혀도
> - **BasicErrorController**: Spring Boot의 `/error` 기본 핸들러. timestamp/status/error/path JSON을 낸다. → Q.Throwable로 넓혀도
> - **HandlerExceptionResolver**: DispatcherServlet이 예외 처리에 쓰는 컴포넌트. 필터에 주입해 부르면 advice로 가는 우회로가 된다. → Q.ErrorResponse로 전달할 방법
> - **AuthenticationEntryPoint**: 인증 안 된 요청의 최종 응답을 만드는 지점. API 서버에선 401 JSON. → Q.정석이야
> - **AnonymousAuthenticationToken**: 인증 안 된 상태를 null이 아닌 객체로 표현한 것. 401/403 판정의 근거. → Q.401과 403은 어디서 갈리나
> - **AuthenticationTrustResolver**: 인증 객체가 익명인지 판정하는 컴포넌트. `isAuthenticated()` 대신 이걸 쓴다. → Q.401과 403은 어디서 갈리나
> - **인증 vs 인가**: 인증=너 누구냐(JwtFilter) / 인가=그거 해도 되냐(AuthorizationFilter). → Q.어느 필터에서 거부되나

## ④ 내가 틀렸던 것 (오개념 로그)

> [!quote]- 오개념 11건
> | 내가 생각했던 것 | 실제 |
> |---|---|
> | `@RestControllerAdvice` 가 애플리케이션의 모든 예외를 감시한다 | 능동적 감시자가 아니다. DispatcherServlet이 불러줘야 돈다 |
> | `@ExceptionHandler(Exception.class)` 로 넓히면 필터 예외도 잡힌다 | 범위가 아니라 위치의 문제라 `Throwable.class` 도 동일 |
> | `ExceptionTranslationFilter` 가 필터에서 난 예외를 처리해준다 | `AuthenticationException`/`AccessDeniedException` 두 타입만. 게다가 자기보다 뒤에서 난 것만 |
> | `/error` 로 재진입하면 예외가 다시 던져진다 | request attribute로만 전달된다. advice가 반응할 대상이 없다 |
> | 같은 `IllegalArgumentException` 이면 어디서 던지든 결과가 같다 | 컨트롤러·서비스에서 던지면 advice가 정상적으로 잡는다. 위치가 답을 바꾼다 |
> | 필터 예외를 처리하려면 ErrorResponse를 직접 new 해야 한다 | HandlerExceptionResolver로 위임하면 기존 advice를 그대로 재사용한다 |
> | 인증 필터는 전부 통과시키는 게 정석이다 | Spring Security 자체 필터는 "없으면 통과, 틀리면 즉시 거절" 하이브리드 |
> | 구글이 302를 보냈다 | 우리 서버의 EntryPoint가 보냈다. 302는 두 번 일어난다 |
> | 우리가 401/403을 두 갈래로 분리했다 | 스프링이 이미 두 슬롯을 뚫어뒀고 구현체만 갈아끼웠다 |
> | `AuthorizationFilter` 가 401/403을 구분해 던진다 | 항상 `AccessDeniedException` 하나만. 구분은 `ExceptionTranslationFilter` 의 몫 |
> | `isAuthenticated()` 가 true면 로그인된 것 | 익명 토큰도 true다. 타입으로 판정해야 한다 |
