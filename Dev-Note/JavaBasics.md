# Java 기초 (언어/문법)

> **주제**: 자바 언어 자체의 기능·문법 · 갱신: 2026-07-26 · 상태: 진행중
> **태그**: #Java #자바기초 #언어기능 #예외처리

## ① 큰 그림 (지도)

자바 언어 기능들이 프레임워크의 "마법"을 실제로 가능하게 한다. 대표가 **리플렉션**.

```
내 코드(엔티티/컨트롤러)  ←── 프레임워크가 리플렉션으로 들여다보고 호출 ──  Hibernate / Spring
                                (어노테이션 스캔 → private 메서드까지 동적 호출)
```

- 보통 호출: **컴파일 때** 이름을 코드에 박아 부른다 (`obj.method()`).
- 리플렉션: **런타임에** 클래스 구조를 조회해 동적으로 부른다 → 프레임워크가 "남의 코드(내 클래스)"를 이름도 모른 채 다룰 수 있는 이유.

또 하나의 언어 기능 **enum**: 상수를 단순 값이 아니라 **객체**로 만들어 타입 안전 + 데이터·행동을 부여한다. (자세히는 아래 2026-07-25 Q)

**예외**도 언어 특성(checked/unchecked, 스택 되감기)이 Spring 전역 예외 처리(`@RestControllerAdvice`)를 떠받친다. 예외는 잡는 곳을 만날 때까지 콜 스택을 거꾸로 타고 올라가고, 아무도 안 잡으면 최상단 DispatcherServlet → advice가 받는다.

```
Service.throw ─┐ (아무도 catch 안 함)
Controller     │  ← 통과만, 안 잡음 (호출줄 아래 코드 실행 안 됨)
DispatcherServlet ← 받음 → HandlerExceptionResolver → @RestControllerAdvice → ErrorResponse(JSON)
```

그런데 이 그물에는 두 가지 구멍이 있고, 둘 다 언어 특성에서 나온다.

```
[구멍 1 — 타입 계층]  catch 는 "부모-자식"만 잡는다. 형제는 못 잡는다.
  RuntimeException
    ├── io.jsonwebtoken.JwtException      ← catch (JwtException) 이 잡는 범위
    └── java.lang.IllegalArgumentException ← 형제라 안 잡힘

[구멍 2 — unchecked]  컴파일러가 강제 검사하지 않으므로 "안 잡았다"는 경고가 없다.
  → 방어선은 컴파일러가 아니라 API 시그니처의 throws 절을 읽는 습관뿐
  → 시그니처: parseSignedClaims(CharSequence) throws JwtException, IllegalArgumentException
```

그리고 이 예외가 **어디서** 터지느냐에 따라 advice가 받느냐가 갈린다 — 필터에서 터지면 DispatcherServlet에 도달하지 못해 위 그림 자체가 성립하지 않는다 → [[ExceptionHandling]].

## ② 질문 트리 (본문)

### 2026-07-20

#### Q. `return ResponseEntity.ok(authService.login(req))` 면 service로 안 넘어가는 거 아냐?
- **한줄답**: 넘어간다. ==안쪽 `authService.login(req)` 이 먼저 실행되고 그 결과를 감싼다==.
- **원리**: 자바는 **메서드를 호출하기 전에 인자를 먼저 평가**한다 → `login()` 실행 → 반환값을 `ResponseEntity.ok(결과)` 에 전달. "감싸는 코드가 바깥에 있으니 먼저 실행된다"는 직관이 틀린 것이고, 실행 순서는 안쪽부터다.
- **주의**: 단, `authService` 가 주입되려면 생성자(또는 `@RequiredArgsConstructor`)가 있어야 한다(final 필드).
- **연결**: → [[Authentication]] (이 코드가 나온 맥락 — 로그인 방식 A)

---

### 2026-07-23

