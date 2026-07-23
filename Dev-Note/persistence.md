# Persistence (JPA · Hibernate · JDBC · JPQL)

> **주제**: 영속성 계층 개념 Q&A · 갱신: 2026-07-24 · 상태: 진행중
> **태그**: #JPA #Hibernate #ORM #영속성 #엔티티매핑

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
- **식별자(PK)는 항상 "하나의 객체"** 로 다뤄진다. `find(Class, Object)`가 값을 하나만 받기 때문. 그래서 단일키든 복합키든 식별자 타입에 붙는 요구사항(직렬화 가능 등)이 동일하게 적용된다. 복합키는 그 "하나의 객체"를 우리가 직접 만들어야 하는 경우일 뿐이다.
- 복합키 매핑의 갈래: `@EmbeddedId`(ID 클래스를 엔티티 필드로 품음) vs `@IdClass`(바깥에 둠) → 어느 쪽이든 ID 클래스는 필요하고, 연관관계와 함께 쓰면 `@MapsId`로 FK 컬럼을 PK에 재사용한다.
- 이 노트가 다루는 갈래: 3층 구조 · PK 채번(`@GeneratedValue`) · **복합키/식별자 타입의 요구사항** · 필드 타입 매핑(`BigDecimal`·타임존/`Instant`·**enum**) · 기본값 처리(DB DEFAULT vs 자바 초기화) · 생명주기 콜백(`@PrePersist`/`@PreUpdate`)

## ② 질문 트리 (본문)

### 2026-07-23

#### Q. @Id 하고 @GeneratedValue 도 해줘야 돼? 안 하면 Auto인가?
- **한줄답**: `@Id`는 필수(없으면 부팅 실패). `@GeneratedValue`는 선택인데, ==아예 안 붙이면 자동생성이 없다==(내가 id를 직접 대입). 붙이고 strategy를 생략해야 그때 AUTO.
- **원리**: `@Id` = "이 필드가 PK"라는 표시. `@GeneratedValue` = 채번을 DB/JPA에 위임할지·어떻게 할지. "설정 안 하면 AUTO"가 아니라 ==@GeneratedValue를 붙였는데 strategy만 생략하면 AUTO== 가 정확한 문장.
- **연결**: → 아래 Q(auto-increment면 IDENTITY?)

#### Q. auto-increment로 할 거면 IDENTITY로 해야 하는 거지?
- **한줄답**: 맞다. DB 컬럼이 `BIGSERIAL`/`SERIAL`/`GENERATED AS IDENTITY`(auto-increment)면 엔티티는 `@GeneratedValue(strategy = GenerationType.IDENTITY)`.
- **원리**: IDENTITY 전략의 의미가 곧 "채번을 DB의 auto-increment 컬럼에 맡긴다". ==AUTO로 두면 안 된다== — Hibernate 6/7 + Postgres에서 AUTO는 SEQUENCE 전략을 골라서, `BIGSERIAL`의 DB 채번과 엇갈려 꼬인다.
- **연결**: 전략 짝 → DB `BIGSERIAL`↔엔티티 `IDENTITY`, DB `CREATE SEQUENCE`↔엔티티 `SEQUENCE`. → 아래 Q(INSERT 때 id?)

#### Q. INSERT할 때 id 컬럼은 안 넣고 다른 값들만 DB에 넣는 거야?
- **한줄답**: 그렇다. id는 빼고 나머지 컬럼만 INSERT → DB가 채번 → Hibernate가 그 값을 엔티티 `id`에 다시 채워준다.
- **원리**: `new` 시점엔 `id == null`. `save()` 하면 `INSERT INTO t (id 제외 컬럼들) VALUES (...)` 실행 → DB가 시퀀스로 채번 → 생성된 키를 리턴받아 객체에 세팅. ==개발자는 id를 절대 직접 대입하지 않는다.==
- **연결**: IDENTITY는 id 회수를 위해 `save()` 시 **즉시 INSERT** → 배치 INSERT 불가. 대량 삽입 성능 필요하면 → SEQUENCE + allocationSize.

