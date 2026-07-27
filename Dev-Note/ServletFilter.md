# 서블릿 필터 (실행 모델 · 응답 작성)

> **주제**: 서블릿 필터 — 체인 흐름 제어와 응답 작성 방식(헤더·쿠키 포함) · 갱신: 2026-07-28 · 상태: 진행중
> **태그**: #서블릿 #필터 #체인 #응답처리 #쿠키

## ① 큰 그림 (지도)

필터 체인은 **자동으로 흐르는 컨베이어벨트가 아니라, 각 필터가 직접 다음 사람을 부르는 릴레이**다. 부르지 않으면 거기서 끝난다.

```
[요청] → Filter1 → JwtFilter → ... → DispatcherServlet → Controller
           │           │
           │           └ chain.doFilter() 호출 O → 다음으로 감
           │             chain.doFilter() 호출 X → 여기서 끝, 되감기 시작
           │
           └ chain.doFilter() 가 리턴된 뒤의 "후처리" 코드는 실행됨

[response 객체의 정체]
컨테이너가 요청 시작 시 1개 생성 → 체인 전체가 같은 객체를 계속 넘겨받음
  → 그래서 "반환(return)"이라는 개념이 없다
  → void 메서드가 그 객체에 직접 써넣는 방식 (Security의 commence 가 이 형태 → [[ExceptionHandling]])
```

```
[response 에 무엇을 쓰는가 — 헤더부와 본문]
addCookie(...)   → Set-Cookie 헤더 등록
sendRedirect(..) → Location 헤더 + 302 등록
getWriter()      → 본문 작성
  ↑ 전부 같은 response 객체의 다른 자리. 응답 전송 시 함께 나간다.
  ↑ 단, "커밋(전송 시작)" 이후엔 헤더를 더 못 얹는다 → 헤더 먼저, 본문 나중
```

필터는 서블릿 레벨이라 `HttpMessageConverter` 가 없다. 컨트롤러처럼 객체를 return 하면 JSON이 되는 구조가 아니라, `response.getWriter()` 에 직접 써야 한다. 필터에서 난 예외를 어떻게 응답으로 바꾸느냐는 → [[ExceptionHandling]]. 이 쿠키·헤더 지식이 실제로 쓰인 맥락은 → [[OAuth2]].

## ② 질문 트리 (본문)

### 2026-07-21

#### Q. save 메서드가 이해가 안 돼. addCookie가 어떻게 브라우저에 저장돼?
> 그때 왜 궁금했나: 자바 메서드 하나 부른 게 어떻게 브라우저 안 쿠키가 되는지가 안 그려졌다.
- **한줄답**: ==객체를 문자열로 바꿔 response에 Set-Cookie로 얹으면, 브라우저가 HTTP 규칙상 자동 저장==한다.
- **원리** — 시작 → 과정 → 결과:
    - [시작] 필터가 만든 객체를 `saveAuthorizationRequest()` 가 받음 (당시 맥락은 OAuth2의 state 저장 → [[OAuth2]])
    - [과정] `SerializationUtils.serialize` + Base64 = 객체를 문자열로 압축
    - [과정] `addCookie(response, ...)` = response에 `Set-Cookie` 헤더 등록
    - [결과] 브라우저가 `Set-Cookie`를 받으면 자동 저장 (우리가 "저장해"라고 명령 안 함)
- **연결**: → 아래 Q(addCookie 인자), → [[OAuth2]] (이 코드가 나온 맥락)

#### Q. addCookie의 인자 4개가 다 저장되는 거야?
- **한줄답**: 아니. ==`response`는 배달부(저장 안 됨), 나머지 3개(이름/값/수명)가 쿠키의 각 부분==이 됨.
- **원리**:
    - `response` = 쿠키 실어보낼 통로 (저장 안 됨)
    - `OAUTH2_AUTH_REQUEST_COOKIE` = 쿠키 이름(키)
    - `value` = 내용물 (객체가 문자열로 압축돼 들어감)
    - `180` = 수명(Max-Age)
- **결과물**: `Set-Cookie: oauth2_auth_request=...; Max-Age=180; HttpOnly` 헤더 한 줄
- **연결**: → 아래 Q(header에 담기)

#### Q. addCookie가 response 객체 안에 header로 쿠키를 담는 거지?
- **한줄답**: 맞음. `response`에 `Set-Cookie` 헤더 형태로 등록하는 것.
- **원리** — 시작 → 과정 → 결과:
    - [시작] `addCookie(response, ...)` 호출 = Cookie 객체를 Set-Cookie 헤더 문자열로 변환
    - [과정] `response.addCookie()` 가 그 헤더를 응답 객체에 얹음 (호출 직후엔 response 안에만 있음)
    - [결과] 응답이 실제 전송될 때 네트워크로 나감
- **왜 순서 중요**: ==`sendRedirect`(응답 커밋)보다 먼저 addCookie 해야 함==. 커밋 후엔 헤더 추가 불가.
- **연결**: → 아래 Q(sendRedirect 헤더), → 아래 2026-07-26 Q(response.isCommitted)