#### Q. (@PrePersist 설명 중) 여기서 말하는 리플렉션이 뭐야?
- **한줄답**: 실행 중(runtime)에 클래스/메서드/필드/어노테이션을 **조회·조작**하는 자바 언어 기능. 보통 호출은 컴파일 때 이름을 박지만, ==리플렉션은 런타임에 동적으로 발견해 호출한다==.
- **원리**: `clazz.getDeclaredMethods()`로 메서드 목록을 얻고 → `m.isAnnotationPresent(PrePersist.class)`로 어노테이션 확인 → `m.setAccessible(true)`로 private을 열고 → `m.invoke(obj)`로 동적 호출. Hibernate가 `@PrePersist` 메서드를 **이름 무관하게** 찾아 부르는 게 이 방식이고, Spring의 `@Autowired` 주입·`@PostMapping` 매핑도 리플렉션 기반이다. → **"프레임워크가 내 코드를 호출한다(IoC)"** 를 기술적으로 가능케 하는 토대.
- **연결**: 이 질문이 나온 맥락 → [[Persistence]] 의 `@PrePersist`/`@PreUpdate` 콜백. 단점(성능/컴파일체크 우회)은 → 아래 ④.

---

### 2026-07-25

#### Q. enum 상수는 "이름표"가 아니라 "객체"라던데 맞아? 필드 없는 enum(Color)도 객체야?
- **한줄답**: 맞다. ==enum 상수는 enum 타입의 싱글턴 인스턴스(객체)==다. `RED`/`GREEN`/`BLUE`처럼 필드가 없어도 마찬가지 — ==필드가 없을 뿐 온전한 객체==다.
- **원리**: 컴파일러가 각 상수를 `public static final Color RED = new Color("RED", 0);` 처럼 **private 생성자로 만든 `static final` 인스턴스**로 바꾸고, static 초기화 블록에서 **선언 순서대로** 생성한다(그래서 `ordinal`이 0,1,2…). 생성자가 항상 private이라 외부 `new`가 불가능 → 선언된 상수 개수만큼만 존재하는 싱글턴이 보장되고 `==` 비교가 성립한다. 필드 있는 enum(`ErrorCode(HttpStatus, message)`)은 생성자 인자를 받아 **데이터를 품은 객체**, 필드 없는 enum(`Color`)은 인자 없는 **데이터 없는 객체**일 뿐 구조는 동일. 필드가 없어도 `name()`/`ordinal()`/`values()`/`valueOf()`는 부모 `Enum`에서 자동으로 갖는다. (예외: 상수별 본문 `RED { ... }`이 있으면 그 enum은 abstract가 되고 상수는 익명 하위클래스 인스턴스가 됨.)
- **연결**: `extends Enum`은 컴파일러만 붙인다(내 소스에 직접 쓰면 컴파일 에러 → 다른 클래스 상속 불가, 인터페이스 구현만 가능). `static final int` 상수와의 차이(타입 안전) → 아래 ④. 이 개념이 나온 맥락 → 전역 예외 핸들러의 `ErrorCode` enum 설계.

#### Q. service에서 예외를 던지면 controller를 거쳐 @RestControllerAdvice가 가로채는 흐름이 어떻게 되나?
- **한줄답**: ==예외는 컨트롤러가 잡는 게 아니라 "통과"하고, DispatcherServlet까지 올라가 @RestControllerAdvice가 받는다==.
- **원리** — 시작 → 과정 → 결과:
    - [시작] 콜 스택은 DispatcherServlet→Controller→Service 순으로 쌓임 → Service가 throw하면 그 줄에서 즉시 중단
    - [과정] 잡는 곳 찾아 스택 되감기 → Controller에 try-catch 없으니 ==통과(호출줄 아래 실행 안 됨)== → DispatcherServlet 도달
    - [결과] DispatcherServlet이 HandlerExceptionResolver에 위임 → `@ExceptionHandler(CustomException)` 매칭 → `ResponseEntity.status(code) + ErrorResponse` → `@RestControllerAdvice(=@ControllerAdvice+@ResponseBody)`라 JSON 직렬화
- **4조각**: `ErrorCode`(enum, 상태+메시지 카탈로그) / `CustomException`(ErrorCode 운반) / `ErrorResponse`(응답 DTO) / `GlobalExceptionHandler`(그물)
- **연결**: CustomException이 왜 RuntimeException인지 → 아래 Q. ErrorCode가 필드를 품는 원리 → 위 enum Q. 리졸버가 advice를 찾는 상세 → 바로 아래 Q.

