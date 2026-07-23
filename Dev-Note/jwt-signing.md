# JWT 서명 원리 (HMAC-SHA256)

> **주제**: JWT 서명 원리 · 갱신: 2026-07-20 · 상태: 진행중
> **태그**: #JWT #서명 #HMAC #암호화 #보안

## ① 큰 그림 (지도)

```
[발급]
secret 문자열 --getBytes(UTF-8)--> 바이트 키 --hmacShaKeyFor--> SecretKey (준비만)
Jwts.builder().claim(...).signWith(SecretKey).compact()  ← 여기서 실제 HMAC-SHA256 서명
토큰 = header . payload . signature
                 (Base64,읽힘)  (256비트 지문)

[검증]
같은 SecretKey로 header+payload를 재해싱 → 토큰의 signature와 대조 (같으면 유효)
```

- **대칭키**: 서명과 검증에 같은 secretKey. 그래서 secretKey는 서버 안에만(유출되면 위조 토큰 발급 가능).
- 비유: HMAC-SHA256 = 비밀 레시피(키)를 섞어 토큰 내용의 지문을 찍는 오븐. SHA는 단방향(복호화 불가).

## ② 질문 트리 (본문)

### 2026-07-20

#### Q. Keys.hmacShaKeyFor(secret.getBytes(UTF_8))는 어떻게 동작하나? hmacShaKeyFor가 뭐야?
- **한줄답**: 문자열 시크릿을 HMAC-SHA용 SecretKey 객체로 "준비"하는 것. 여기서 해싱/서명은 안 한다.
- **원리**: `getBytes`로 바이트화 → `hmacShaKeyFor`가 SecretKey 생성(+길이 검증 +알고리즘 선택). "For" = HMAC-SHA에 쓸 키를 만들라는 뜻.
- **연결**: → 아래 Q(서명은 signWith)

#### Q. getBytes(UTF-8)는 인코딩인가? 32바이트가 256비트가 되는 거야?
- **한줄답**: 맞다. `getBytes(UTF-8)`는 문자→바이트 인코딩. 32바이트 = 256비트는 인코딩이 아니라 단위 환산(1byte=8bit).
- **원리**: 암호 연산은 바이트 단위 → UTF-8로 문자를 바이트로 표현(ASCII 1글자=1바이트). 그 바이트가 곧 HMAC의 비밀키. "바이트→비트 인코딩" 같은 건 없고, 32바이트를 비트로 세면 256비트일 뿐.
- **연결**: → 아래 Q(예시 키)

#### Q. Q}a1L80b/... 같은 키가 어떻게 256비트가 되는 거야?
- **한줄답**: 그 키는 43글자=43바이트=344비트. 256비트가 "되는" 게 아니라 256비트 최소치를 넘긴 것.
- **원리**: 256비트는 고정 크기가 아니라 HS256의 **하한**(32바이트). 이상이면 다 유효. 주의: 글자수=바이트수는 ASCII만 성립(한글 3바이트, 이모지 4바이트).
- **연결**: → 아래 Q(내 코드 HS256?)

#### Q. 같은 secretKey로 서명하고 검증도 이걸로? 그래서 대칭키야?
- **한줄답**: 맞다. 서명·검증에 같은 키 → 대칭키. 검증은 "같은 키로 서명을 다시 계산해 대조".
- **원리**: 검증기가 header+payload를 secretKey로 재해싱해 토큰의 signature와 비교. 같으면 유효(위조 없음), 다르면 예외. 키를 모르면 유효한 서명을 못 만들어 위조 불가. 대칭키라 secretKey는 절대 외부 노출 금지.
- **연결**: → 아래 Q(HMAC vs SHA)

#### Q. hmac-sha는 sha와 다른 거네? sha256이 뭘 하는 거고 hmac-sha256은 뭐지?
- **한줄답**: SHA-256은 단방향 해시(임의 입력 → 256비트 지문). HMAC-SHA256은 거기에 비밀키를 섞은 버전.
- **원리**: SHA는 암호화(복호화 가능)가 아니라 되돌릴 수 없는 해싱. 그냥 SHA는 누구나 계산 가능(키 없음) → 위조 가능. HMAC은 키를 섞어(SHA를 키와 함께 2번) "키 아는 사람만 만들 수 있는 해시"로 만든 것.
- **연결**: → 아래 Q(무엇이 해싱되나)

