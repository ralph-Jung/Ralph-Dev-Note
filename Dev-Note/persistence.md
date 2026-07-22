# Persistence (JPA · Hibernate · JDBC · JPQL)

> 주제: 영속성 계층 개념 Q&A · 갱신: 2026-07-23 · 상태: 진행중
> 태그: #JPA #Hibernate #JDBC #JPQL #ORM #영속성 #SpringDataJPA #BigDecimal #타임존 #Instant #라이프사이클콜백
> 다음에 팔 것: 영속성 컨텍스트/1차 캐시, N+1과 fetch 전략, JPQL vs QueryDSL, @Transactional 전파, 낙관/비관 락

영속성 계층(JPA/Hibernate/JDBC/JPQL) 관련 "그때 그 질문"을 한곳에 모으는 누적 Q&A.
새 질문이 생기면 ②에 순서대로 추가한다.

## ① 큰 그림 (지도)

영속성은 3층으로 쌓여 있다. 문제 생기면 어느 층인지부터 가른다.

```
[JPA]  자바 ORM 표준 스펙 (jakarta.persistence.*)   ← 규칙/어노테이션/인터페이스
   ↓ 구현
[Hibernate]  실제 엔진 (SQL 생성·실행, 1차 캐시)     ← ORM core 7.x
   ↓ 저수준 실행
[JDBC]  자바-DB 통신 저수준 API                      ← Hibernate도 결국 JDBC로 SQL 실행

[Spring Data JPA]  위 3층을 Spring에서 쉽게 쓰는 편의계층 (org.springframework.*)
                   → JpaRepository, 메서드 이름 쿼리
```

- **JPQL**: 테이블이 아니라 **엔티티**를 대상으로 하는 객체지향 쿼리. Hibernate가 실제 SQL로 번역한다.
- import 경로가 소속을 말해준다: `jakarta.persistence.*` = JPA표준, `org.springframework.*` = Spring.
- 이 노트가 다루는 갈래: 3층 구조 · PK 채번(`@GeneratedValue`) · 필드 타입 매핑(`BigDecimal`·타임존/`Instant`) · 기본값 처리(DB DEFAULT vs 자바 초기화) · 생명주기 콜백(`@PrePersist`/`@PreUpdate`)

## ② 질문 트리 (본문) ★핵심

<!-- 2026-07-23 세션 -->

### Q. @Id 하고 @GeneratedValue 도 해줘야 돼? 안 하면 Auto인가?
- **한줄답**: `@Id`는 필수(없으면 부팅 실패). `@GeneratedValue`는 선택인데, **아예 안 붙이면 자동생성 없음**(내가 id를 직접 대입). 붙이고 strategy를 생략해야 그때 AUTO.
- **원리**: `@Id` = "이 필드가 PK"라는 표시. `@GeneratedValue` = 채번을 DB/JPA에 위임할지·어떻게 할지. "설정 안 하면 AUTO"가 아니라 **"@GeneratedValue를 붙였는데 strategy만 생략하면 AUTO"** 가 정확한 문장.
- **연결**: → 아래 Q(auto-increment면 IDENTITY?)

### Q. auto-increment로 할 거면 IDENTITY로 해야 하는 거지?
- **한줄답**: 맞다. DB 컬럼이 `BIGSERIAL`/`SERIAL`/`GENERATED AS IDENTITY`(auto-increment)면 엔티티는 `@GeneratedValue(strategy = GenerationType.IDENTITY)`.
- **원리**: IDENTITY 전략의 의미가 곧 "채번을 DB의 auto-increment 컬럼에 맡긴다". **AUTO로 두면 안 됨** — Hibernate 6/7 + Postgres에서 AUTO는 SEQUENCE 전략을 골라서, `BIGSERIAL`의 DB 채번과 엇갈려 꼬인다.
- **연결**: 전략 짝 → DB `BIGSERIAL`↔엔티티 `IDENTITY`, DB `CREATE SEQUENCE`↔엔티티 `SEQUENCE`. → 아래 Q(INSERT 때 id?)