#### Q. (심화) DispatcherServlet 위임과 @ExceptionHandler 매칭 사이 — 리졸버는 정확히 무엇을 하나?
- **한줄답**: ==HandlerExceptionResolver 체인 중 `ExceptionHandlerExceptionResolver`가 @RestControllerAdvice · 컨트롤러의 @ExceptionHandler들을 뒤져 던져진 예외 타입에 매핑되는 메서드를 찾는다==(가장 구체적 타입 우선).
- **원리** — 시작 → 과정 → 결과:
    - [시작] DispatcherServlet은 여러 개의 `HandlerExceptionResolver`를 갖고, 예외가 나면 앞에서부터 하나씩 "네가 처리할 수 있냐"고 물어봄
    - [과정] 후보 **3종** — 
	    - **`ExceptionHandlerExceptionResolver`** : (@ExceptionHandler 담당) 
		- **`ResponseStatusExceptionResolver`** : (@ResponseStatus·ResponseStatusException) 
		- **`DefaultHandlerExceptionResolver`** (스프링 표준 예외) → CustomException은 첫 번째가 처리
    - [과정] 첫 리졸버가 탐색: ==(a) 예외 난 컨트롤러 자신의 @ExceptionHandler 먼저 → (b) 없으면 @ControllerAdvice/@RestControllerAdvice==. 여러 개 걸리면 가장 구체적 타입 우선(`CustomException`용이 `RuntimeException`용을 이김)
    - [결과] 찾은 메서드에 예외 객체 `e`를 실인자로 넘겨 호출 → 반환 `ResponseEntity`가 실제 응답 (`ErrorResponse`는 static 팩토리로 `new` 되는 일반 DTO — Bean 아님)
- **연결**: 이 단계의 큰 그림 → 위 예외 흐름 Q.

#### Q. CustomException은 왜 RuntimeException(unchecked)을 상속하나? checked면 어떻게 되나?
- **한줄답**: ==unchecked라야 throws 오염 없이 조용히 전파되고, @Transactional이 자동 롤백한다==.
- **원리**: checked(=`Exception` 상속)면 컴파일러가 "잡거나 throws 선언"을 강제 → Service·Controller 시그니처에 `throws`가 줄줄이 붙고 안 하면 컴파일 에러. 게다가 @Transactional 기본 롤백은 ==RuntimeException/Error만==, checked는 커밋돼버림(정합성 깨짐). checked/unchecked는 "컴파일러가 처리를 강제 검사하냐"의 구분이지 "언제 처리하냐"가 아님(둘 다 런타임 발생). 기준: checked=복구해야 할 외부 실패(IOException), unchecked=고쳐야 할 버그(NPE).
- **연결**: enum이 필드를 갖는 원리(ErrorCode가 HttpStatus·메시지를 품음) → 위 enum Q. checked를 실제로 어떻게 처리하는지 → 아래 Q.

#### Q. checked 예외를 만나면 던지는 메서드부터 throws를 붙여야 하나? 아니면 잡아서 다시 던지나(RuntimeException으로)?
- **한줄답**: 두 갈래 — ==throws로 위로 넘기거나(오염 전파), try-catch로 잡아 RuntimeException으로 감싸 재던진다(오염 차단)==. 후자가 예외 변환(translation) 패턴.
- **원리** — 시작 → 과정 → 결과:
    - [시작] checked를 `throw`하는 순간 그 메서드부터 `throws` 필수
    - [과정] 안 잡으면 호출자로 `throws`가 계속 번짐(오염)
    - [과정] 오염을 끊는 표준 = **래핑**: 남의 checked(`IOException`/`SQLException`)를 `catch`해 우리 `CustomException`(unchecked)으로 감싸 재던짐. 이때 ==원본을 cause로 넘겨야(`super(msg, cause)`) 스택트레이스 보존==(안 넘기면 "왜 터졌나" 유실)
    - [결과] 밖에서 unchecked만 보이니 `throws` 사라짐 (삼키기=잡고 아무것도 안 하기는 금지 / Spring도 `SQLException`→`DataAccessException` 동일 변환)