#### Q. SHA256이 secretKey를 256비트로 해싱한다는 거야?
- **한줄답**: 아니다. 해싱 대상은 secretKey가 아니라 "토큰 내용(header+payload)". secretKey는 거기 섞는 재료.
- **원리**: SHA-256 입력은 1개(메시지)지만 HMAC은 입력 2개(키+메시지). 키를 256비트로 바꾸는 게 아니라, 키로 메시지를 해싱한 결과(서명)가 256비트. 256이 되는 건 서명이지 키가 아니다.
- **연결**: → 아래 Q(해싱되는 내용)

#### Q. 256비트로 해싱되는 내용은 뭐야? header+sign을 합쳐서 256? payload는 그대로 둔다는 게 무슨 말?
- **한줄답**: 해싱 입력 = header+payload, 결과 = signature(256비트). payload 원본은 그대로 남고 해싱은 별도 지문을 만들 뿐(원본 소비 안 함).
- **원리**: 토큰은 `header.payload.signature` 3조각. signature = HMAC-SHA256(secretKey, header+payload) → 세 번째 조각. payload는 Base64로 읽히게 그대로 저장되고, 그걸 해싱한 256비트 지문이 옆에 붙는다(복사기처럼 원본 유지). 토큰 전체가 256비트인 게 아니라 signature만 256비트. header를 줄여 256 맞추는 일도 없다.
- **연결**: → [[spring-security-auth-flow]] (claim에 민감정보 금지 — payload는 누구나 읽음)

#### Q. hmacShaKeyFor에서 이미 hmac-sha를 한 거 아냐? 정의만 한 거야?
- **한줄답**: 정의(준비)만. 실제 HMAC-SHA 서명은 `signWith(secretKey)` + `compact()`에서 일어난다.
- **원리**: 생성자의 hmacShaKeyFor는 열쇠를 깎아 필드에 저장(앱 시작 시 1회). 토큰 만들 때마다 `signWith`가 header+payload를 그 열쇠로 실제 해싱해 signature 생성. 검증 시엔 `verifyWith(secretKey)`가 재해싱.
- **연결**: → 위 Q(서명 대상)

#### Q. 그러면 지금 내 코드는 HMAC-SHA256인 거야?
- **한줄답**: 맞다. `hmacShaKeyFor`+`signWith` 조합은 HMAC-SHA. 키가 32~47바이트면 HS256.
- **원리**: jjwt가 키 길이로 알고리즘 자동 선택(32~47B→HS256, 48~63B→HS384, 64B~→HS512). 확인은 토큰 header 디코드 → `{"alg":"HS256"}`. 키 32바이트 미만이면 WeakKeyException.
- **연결**: → 위 Q(예시 키 344비트)

## ③ 용어 카드 (역참조)

> [!quote]- 용어 6개
> - **HMAC-SHA256**: 비밀키를 섞어 header+payload의 256비트 서명을 만드는 대칭키 방식. → Q.HMAC vs SHA
> - **SHA-256**: 단방향 해시(임의 입력→256비트). 암호화 아님(복호화 불가). → Q.sha256이 뭘
> - **hmacShaKeyFor**: 바이트 키로 SecretKey를 준비하는 유틸(해싱 아님, 길이 검증+알고리즘 선택). → Q.동작
> - **signWith / verifyWith**: 실제 서명 / 검증(재해싱)이 일어나는 지점. → Q.정의만?
> - **signature**: header+payload를 secretKey로 해싱한 256비트 지문(토큰 3번째 조각). → Q.해싱되는 내용
> - **대칭키**: 서명·검증에 같은 키. secretKey 유출 시 위조 토큰 발급 가능. → Q.대칭키

## ④ 내가 틀렸던 것 (오개념 로그)

> [!quote]- 오개념 7건
> | 내가 생각했던 것 | 실제 |
> |---|---|
> | SHA는 암호화다 | 단방향 해싱이다(복호화 불가) |
> | SHA가 secretKey를 256비트로 바꾼다 | 해싱 대상은 header+payload, 256은 서명(키는 재료) |
> | header+signature를 합쳐서 256 | header+payload를 해싱한 결과가 signature(256), sign은 결과지 입력 아님 |
> | payload가 해싱되면 사라진다 | 원본은 그대로 남음(복사기), 256비트 서명은 별도로 붙음 |
> | 키가 정확히 256비트여야 한다 | 최소 256비트(그 이상 OK, 예시는 344비트) |
> | hmacShaKeyFor가 서명을 실행한다 | 준비만. 실제 서명은 signWith |
> | 글자 수 = 바이트 수 | ASCII만 성립. 한글 3바이트, 이모지 4바이트 |