### Q. INSERT할 때 id 컬럼은 안 넣고 다른 값들만 DB에 넣는 거야?
- **한줄답**: 그렇다. id는 빼고 나머지 컬럼만 INSERT → DB가 채번 → Hibernate가 그 값을 엔티티 `id`에 다시 채워준다.
- **원리**: `new` 시점엔 `id == null`. `save()` 하면 `INSERT INTO t (id 제외 컬럼들) VALUES (...)` 실행 → DB가 시퀀스로 채번 → 생성된 키를 리턴받아 객체에 세팅. **개발자는 id를 절대 직접 대입하지 않는다.**
- **연결**: IDENTITY는 id 회수를 위해 `save()` 시 **즉시 INSERT** → 배치 INSERT 불가. 대량 삽입 성능 필요하면 → SEQUENCE + allocationSize.

### Q. @GeneratedValue(strategy = ) 에 어떤 값이 올 수 있어?
- **한줄답**: `GenerationType` enum 5개 — `IDENTITY` / `SEQUENCE` / `AUTO` / `TABLE` / `UUID`. IDE에선 `GenerationType.`까지 치거나 스마트 자동완성(IntelliJ ⌃⇧Space)으로 목록 확인.
- **원리**: `strategy` 파라미터 타입이 **enum `GenerationType`** 이라 값이 그 상수로 고정된다. 그래서 IDE가 유효한 값만 걸러 보여주고, 오타는 컴파일 에러로 바로 잡힌다. (각 값 의미는 ③ 용어 카드 참고)
- **연결**: → 위 Q(auto-increment면 IDENTITY?) 와 짝.

### Q. NUMERIC(5,2) 컬럼은 엔티티에서 어떤 타입으로 받아?
- **한줄답**: `java.math.BigDecimal` + `@Column(precision = 5, scale = 2)`.
- **원리**: `NUMERIC`/`DECIMAL`은 **정확한 소수**(부동소수점 오차 없음)라 `BigDecimal`을 쓴다. `Double`은 오차가 생겨 금지. `precision`=전체 자릿수(5), `scale`=소수 자릿수(2) → 최대 `999.99`. NOT NULL 아니면 wrapper 타입이라 null 허용.
- **연결**: → 아래 Q(DEFAULT 처리?)

### Q. NOT NULL DEFAULT 있는 컬럼은 default 값을 어떻게 처리해?
- **한줄답**: **JPA는 DB DEFAULT를 자동으로 안 쓴다.** 자바 필드에서 초기화(`private int x = 0;`)로 기본값을 준다.
- **원리**: Hibernate는 INSERT에 **모든 컬럼을 넣어서**, 필드가 null이면 `NULL`을 그대로 INSERT → NOT NULL이면 터지고 DB DEFAULT는 미적용(DEFAULT는 컬럼을 생략했을 때만 작동). `@ColumnDefault`는 DDL 생성용이라 Flyway+`validate`에선 런타임 무효. DB DEFAULT는 JPA 안 거치는 raw SQL용 안전망으로만 남긴다.
- **연결**: → 위 Q(NUMERIC?) 와 같은 "엔티티 필드 매핑" 갈래.

### Q. TIMESTAMPTZ를 LocalDateTime으로 매핑하면 나라마다 다른 값이 되는 거 아냐?
- **한줄답**: TIMESTAMPTZ는 **`Instant`**(또는 `OffsetDateTime`)로 매핑한다. `LocalDateTime`은 쓰지 마라.
- **원리**: TIMESTAMPTZ는 DB에 **UTC 한 시점**으로 저장(타임존 미저장. UTC=그리니치 오프셋0, 한국 +9). `LocalDateTime`은 존이 없어 Hibernate가 **JVM 타임존을 가정**해 변환 → 환경(로컬 KST vs 운영 UTC)마다 같은 로우가 다른 벽시계로 튐, 다른 존 데이터 비교·정렬 깨짐(제출 순서/solving_time 직결), DST 문제. `Instant`는 UTC 절대시점이라 JVM 존 무관하게 동일. 현지화(+9 표시)는 **표시 계층에서만**.
- **연결**: → 아래 Q(created_at/updated_at 자동 기록?)