- **연결**: 왜 unchecked로 통일하나 → 위 RuntimeException Q. throw/throws 강제 메커니즘 → 아래 Q.

#### Q. throw와 throws는 뭐가 다르고, checked 예외를 호출한 메서드는 왜 throws나 try-catch 중 하나를 강제당하나?
- **한줄답**: ==`throw`는 실제로 던지는 행동, `throws`는 "이 예외가 밖으로 나갈 수 있다"고 호출자에게 알리는 선언==. checked면 호출한 메서드는 ==잡거나(try-catch) 통과 선언(throws) 중 하나 필수 — 둘 다 안 하면 컴파일 에러==.
- **원리**: 호출자는 메서드 본문을 못 보니, 위험성을 시그니처(`throws`)에 미리 선언해야 컴파일러가 호출부에서 처리를 강제할 수 있다. 그래서 `throws`만 있고 `throw`는 없는 메서드도 존재한다 — 남(예: `read()`)이 던진 걸 안 잡고 통과(중계)만 시킬 때. 반대로 호출 대상이 unchecked면 강제 자체가 없어 둘 다 없어도 컴파일된다. (에러 메시지: `unreported exception ...; must be caught or declared to be thrown`)
- **연결**: checked를 실제로 처리하는 법(잡아 unchecked로 래핑) → 위 checked 처리 Q.

#### Q. 커스텀 예외를 만들 때 RuntimeException/Exception 중 하나를 꼭 상속해야 하나?
- **한줄답**: ==그렇다. `throw` 가능한 건 `Throwable` 자손뿐이라 `Exception`(checked)이나 `RuntimeException`(unchecked) 중 하나는 반드시 상속==해야 한다. 비즈니스 예외는 `RuntimeException`을 고른다.
- **원리**: 아무것도 상속 안 한 클래스는 `throw` 자체가 컴파일 에러(`Throwable`이 아니라서). 현실 선택지는 `Exception`(checked) vs `RuntimeException`(unchecked)이고, `Error`는 JVM용(OutOfMemory 등)이라 상속하지 않는다. 비즈니스 예외를 `RuntimeException`으로 두는 이유는 throws 오염 방지 + @Transactional 자동 롤백.
- **연결**: 왜 unchecked를 고르나 → 위 RuntimeException Q.

---

### 2026-07-26

#### Q. isValid(token) 여기서 token에 빈 문자열이 들어가면 IllegalArgumentException을 던진다는 거야?
- **한줄답**: 그렇다. 그리고 ==`catch (JwtException)` 으로는 안 잡힌다== — 두 예외는 부모-자식이 아니라 형제이기 때문이다.
- **원리** — 시작 → 과정 → 결과:
    - [시작] jjwt의 `parseSignedClaims` 는 토큰 형식을 뜯어보기 **전에** 인자 자체가 말이 되는지 확인한다. 내부는 대략 `if (!Strings.hasText(jwt)) throw new IllegalArgumentException("JWT string cannot be null or empty.")`
    - [과정] 그래서 ==“잘못된 JWT”가 아니라 “잘못된 메서드 호출”로 취급==된다 → `JwtException` 이 아니라 `IllegalArgumentException`
    - [과정] API 시그니처가 이미 두 예외를 나란히 선언하고 있었다: `Jws<Claims> parseSignedClaims(CharSequence jws) throws JwtException, IllegalArgumentException;`
    - [과정] 클래스 계층을 보면 `RuntimeException` 아래에 `io.jsonwebtoken.JwtException` 과 `java.lang.IllegalArgumentException` 이 **나란히** 있다. 부모-자식이 아니라 형제라 `catch (JwtException)` 이 못 잡는다
    - [결과] unchecked(RuntimeException)라 컴파일러도 경고하지 않는다. 조용히 호출자에게 전파된다