#### Q. @GeneratedValue(strategy = ) 에 어떤 값이 올 수 있어?
- **한줄답**: `GenerationType` enum 5개 — `IDENTITY` / `SEQUENCE` / `AUTO` / `TABLE` / `UUID`. IDE에선 `GenerationType.`까지 치거나 스마트 자동완성(IntelliJ ⌃⇧Space)으로 목록 확인.
- **원리**: ==strategy 파라미터 타입이 enum이라 값이 그 상수로 고정된다.== 그래서 IDE가 유효한 값만 걸러 보여주고, 오타는 컴파일 에러로 바로 잡힌다. (각 값 의미는 ③ 용어 카드 참고)
- **연결**: → 위 Q(auto-increment면 IDENTITY?) 와 짝.

#### Q. NUMERIC(5,2) 컬럼은 엔티티에서 어떤 타입으로 받아?
- **한줄답**: `java.math.BigDecimal` + `@Column(precision = 5, scale = 2)`.
- **원리**: `NUMERIC`/`DECIMAL`은 **정확한 소수**(부동소수점 오차 없음)라 `BigDecimal`을 쓴다. ==Double은 오차가 생겨 금지.== `precision`=전체 자릿수(5), `scale`=소수 자릿수(2) → 최대 `999.99`. NOT NULL 아니면 wrapper 타입이라 null 허용.
- **연결**: → 아래 Q(DEFAULT 처리?)

#### Q. NOT NULL DEFAULT 있는 컬럼은 default 값을 어떻게 처리해?
- **한줄답**: **JPA는 DB DEFAULT를 자동으로 안 쓴다.** 자바 필드에서 초기화(`private int x = 0;`)로 기본값을 준다.
- **원리**: Hibernate는 INSERT에 **모든 컬럼을 넣어서**, 필드가 null이면 `NULL`을 그대로 INSERT → NOT NULL이면 터지고 DB DEFAULT는 미적용 — ==DEFAULT는 컬럼을 아예 생략했을 때만 작동하기 때문==이다. `@ColumnDefault`는 DDL 생성용이라 Flyway+`validate`에선 런타임 무효. DB DEFAULT는 JPA 안 거치는 raw SQL용 안전망으로만 남긴다.
- **연결**: → 위 Q(NUMERIC?) 와 같은 "엔티티 필드 매핑" 갈래. → 아래 Q(필드 초기화는 많이 안 쓰나?)의 실전편.

#### Q. TIMESTAMPTZ를 LocalDateTime으로 매핑하면 나라마다 다른 값이 되는 거 아냐?
- **한줄답**: TIMESTAMPTZ는 **`Instant`**(또는 `OffsetDateTime`)로 매핑한다. ==LocalDateTime은 쓰지 마라.==
- **원리**: TIMESTAMPTZ는 DB에 **UTC 한 시점**으로 저장(타임존 미저장. UTC=그리니치 오프셋0, 한국 +9). `LocalDateTime`은 존이 없어 Hibernate가 ==JVM 타임존을 가정해 변환==한다 → 환경(로컬 KST vs 운영 UTC)마다 같은 로우가 다른 벽시계로 튐, 다른 존 데이터 비교·정렬 깨짐(제출 순서/solving_time 직결), DST 문제. `Instant`는 UTC 절대시점이라 JVM 존 무관하게 동일. 현지화(+9 표시)는 **표시 계층에서만**.
- **연결**: → 아래 Q(created_at/updated_at 자동 기록?)

#### Q. created_at / updated_at 을 자동으로 찍으려면?
- **한줄답**: **`@PrePersist`(생성 직전)·`@PreUpdate`(수정 직전)** 콜백에서 `now()`를 찍는다.
- **원리**: 필드 초기화(`= now()`)는 `new` 될 때 ==1회만 실행되므로 updated_at이 수정 시각을 못 따라간다==(버그). `created_at`은 필드초기화도 버그는 아니나, 생성(new) vs 저장(persist) 시점 차이 + 일관성 때문에 `@PrePersist` 권장. 콜백은 Hibernate가 flush 직전 자동 호출(**어노테이션이 트리거**, 메서드명 자유, 엔티티 내부면 `void`+no-arg, `private` 가능). ==콜백 안에서 다른 DB 작업은 금지.==
- **연결**: 콜백이 이름 무관하게 호출되는 내부 원리(리플렉션)는 → [[java-basics]] (자바 언어 개념).

