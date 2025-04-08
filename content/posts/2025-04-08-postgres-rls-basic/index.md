---
title: Postgres Row Level Security 알아보기
date: 2025-04-08T20:13:34+09:00
tags: [postgres]
---

## RLS란 무엇인가?

**Row Level Security(RLS)** 는 PostgreSQL에서 제공하는 **행 단위 접근 제어 기능**입니다. 사용자마다 테이블의 어떤 행을 읽고, 수정하고, 삭제할 수 있는지를 **DB 레벨에서** 결정할 수 있어 보안성과 안전성이 높아집니다.

PostgreSQL의 RLS는 테이블에 대해 다음과 같은 세부 권한 제어를 제공합니다.

```sql
ALTER TABLE posts ENABLE ROW LEVEL SECURITY;
```

---

## USING vs WITH CHECK 차이

RLS 정책에는 두 가지 조건절이 있습니다

| **절**     | **작동 시점**                                | **설명**                                            |
| ---------- | -------------------------------------------- | --------------------------------------------------- |
| USING      | 읽기/삭제/수정할 **기존 행을 대상으로** 작동 | 이 행을 내가 SELECT / UPDATE / DELETE 할 수 있는가? |
| WITH CHECK | 삽입 또는 수정할 **새 행을 대상으로** 작동   | 이 행을 내가 INSERT / UPDATE 할 수 있는가?          |

예시

```sql
CREATE POLICY user_posts_policy ON posts
  FOR ALL TO app_user
  USING (user_id = current_setting('jwt.claims.user_id')::uuid)
  WITH CHECK (user_id = current_setting('jwt.claims.user_id')::uuid);
```

이 정책은 다음과 같은 상황을 제어합니다:

- SELECT, UPDATE, DELETE: 기존 행의 user_id가 나인지 확인 (USING)
- INSERT, UPDATE: 삽입 또는 수정된 행의 user_id가 나인지 확인 (WITH CHECK)

---

## SQL 명령어별 적용 관계

| **명령어** | **USING 적용** | **WITH CHECK 적용** | **함께 사용 가능** |
| ---------- | -------------- | ------------------- | ------------------ |
| SELECT     | O              | X                   | X                  |
| INSERT     | X              | O                   | O (의미 없음)      |
| UPDATE     | O              | O                   | **O (주 사용)**    |
| DELETE     | O              | X                   | X                  |
| ALL        | O              | O                   | **O**              |

- UPDATE: 두 조건 모두 필요
- ALL: 모든 작업에 대한 공통 정책 작성 시 편리
- INSERT: WITH CHECK만 의미 있음

---

## 웹 사용자 기준으로 RLS 적용하기 (Golang 예시 포함)

웹 사용자 기준으로 정책을 만들기 위해선 **PostgreSQL 세션 변수**를 설정해야 합니다.

```sql
SET LOCAL jwt.claims.user_id = 'abc-123';
```

Golang에서 적용 방법 (표준 database/sql)

```go
tx, _ := db.BeginTx(ctx, nil)
defer tx.Rollback()

// 사용자 ID를 세션에 설정
tx.ExecContext(ctx, "SET LOCAL jwt.claims.user_id = $1", userID)

// 이후 쿼리들은 모두 RLS 정책이 적용됨
tx.QueryContext(ctx, "SELECT title FROM posts")

tx.Commit()
```

> SET LOCAL은 트랜잭션 내에서만 유효 → 반드시 트랜잭션을 사용해야 함

정책 예시

```sql
CREATE POLICY posts_rls ON posts
  FOR ALL TO app_user
  USING (user_id = current_setting('jwt.claims.user_id')::uuid)
  WITH CHECK (user_id = current_setting('jwt.claims.user_id')::uuid);
```

---

## 정책이 여러개일 경우

PostgreSQL의 Row Level Security(RLS)는 **하나의 테이블에 여러 정책(POLICY)을 정의할 수 있으며**, 이 경우 **“OR 조건”** 으로 동작합니다.

핵심 개념

- **여러 RLS 정책이 존재하면, 그 중 하나라도 통과하면 행에 접근이 허용됨**
- 즉, 모든 정책이 **AND 조건**이 아니라 **OR 조건**으로 평가됨

정책 우선순위는?

- **순서에 관계 없이 모두 평가**됩니다.
- PostgreSQL은 정책을 모두 수집해서 조건을 **OR**로 결합하여 최종 실행 계획에 적용합니다.

  **너무 느슨한 조건을 추가하면 모두 열림**

```sql
 CREATE POLICY allow_all ON posts
  FOR SELECT TO app_user
  USING (true);
```

- 이 정책 하나가 있으면, 다른 모든 SELECT 제한이 **무의미**해집니다.
- 모든 행이 조건 true를 만족하기 때문

---

## RLS를 꼭 써야 할까?

RLS는 **강력하지만 복잡**합니다. 아래 기준에 따라 사용 여부를 판단하세요.

| **조건**                   | **RLS 추천 여부** | **이유**                     |
| -------------------------- | ----------------- | ---------------------------- |
| 사용자별 데이터 격리 필요  | O                 | 보안을 DB 레벨에서 보장      |
| 단일 사용자 앱 / 내부 툴   | X                 | 과도한 설정일 수 있음        |
| 인증 미들웨어가 부족함     | O                 | 코드 실수를 막아줌           |
| 성능 중요 / 쿼리 튜닝 필요 | 신중              | 쿼리 계획이 복잡해질 수 있음 |
| ORM 주 사용                | 신중              | ORM과 충돌 가능성 있음       |

---

## 정리

- USING: 기존 행의 접근 제어 (SELECT, UPDATE, DELETE)
- WITH CHECK: 새 행의 제어 (INSERT, UPDATE)
- UPDATE는 **두 조건 모두 필요**
- SET LOCAL로 세션에 사용자 정보 전달 → 반드시 트랜잭션 안에서 실행
- Golang에서는 tx.Exec("SET LOCAL ...") → tx.Query(...) 흐름으로 안전하게 관리
- 초기 서비스에는 과할 수 있지만, **보안이 중요한 멀티유저 앱**이라면 충분히 투자할 가치 있음