- **왜 이렇게(checked였다면)**: checked였다면 컴파일러가 "잡거나 throws 선언하라"고 강제해서 존재를 모를 수가 없다. unchecked의 정의가 "컴파일러가 강제하지 않는다"이므로, 방어선은 컴파일러가 아니라 **API 시그니처의 throws 절을 읽는 습관**뿐이다.
- **주의(어디서 막나)**: 두 겹으로 막는다. **사전 차단** `if (!StringUtils.hasText(token)) return false;` 는 "빈 토큰은 검증 실패"라는 규칙을 코드로 선언하는 의도 표현이고, **사후 포획** `catch (JwtException | IllegalArgumentException e)` 는 안전망이다. ==`isValid` 는 boolean을 반환하는 술어(predicate) 메서드다== — 어떤 입력에도 예외 없이 true/false만 반환해야 호출부가 `if (isValid(t))` 로 안심하고 쓴다. 사전 차단만 하면 계약이 "빈 문자열에 한해서만" 지켜진다.
- **연결**: → [[ExceptionHandling]] (이 예외가 필터에서 터졌을 때), → [[TroubleShooting]], → [[TokenAuth]]

## ③ 용어 카드 (역참조)

> [!quote]- 용어 20개
> - **템플릿 메서드 패턴**: 공통 흐름은 부모가 완성해두고, 가변 부분만 `abstract` 로 비워 자식이 채우는 구조. → [[Authentication]] Q.doFilter는 오버라이드 안 하나
> - **동적 디스패치(다형성)**: 인터페이스 타입 필드를 호출하면 실제로 담긴 객체의 메서드가 실행되는 것. 커스텀 핸들러가 실행되는 원리. → [[Authentication]] Q.successHandler 실행 원리
> - **리플렉션(Reflection)**: 런타임에 클래스 구조(메서드·필드·어노테이션)를 조회·조작하는 기능. `java.lang.reflect`. → Q(리플렉션?)
> - **setAccessible(true)**: 리플렉션으로 `private` 멤버에 접근 허용. 그래서 콜백을 private으로 둬도 호출됨. → Q(리플렉션?)
> - **IoC(제어의 역전)**: 내가 프레임워크를 부르는 게 아니라, 프레임워크가 내 코드를 부르는 구조. 리플렉션이 그 기술적 토대. → Q(리플렉션?)
> - **어노테이션 스캔**: `isAnnotationPresent`로 특정 어노테이션 붙은 요소를 찾는 것. 메서드 "이름"이 아니라 "명찰"로 찾기. → Q(리플렉션?)
> - **enum 상수**: enum 타입의 미리 정해진 싱글턴 인스턴스. private 생성자로 static 초기화 블록에서 생성됨. → Q(enum 상수=객체?)
> - **ordinal**: enum 상수의 선언 순서 인덱스(0부터). 부모 `Enum`이 자동 부여. → Q(enum 상수=객체?)
> - **상수별 본문(constant-specific body)**: 상수마다 메서드를 오버라이드(`RED { ... }`)하면 그 상수는 익명 하위클래스 인스턴스가 되고 enum은 abstract가 됨. → Q(enum 상수=객체?)
> - **@RestControllerAdvice**: 모든 컨트롤러 밖에서 예외를 전역으로 잡는 그물. `=@ControllerAdvice+@ResponseBody`. → Q(예외 흐름?)
> - **@ExceptionHandler**: advice 안에서 특정 예외 타입을 맡는 메서드. → Q(예외 흐름?)
> - **HandlerExceptionResolver**: DispatcherServlet이 예외를 위임하는 해결기 체인. → Q(예외 흐름?)
> - **ExceptionHandlerExceptionResolver**: 그 체인 중 @ExceptionHandler/@RestControllerAdvice를 뒤져 매핑 메서드를 찾는 리졸버(가장 구체적 타입 우선). → Q(리졸버 심화?)
> - **checked 예외**: 컴파일러가 처리/선언을 강제 검사하는 예외(`Exception` 상속). → Q(RuntimeException?)
> - **unchecked 예외**: 컴파일러가 강제 안 하는 예외(`RuntimeException` 상속). 강제하지 않으므로 존재를 알려면 API 시그니처의 `throws` 절을 직접 읽어야 한다. → Q(RuntimeException?) · Q(빈 문자열이 들어가면?)
> - **예외 변환/래핑(translation)**: checked를 잡아 unchecked로 감싸 재던져 throws 오염을 끊는 패턴. Spring도 SQLException→DataAccessException으로 함. → Q(checked 처리?)
> - **cause 체이닝**: `super(msg, cause)`로 원본 예외를 연결해 스택트레이스("Caused by:") 보존. 래핑 시 필수. → Q(checked 처리?)
> - **throw vs throws**: `throw`=실제 던지는 행동, `throws`=호출자에게 "이 예외 나갈 수 있음"을 알리는 시그니처 선언(던지는 게 아님). → Q(throw vs throws?)
> - **술어(predicate) 메서드 계약**: boolean을 반환하는 메서드는 어떤 입력에도 예외 없이 값만 반환해야 한다. → Q(빈 문자열이 들어가면?)
> - **guard clause(사전 차단)**: 본 로직 전에 비정상 입력을 걸러 조기 반환하는 패턴. → Q(빈 문자열이 들어가면?)