#### Q. 복합키를 쓸 때 왜 ID 클래스를 따로 만들어야 하나? "JPA는 @Id가 하나여야 한다"가 이유인가?
- **한줄답**: 그 이유는 틀렸다. ==@IdClass를 쓰면 @Id를 두 개 달 수 있다.== 진짜 이유는 ==식별자를 "값 하나"로 넘겨야 하는 API 지점==이 존재하기 때문이다.
- **원리**: `EntityManager`의 시그니처가 가변인자가 아니다.

```java
<T> T find(Class<T> entityClass, Object primaryKey);   // 인자 딱 하나
```

  `find(ProblemTag.class, 1L, 5L)` 같은 API는 존재하지 않는다. 그래서 `@IdClass`로 `@Id`를 두 개 달았더라도 **조회할 때는 결국 ID 클래스를 만들어 넘겨야 한다.**

```java
em.find(ProblemTag.class, new ProblemTagId(1L, 5L));   // @IdClass든 @EmbeddedId든 동일
```

  같은 제약이 걸리는 다른 지점: `getReference()`, `JpaRepository<T, ID>`의 두 번째 타입 파라미터.
  `@Id`를 두 개 다는 것 자체는 스펙상 합법이다 — 아래가 반례다.

```java
@Entity
@IdClass(ProblemTagId.class)
public class ProblemTag {
    @Id @ManyToOne(fetch = LAZY) @JoinColumn(name = "problem_id") private Problem problem;
    @Id @ManyToOne(fetch = LAZY) @JoinColumn(name = "tag_id")     private Tag tag;
}
```

  결론: `@IdClass`와 `@EmbeddedId`의 차이는 **ID 클래스를 바깥에 두느냐 엔티티 필드로 품느냐**일 뿐이고, ==ID 클래스 자체는 어느 쪽이든 못 피한다.==
- **연결**: → 아래 Q(@Embeddable과 @EmbeddedId 차이?), → 아래 Q(왜 Serializable?)

#### Q. @Embeddable도 있고 @EmbeddedId도 있는데 차이가 뭐야? 용도가 다른가?
- **한줄답**: ==대안 관계가 아니라 짝을 이루는 관계==다. 붙는 위치가 다르다 — `@Embeddable`은 **클래스**(재료 정의), `@EmbeddedId`는 **필드**(쓰는 자리).
- **원리**: `@Embeddable` 없이 `@EmbeddedId`만 쓰면 부팅 시점에 매핑 예외가 난다.

```java
@Embeddable                                  // 재료 선언 (클래스)
public class ProblemTagId implements Serializable { ... }

@Entity
public class ProblemTag {
    @EmbeddedId                              // 쓰는 자리 (필드)
    private ProblemTagId id;
}
```

  쓰는 자리는 두 곳이다 — 식별자면 `@EmbeddedId`, 일반 필드면 `@Embedded`. 즉 ==@EmbeddedId의 진짜 대립항은 @Embeddable이 아니라 @Embedded==다.

| | 클래스 쪽 | 필드 쪽 | 의미 |
|---|---|---|---|
| 식별자로 | `@Embeddable` | `@EmbeddedId` | PK가 됨 |
| 일반 값으로 | `@Embeddable` | `@Embedded` | 그냥 컬럼들 |

  복합키는 `@Embeddable`의 **부수적 쓰임**이다. 본래 목적은 의미 있는 필드들을 묶어 **값 객체**로 만드는 것 — 예를 들어 `Problem`의 `timeLimit`/`memoryLimit`을 `JudgeLimit`으로 묶으면 `exceeds(runtimeMs, memoryKb)` 같은 **로직을 붙일 자리**가 생긴다. 테이블은 그대로고(컬럼이 그 자리에 그대로 남음) 자바 쪽에서만 묶여 보인다.

  같은 `@Embeddable`이라도 어디에 쓰느냐에 따라 요구사항이 다르다:

| | `@EmbeddedId` (식별자) | `@Embedded` (값) |
|---|---|---|
| `Serializable` | 스펙상 필수 | 불필요 |
| `equals`/`hashCode` | 필수 | 권장 |
| null 허용 | 불가 | 가능 (전 필드 null이면 객체도 null) |
| 인스턴스 공유 | 해당 없음 | **금지** |

  마지막 항목이 함정이다 — ==@Embedded 값 객체를 두 엔티티가 같은 인스턴스로 공유하면 한쪽 수정이 다른 쪽에 전파된다.== 그래서 값 객체는 setter 없이 불변으로 만들고 통째로 교체한다(복합키가 불변이어야 하는 이유와 같은 맥락). 같은 값 객체를 한 엔티티에 두 번 쓰면 컬럼명이 충돌하므로 `@AttributeOverride`로 가른다.
- **연결**: → 위 Q(왜 ID 클래스가 필요한가?), 불변성은 → 아래 Q(왜 Serializable?)

#### Q. 복합키 클래스는 왜 Serializable로 구현을 하는 거야?
- **한줄답**: ==복합키라서 생긴 요구가 아니다.== **식별자는 원래부터 직렬화 가능해야 했고**, `Long`·`String`은 JDK가 이미 만족시켜 놔서 티가 안 났을 뿐이다.
- **원리**: `Long extends Number`이고 `Number implements Serializable`, `String`도 직접 `implements Serializable`. 그래서 `@Id Long id`는 손댈 게 없었다. 반면 복합키 클래스는 **내가 만든 타입**이라 내가 붙여야 한다. ==실제 코드에도 분기가 없다== — `EntityKey.serialize()`가 `oos.writeObject(this.identifier)` 한 줄로 `Long`이든 `ProblemTagId`든 똑같이 처리한다.
  터지는 지점은 두 곳: ① 영속성 컨텍스트 직렬화(`StatefulPersistenceContext.serialize()` → 세션 클러스터링/passivation), ② 분산 2차 캐시 키(`CacheKeyImplementation implements Serializable`이 `Object id`를 필드로 보유 → Redis/Hazelcast 전송 시).
  단, **Hibernate 7.4.1에 부팅 시점 검증은 없다**(mapping·boot 패키지에 검사 코드 부재). 즉 안 붙여도 뜨고, 캐시나 다중 인스턴스를 붙이는 날 `NotSerializableException`으로 터진다.
  스펙 조문상 경로도 둘로 갈린다 — 복합 PK는 §2.4가 "serializable이어야 한다"고 직접 명시하고, 단일 PK는 **허용 타입 목록**(primitive/wrapper/String/BigDecimal/Date…)을 열거하는데 그 목록이 전부 Serializable이다.
- **연결**: `Number` 상속 계층과 `find(Class, Object)`의 오토박싱은 → [[java-basics]]. → 위 Q(왜 ID 클래스가 필요한가?)와 같은 "식별자" 갈래.

#### Q. @Enumerated(EnumType.STRING) 하고 @JdbcTypeCode(SqlTypes.NAMED_ENUM), 이 두 줄이 무슨 말이야?
- **한줄답**: 변환이 **두 번** 일어나는데 각각 다른 층을 정한다. `@Enumerated` = **무엇을 보낼지**(값), `@JdbcTypeCode` = **어떤 타입으로 보낼지**(봉투). **두 줄을 다 붙이는 건 PostgreSQL + 네이티브 ENUM 스키마일 때뿐이다.** 그 외에는 전부 `@Enumerated(STRING)` 하나만 쓴다.
- **원리**:

```
1층 — @Enumerated: 자바 enum을 어떤 값으로 표현할지
    STRING  → 'UNSOLVED'  (상수 이름)
    ORDINAL → 0           (선언 순서 번호, 생략 시 기본값)
2층 — @JdbcTypeCode: 그 값을 어떤 JDBC 타입으로 실어보낼지
    (없으면)   → 평범한 VARCHAR 파라미터
    NAMED_ENUM → 이름 있는 ENUM 타입이라고 표시해서 보냄

언제 두 줄 다 쓰나 — 조건 두 개가 모두 맞을 때만:
    1) DB가 PostgreSQL
    2) 컬럼이 CREATE TYPE ... AS ENUM 으로 만든 네이티브 ENUM 타입
    → 둘 다 맞으면   @Enumerated(STRING) + @JdbcTypeCode(NAMED_ENUM)
    → 하나라도 아니면 @Enumerated(STRING) 만
```

  `CREATE TYPE ... AS ENUM`은 제약조건이 아니라 **독립 타입**을 만든 것이라, ==문자열처럼 보여도 VARCHAR가 아니다.== 그래서 varchar 파라미터를 그대로 못 받는다. 대부분의 스키마는 enum을 VARCHAR 컬럼으로 잡으므로 이 조건에 안 걸린다.
  DB별로 갈리는 지점 — Hibernate가 상수를 나눠둔 이유: `SqlTypes.ENUM`=6000(인라인 ENUM, MySQL은 컬럼 정의에 값이 박힘), `SqlTypes.NAMED_ENUM`=6001(이름 붙은 ENUM 타입, PostgreSQL). MySQL 인라인 ENUM은 문자열을 그대로 받아서 `@Enumerated(STRING)` 하나로 충분하다. 즉 실질적으로 PostgreSQL 전용 얘기지만, 더 정확한 기준은 "PostgreSQL이라서"가 아니라 ==타입에 이름이 있어서==다. (DB를 바꿔도 이 기준으로 판단하면 된다.)
  **psql에선 되는데 JPA에선 안 되는 이유**: 리터럴(`'UNSOLVED'`)은 타입이 미정이라 Postgres가 문맥을 보고 해석해 주지만, JDBC 파라미터 바인딩은 **타입이 못박혀** 나가서 암묵 캐스팅이 사라진다.
  ORDINAL은 네이티브 ENUM이 아니어도 쓰면 안 된다 — **순서 번호**라서 enum 중간에 값을 하나 끼워 넣으면 기존 데이터의 의미가 통째로 밀린다(SOLVED였던 행이 REVIEW가 되는 식). 에러도 안 난다. `@Enumerated`는 항상 명시한다.
- **연결**: enum 자체는 → [[java-basics]]. → 위 Q(NUMERIC 컬럼은 어떤 타입으로?)와 같은 "필드 타입 매핑" 갈래.

#### Q. private Integer displayOrder = 0; 처럼 필드에 값을 초기화하는 방식은 보통 많이 안 쓰나?
- **한줄답**: 많이 쓴다. ==JPA에서는 기본값을 주는 사실상 유일한 수단==이다. 다만 ==생성자가 받는 필드에 쓰면 죽은 코드==가 된다.
- **원리**: JPA는 DB의 `DEFAULT`를 안 쓴다(위 Q 참조) — Hibernate가 INSERT에 모든 컬럼을 넣어서 필드가 null이면 NULL이 그대로 들어가고 NOT NULL이면 터진다. 그래서 자바 필드 초기화가 기본값 역할을 대신해야 한다.

  판단 기준은 하나 — **이 값을 만드는 쪽이 정해야 하나, 아니면 항상 같은 값으로 시작하나?**

```
항상 같은 값으로 시작  → 필드 초기화
만들 때마다 다름       → 생성자 파라미터
```

  "생성자가 받으면 초기화를 지운다"는 규칙이 아니라 **이 기준의 결과**다. 생성자가 받는다는 건 이미 "만드는 쪽이 정한다"고 결정했다는 뜻이라, 초기화가 논리적으로 죽는다. 실행 순서가 ==필드 초기화 → 생성자 본문==이므로 항상 덮어써진다. DB에서 읽어올 때도 no-arg 생성자로 만든 뒤 리플렉션으로 주입하므로 마찬가지.

  필드 초기화가 정답인 대표 사례:
  1. **컬렉션** — `private List<ProblemTag> tags = new ArrayList<>();` ==JPA 엔티티에서 컬렉션은 무조건 초기화==한다. null로 두면 `getTags().add(...)` 에서 NPE
  2. **생성 시점에 값이 정해진 상태 필드** — `status = UNSOLVED`, `tryCount = 0`, `isBookmarked = false`. "새로 만드는데 이미 SOLVED"는 말이 안 되므로 호출부가 정할 여지가 없다
  3. **통계·카운터** — `solvedCount = 0`, `submissionCount = 0`

  **한 필드가 경로마다 다를 수도 있다** — `TestCase.displayOrder` 가 그 예다.