#### Q. sendRedirect 할 때 쿠키를 header에 담아 보내는 거야?
- **한줄답**: 쿠키는 `Set-Cookie` 응답 헤더에 실려 나간다. 단, ==sendRedirect가 담는 게 아니라 이미 등록된 게 같이 나가는 것==.
- **원리**:
    - `addCookie` = `Set-Cookie` 헤더 등록 / `sendRedirect` = `Location` 헤더 + 302 등록
    - 둘은 같은 response의 다른 헤더 → 응답 전송 시 함께 나감
- **헷갈림 주의**: `Set-Cookie`(서버→브라우저 "저장해") vs `Cookie`(브라우저→서버 "가져왔어") = 방향 반대.
- **연결**: → [[OAuth2]] Q.state를 언제 구글에 보냈어

---

### 2026-07-26

#### Q. commence에서 response객체를 만들었잖아? 이게 이 필터에서 바로 반환되는 건가, 아니면 나머지 필터까지 다 돌아야 되는 건가?
- **한줄답**: ==해당 필터에서 바로 끝난다. 나머지 필터를 돌지 않는다.== 그리고 commence는 response를 "만드는" 게 아니다.
- **원리** — 시작 → 과정 → 결과:
    - [시작] response는 요청이 처음 들어올 때 **서블릿 컨테이너가 이미 만들어** 필터 체인 전체에 같은 객체를 넘겨주고 있다. commence는 그 객체에 상태 코드와 본문을 써넣을 뿐이다
    - [과정] ==필터 체인은 chain.doFilter()를 호출해야만 다음으로 넘어간다.== 자동으로 흘러가지 않는다
    - [과정] catch 블록에서 commence만 호출하고 `chain.doFilter()` 를 안 부르면, 부르는 코드가 없으니 뒤로 안 간다
    - [과정] 메서드가 그냥 끝나고 스택을 되감아 컨테이너까지 올라간 뒤, 그 시점의 response 상태 그대로 클라이언트에게 전송된다
    - [결과] `ExceptionTranslationFilter`, `AuthorizationFilter`, `DispatcherServlet`, 컨트롤러 — 전부 실행되지 않는다
- **왜 이렇게**: 반환 타입이 void라 애초에 "반환"이라는 개념이 없다. 넘겨받은 객체를 직접 수정하는 방식이라서 그렇다.
- **주의(되감기)**: 뒤(downstream)로는 안 가지만 자기보다 앞에 있던 필터들의 `chain.doFilter()` 가 리턴되는 지점은 지나간다. 앞 필터의 "후처리" 코드 — 로깅 필터의 응답시간 측정, CORS 필터의 헤더 부착 — 는 정상 동작한다. 응답이 중간에 증발하는 게 아니라 완성된 채로 되감겨 나간다.
- **주의(재호출 금지)**: commence 이후에 실수로 `chain.doFilter()` 를 또 부르면 뒤 필터나 컨트롤러가 같은 response에 또 쓰려다 `IllegalStateException`(getWriter 중복 호출)이 나거나 JSON 두 개가 이어붙은 깨진 응답이 나간다. ==commence 이후엔 그 요청에 대해 아무것도 하지 않는다== — 이것만 지키면 된다. `response.isCommitted()` 로 이미 전송이 시작됐는지 방어적으로 확인할 수 있다.
- **연결**: → [[ExceptionHandling]] (체인 중단이 왜 유일한 선택지인가)

## ③ 용어 카드 (역참조)

> [!quote]- 용어 5개
> - **Set-Cookie / Cookie**: 전자=서버→브라우저(저장해, 응답헤더), 후자=브라우저→서버(가져왔어, 요청헤더). → Q.sendRedirect
> - **직렬화/역직렬화**: 객체↔문자열 변환(+Base64). 쿠키는 문자열만 담아서 필요. → Q.save 이해
> - **chain.doFilter()**: 다음 필터로 넘기는 호출. 이걸 안 부르면 체인은 거기서 끝난다. → Q.바로 반환되는 건가
> - **되감기(unwinding)**: 체인 중단 후 앞 필터들의 후처리 코드를 거쳐 컨테이너까지 올라가는 경로. → Q.바로 반환되는 건가
> - **response.isCommitted()**: 응답 전송이 이미 시작됐는지 확인. 중복 작성 방어용. → Q.바로 반환되는 건가

## ④ 내가 틀렸던 것 (오개념 로그)

> [!quote]- 오개념 6건
> | 내가 생각했던 것 | 실제 |
> |---|---|
> | addCookie의 response도 쿠키에 저장된다 | response는 배달부. 이름/값/수명만 쿠키가 됨 |
> | sendRedirect가 쿠키를 담아 보낸다 | addCookie가 미리 Set-Cookie 등록, redirect 응답에 같이 실림 |
> | commence가 response 객체를 만든다 | 컨테이너가 만들어 체인 전체가 공유하는 객체다. commence는 거기에 써넣을 뿐 |
> | 필터 체인은 자동으로 다음으로 흐른다 | `chain.doFilter()` 를 호출해야만 넘어간다 |
> | 응답을 만들었으면 나머지 필터도 다 돈다 | 체인 중단 시 뒤로는 안 간다. 앞 필터의 후처리(되감기)만 지난다 |
> | 필터에서도 컨트롤러처럼 return으로 응답을 만들 수 있다 | 서블릿 레벨이라 `HttpMessageConverter` 가 없다. `response.getWriter()` 에 직접 써야 한다 |