## ④ 내가 틀렸던 것 (오개념 로그)

> [!quote]- 오개념 16건
> | 내가 생각했던 것 | 실제 |
> |---|---|
> | 필드 접근에 `this` 안 붙이는 게 맞다 | 필드면 `this`가 맞다. 지역변수일 때만 `this` 없이 쓴다 |
> | 리플렉션은 무슨 특별한 라이브러리다 | 자바 **언어 자체 기능**(`java.lang.reflect`). 프레임워크가 활용할 뿐 |
> | `@PrePersist` 메서드는 이름이 정해져 있다 | 이름 무관. 리플렉션이 **어노테이션으로** 찾음. private이어도 `setAccessible`로 호출 |
> | 리플렉션은 공짜다 | 직접 호출보다 느림(그래서 스캔 결과를 캐싱), 컴파일 타임 타입 체크를 우회 → 런타임에야 에러가 드러남 |
> | enum 상수는 문자열/정수 같은 이름표(별칭)다 | enum **타입의 객체(인스턴스)**다. 그래서 필드·메서드를 가질 수 있음 |
> | 필드 없는 enum(`Color`)은 그냥 상수값이라 객체가 아니다 | 데이터만 없을 뿐 **온전한 객체**. `name()`/`ordinal()` 등을 이미 가짐 |
> | `RED`/`GREEN`/`BLUE`는 하나의 `Color` 객체다 | **서로 다른 3개**의 싱글턴 인스턴스 |
> | 컨트롤러가 예외를 받아서 처리한다 | 컨트롤러는 통과만. DispatcherServlet→advice가 받음 |
> | checked/unchecked는 "언제 처리하냐"의 차이 | "컴파일러가 강제 검사하냐"의 차이(둘 다 런타임 발생) |
> | 아무 예외나 던지면 @Transactional이 롤백된다 | RuntimeException/Error만 기본 롤백. checked는 커밋됨 |
> | checked는 throws로 넘기는 수밖에 없다 | catch해서 unchecked로 감싸 재던지면(cause 보존) 오염이 끊긴다 |
> | `throws Exception`은 예외를 던지는 것이다(throw와 같은 일) | `throws`는 선언(경고판)일 뿐 안 던짐. 실제 발사는 `throw`만 |
> | 아무 클래스나 만들어 throw할 수 있다 | `Throwable` 자손만 throw 가능. 커스텀 예외는 반드시 Exception/RuntimeException 상속 |
> | `catch (JwtException)` 이면 JWT 파싱 관련 예외는 다 잡힌다 | `IllegalArgumentException` 은 자식이 아니라 **형제**(둘 다 `RuntimeException` 직계)라 안 잡힌다 |
> | 안 잡은 unchecked 예외는 컴파일러가 알려준다 | 안 알려준다. 그게 unchecked의 정의다 |
> | 사전 차단만 해두면 충분하다 | 빈 문자열만 막힌다. 술어 메서드의 계약을 지키려면 사후 포획도 필요하다 |