```
sample()   백준 "예제 1,2,3" 순서를 호출부가 앎  → 파라미터로 받음
custom()   개인 테스트케이스라 순서 무의미        → 항상 0
```

  이럴 때 정적 팩토리를 나눠두면 각자 다르게 처리할 수 있다. 생성자 하나로 다 받으면 이 차이를 표현할 방법이 없다.
- **연결**: → 위 Q(NOT NULL DEFAULT 있는 컬럼은 어떻게 처리해?)의 실전 적용편.

## ③ 용어 카드 (역참조)

> [!quote]- 용어 17개
> - **JPA**: 자바 ORM 표준 스펙(`jakarta.persistence`). 어노테이션·인터페이스 규칙만 정의. → ① 큰 그림
> - **Hibernate**: JPA 구현체(엔진). 실제 SQL 생성·실행, 1차 캐시/dirty checking. → ① 큰 그림
> - **Spring Data JPA**: JPA/Hibernate를 Spring에서 쉽게 쓰게 감싼 계층. `JpaRepository`, 메서드 이름 쿼리. → ① 큰 그림
> - **JDBC**: 자바-DB 저수준 통신 API. Hibernate도 내부적으로 JDBC로 SQL을 던진다. → ① 큰 그림
> - **JPQL**: 엔티티(테이블 아님) 대상 객체지향 쿼리. Hibernate가 SQL로 번역. → ① 큰 그림
> - **@GeneratedValue 전략**: IDENTITY(DB auto-increment) / SEQUENCE(DB 시퀀스, 배치 유리) / AUTO(방언이 선택, PG=SEQUENCE) / TABLE(채번 테이블, 거의 안 씀) / UUID(JPA 3.1). → Q(@GeneratedValue 값?)
> - **BigDecimal**: 정확한 소수 타입(부동소수점 오차 없음). NUMERIC/DECIMAL 매핑, precision/scale. → Q(NUMERIC?)
> - **Instant**: UTC 타임라인 위의 한 점(오프셋0 기준). TIMESTAMPTZ 매핑에 적합. → Q(TIMESTAMPTZ?)
> - **@PrePersist / @PreUpdate**: JPA 생명주기 콜백. INSERT/UPDATE 직전 Hibernate가 자동 호출(엔티티 필드 손보는 용도). → Q(created_at/updated_at?)
> - **@IdClass**: 복합키를 엔티티 **바깥** 클래스로 두는 방식. 엔티티에 `@Id`를 여러 개 달며, ID 클래스와 필드를 두 벌 선언해야 한다. → Q(왜 ID 클래스가 필요한가?)
> - **@Embedded**: `@Embeddable` 클래스를 **일반 필드**로 포함. `@EmbeddedId`와 짝을 이루는 대립항. → Q(@Embeddable과 차이?)
> - **값 객체(Value Object)**: 의미 있는 필드들을 묶어 로직을 붙일 자리를 만드는 `@Embeddable` 타입. 테이블 구조는 그대로다. 불변으로 만들고 인스턴스를 공유하지 않는다. → Q(@Embeddable과 차이?)
> - **Serializable**: 마커 인터페이스(메서드 없음). JPA 식별자 타입의 요구사항이며 `Number`·`String`이 이미 구현해 단일키는 자동 충족. 복합키 클래스는 직접 붙여야 한다. → Q(왜 Serializable?)
> - **EntityKey**: Hibernate 1차 캐시(`Map<EntityKey, Object>`)의 키. 식별자 + 엔티티명을 묶는다. `serialize()`가 식별자를 `writeObject`로 내보낸다. → Q(왜 Serializable?)
> - **네이티브 ENUM (Postgres)**: `CREATE TYPE ... AS ENUM`으로 만든 **독립 타입**. VARCHAR가 아니라 별개 타입이라 varchar 파라미터를 그대로 못 받는다. 정렬은 **선언 순서**를 따른다. → Q(두 줄이 무슨 말?)
> - **@Enumerated**: 자바 enum을 이름(STRING)으로 쓸지 순서번호(ORDINAL)로 쓸지 결정. **생략하면 ORDINAL**. → Q(두 줄이 무슨 말?)
> - **@JdbcTypeCode**: 값을 어떤 JDBC/SQL 타입으로 바인딩할지 지정(Hibernate 6+). `SqlTypes.ENUM`=6000(인라인), `NAMED_ENUM`=6001(이름 붙은 타입). → Q(두 줄이 무슨 말?)