### Q. created_at / updated_at 을 자동으로 찍으려면?
- **한줄답**: **`@PrePersist`(생성 직전)·`@PreUpdate`(수정 직전)** 콜백에서 `now()`를 찍는다.
- **원리**: 필드 초기화(`= now()`)는 `new` 될 때 **1회만** 실행 → `updated_at`이 수정 시각을 못 따라감(버그). `created_at`은 필드초기화도 버그는 아니나, 생성(new) vs 저장(persist) 시점 차이 + 일관성 때문에 `@PrePersist` 권장. 콜백은 Hibernate가 flush 직전 자동 호출(**어노테이션이 트리거**, 메서드명 자유, 엔티티 내부면 `void`+no-arg, `private` 가능). 콜백 안에서 다른 DB 작업 금지.
- **연결**: 콜백이 이름 무관하게 호출되는 내부 원리(리플렉션)는 → [[java-basics]] (자바 언어 개념).

## ③ 용어 카드 (역참조)

- **JPA**: 자바 ORM 표준 스펙(`jakarta.persistence`). 어노테이션·인터페이스 규칙만 정의. → ① 큰 그림
- **Hibernate**: JPA 구현체(엔진). 실제 SQL 생성·실행, 1차 캐시/dirty checking. → ① 큰 그림
- **Spring Data JPA**: JPA/Hibernate를 Spring에서 쉽게 쓰게 감싼 계층. `JpaRepository`, 메서드 이름 쿼리. → ① 큰 그림
- **JDBC**: 자바-DB 저수준 통신 API. Hibernate도 내부적으로 JDBC로 SQL을 던진다. → ① 큰 그림
- **JPQL**: 엔티티(테이블 아님) 대상 객체지향 쿼리. Hibernate가 SQL로 번역. → ① 큰 그림
- **@GeneratedValue 전략**: IDENTITY(DB auto-increment) / SEQUENCE(DB 시퀀스, 배치 유리) / AUTO(방언이 선택, PG=SEQUENCE) / TABLE(채번 테이블, 거의 안 씀) / UUID(JPA 3.1). → Q(@GeneratedValue 값?)
- **BigDecimal**: 정확한 소수 타입(부동소수점 오차 없음). NUMERIC/DECIMAL 매핑, precision/scale. → Q(NUMERIC?)
- **Instant**: UTC 타임라인 위의 한 점(오프셋0 기준). TIMESTAMPTZ 매핑에 적합. → Q(TIMESTAMPTZ?)
- **@PrePersist / @PreUpdate**: JPA 생명주기 콜백. INSERT/UPDATE 직전 Hibernate가 자동 호출(엔티티 필드 손보는 용도). → Q(created_at/updated_at?)

## ④ 내가 틀렸던 것 (오개념 로그) ★가치 높음

| 내가 생각했던 것 | 실제 |
|---|---|
| `@GeneratedValue`를 안 붙이면 AUTO로 자동생성된다 | 안 붙이면 자동생성 자체가 없음(내가 id 대입). **붙이고 strategy 생략**해야 AUTO |
| AUTO = auto-increment(IDENTITY) | Hibernate 6/7 + Postgres에서 AUTO는 **SEQUENCE**. auto-increment면 `IDENTITY`를 명시해야 함 |
| INSERT할 때 id도 넣어야 한다 | id는 빼고 INSERT → DB가 채번 → Hibernate가 회수해 세팅 |
| DB에 `DEFAULT`가 있으면 JPA가 알아서 기본값을 넣어준다 | JPA는 모든 컬럼을 INSERT에 넣어 필드 null이면 NULL → DEFAULT 미적용. **자바 필드 초기화** 필요 |
| TIMESTAMPTZ는 LocalDateTime으로 받으면 된다 | LocalDateTime은 존이 없어 JVM 타임존 가정 → 환경별 값 이동·비교 깨짐. **Instant**로 |
| `updated_at`도 필드 `= now()`로 두면 갱신된다 | 그건 생성 때 1회뿐. 수정마다 반영하려면 **@PreUpdate** |
