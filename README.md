# Ralph Dev Note

> 개념이 아니라 "그때 그 질문"으로 되찾는 개발 학습 노트

공부한 내용을 개념 사전처럼 쌓지 않는다. 미래의 나는 "JWT 서명이 뭐였지"가 아니라
**"그때 내가 왜 이걸 못 알아들었지"** 로 기억을 되짚는다. 그래서 이 볼트는 **질문을 인덱스로** 삼는다.

---

## 노트 읽는 법 — 4층 구조

모든 노트는 같은 4층 구조를 따른다. 위에서 아래로 갈수록 세부적이다.

| 층 | 이름 | 역할 |
|---|---|---|
| ① | 큰 그림 (지도) | 세부를 다 잊어도 이것만 보면 맥락이 복구된다 |
| ② | 질문 트리 ★ | 그때 던진 질문을 실제 순서대로. 이 노트의 핵심 |
| ③ | 용어 카드 | 빠른 점프용 짧은 사전 |
| ④ | 내가 틀렸던 것 | "아 그게 아니구나" 한 오개념 교정 로그 (가치 높음) |

읽는 순서 팁: **막혔을 땐 ①로 맥락을 잡고, 되살릴 땐 ②의 질문을 따라간다.**

---

## 노트 목록

<!-- NOTE-INDEX:START — 이 블록은 note 스킬이 Dev-Note/를 스캔해 자동 재생성한다. 직접 수정 금지 -->
**Spring Security** (`Dev-Note/SpringSecurity/`)

- [필터 체인 구조 (어디서 도는가)](Dev-Note/SpringSecurity/FilterChain.md)
- [인증 (너 누구냐)](Dev-Note/SpringSecurity/Authentication.md)
- [인가 (그거 해도 되냐)](Dev-Note/SpringSecurity/Authorization.md)
- [예외 처리 (거부되면 뭘 돌려주나)](Dev-Note/SpringSecurity/ExceptionHandling.md)
- [방어 계층 (공격을 어떻게 막나)](Dev-Note/SpringSecurity/Protection.md)
- [OAuth2 소셜 로그인 (구글)](Dev-Note/SpringSecurity/OAuth2.md)

**인증·보안**

- [토큰 기반 인증 (JWT · Refresh)](Dev-Note/TokenAuth.md)
- [서블릿 필터 (실행 모델 · 응답 작성)](Dev-Note/ServletFilter.md)

**데이터**

- [Persistence (JPA · Hibernate · JDBC · JPQL)](Dev-Note/Persistence.md)
- [DB 기초 (SQL · 스키마 · 마이그레이션)](Dev-Note/DbBasics.md)

**언어**

- [Java 기초 (언어/문법)](Dev-Note/JavaBasics.md)

**기록**

- [Trouble Shooting](Dev-Note/TroubleShooting.md)
<!-- NOTE-INDEX:END -->

---

## 노트는 어떻게 만들어지나

Claude Code와의 학습 세션 대화를 그대로 흘려보내지 않고, 세션이 끝날 때
**질문이 파고든 흐름을 4층 구조로 정리해** 이 볼트에 저장한다.

- 한 개념을 반복해서 되물은 주제일수록 복습 가치가 높다고 보고 우선 정리한다.
- 관련된 기존 노트가 있으면 새 파일을 만들지 않고 그 노트에 이어 붙여, 문서가 자라나게 한다.

## Obsidian 볼트

이 저장소는 Obsidian 볼트다. 노트끼리 `[[위키링크]]`로 연결되어 그래프 뷰에서
개념 간 관계가 한눈에 보인다. (GitHub에서는 위키링크 대신 위 목록의 일반 링크로 이동)

로컬 Obsidian 설정(`.obsidian/`)은 개인 UI 상태라 저장소에 포함하지 않는다.