## ④ 내가 틀렸던 것 (오개념 로그)

> [!quote]- 오개념 16건
> | 내가 생각했던 것 | 실제 |
> |---|---|
> | `@GeneratedValue`를 안 붙이면 AUTO로 자동생성된다 | 안 붙이면 자동생성 자체가 없음(내가 id 대입). **붙이고 strategy 생략**해야 AUTO |
> | AUTO = auto-increment(IDENTITY) | Hibernate 6/7 + Postgres에서 AUTO는 **SEQUENCE**. auto-increment면 `IDENTITY`를 명시해야 함 |
> | INSERT할 때 id도 넣어야 한다 | id는 빼고 INSERT → DB가 채번 → Hibernate가 회수해 세팅 |
> | DB에 `DEFAULT`가 있으면 JPA가 알아서 기본값을 넣어준다 | JPA는 모든 컬럼을 INSERT에 넣어 필드 null이면 NULL → DEFAULT 미적용. **자바 필드 초기화** 필요 |
> | TIMESTAMPTZ는 LocalDateTime으로 받으면 된다 | LocalDateTime은 존이 없어 JVM 타임존 가정 → 환경별 값 이동·비교 깨짐. **Instant**로 |
> | `updated_at`도 필드 `= now()`로 두면 갱신된다 | 그건 생성 때 1회뿐. 수정마다 반영하려면 **@PreUpdate** |
> | JPA는 `@Id`가 하나여야 해서 ID 클래스가 필요하다 | `@IdClass`는 `@Id` 두 개 가능. 진짜 이유는 `find(Class, Object)`가 값을 **하나만** 받아서 |
> | `@Embeddable`과 `@EmbeddedId` 중 하나를 고른다 | 둘은 **짝**(클래스 vs 필드). 고르는 건 `@EmbeddedId` vs `@Embedded` |
> | `@Embedded` 값 객체는 재사용하면 효율적이다 | 인스턴스를 공유하면 한쪽 수정이 다른 쪽에 전파됨. 불변으로 만들고 통째로 교체 |
> | 복합키라서 특별히 직렬화가 필요하다 | 단일키도 원래 요구사항. `Long`이 `Number`를 통해 이미 Serializable이라 안 보였을 뿐 |
> | Serializable을 빼면 부팅 때 걸러진다 | Hibernate는 부팅 검증을 안 함. 2차 캐시·세션 클러스터링을 붙이는 시점에 런타임으로 터짐 |
> | Postgres ENUM 컬럼은 결국 문자열이니 `@Enumerated(STRING)`이면 된다 | VARCHAR와 **별개 타입**이라 실패. `@JdbcTypeCode(NAMED_ENUM)`이 추가로 필요 |
> | `@Enumerated`를 생략하면 이름으로 저장된다 | 기본값은 **ORDINAL**(숫자). 순서가 밀리면 기존 데이터 의미가 통째로 바뀜 |
> | `@JdbcTypeCode`는 enum 매핑에 늘 필요하다 | 평소엔 `@Enumerated(STRING)` 하나면 됨. **PostgreSQL + 네이티브 ENUM**일 때만 추가 |
> | psql에서 `'UNSOLVED'`가 들어가니 JPA에서도 된다 | 리터럴은 타입 미정이라 캐스팅됨. JDBC 파라미터는 타입이 못박혀 캐스팅 안 됨 |
> | 생성자로도 받고 필드 초기화도 해두면 안전망이 된다 | 필드 초기화 → 생성자 본문 순이라 **항상 덮어써짐**. null을 넘기면 그대로 null이 되어 안전망처럼 보이지만 아니다 |
