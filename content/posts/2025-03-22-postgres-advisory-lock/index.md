---
title: PostgreSQL Advisory Lock 동시성 제어를 위한 강력한 도구
date: 2025-03-22T04:05:56+09:00
tags: [postgres]
---

## 1. 서론

### Advisory Lock이란?

Advisory Lock은 **PostgreSQL에서 제공하는 사용자 정의 잠금(User-defined lock) 메커니즘**입니다. 테이블이나 행을 직접 잠그는 것이 아니라, 사용자가 정의한 키(key) 값을 기반으로 잠금을 설정할 수 있습니다.

이 잠금은 PostgreSQL 내부에서 관리되며, **논리적(Logical) 잠금**으로 작동합니다. 즉, Advisory Lock은 강제성이 없고, 애플리케이션 레벨에서 적절히 구현해야 합니다.

### PostgreSQL에서의 Lock 개요

PostgreSQL에는 여러 종류의 잠금이 존재합니다.

- **Row-level Lock**: `SELECT ... FOR UPDATE`를 사용하여 특정 행을 잠금
- **Table-level Lock**: `LOCK TABLE`을 사용하여 테이블 전체를 잠금
- **Advisory Lock**: 특정 숫자 값을 기반으로 사용자가 직접 관리하는 잠금

Advisory Lock은 트랜잭션이나 테이블과 무관하게 특정 리소스(예: 사용자 ID, 주문 ID 등)에 대한 동시성을 제어하는 데 유용합니다.

### Advisory Lock이 필요한 이유

- 특정 리소스(예: 특정 작업 ID)에 대한 **동시 실행 방지**
- 트랜잭션 기반이 아닌 **애플리케이션 레벨에서 동시성 관리 가능**
- 기존 행(Row)에 대한 직접적인 락을 걸지 않으므로 **데드락(Deadlock) 위험 감소**
- **분산 시스템에서도 활용 가능** (Advisory Lock을 통해 동기화)

---

## 2. Advisory Lock의 개념

### PostgreSQL의 Advisory Lock vs. 일반 Lock

| Lock 유형     | 대상         | 자동 해제             | 강제성              |
| ------------- | ------------ | --------------------- | ------------------- |
| Row Lock      | 특정 행      | 트랜잭션 종료 시      | 강제                |
| Table Lock    | 특정 테이블  | 트랜잭션 종료 시      | 강제                |
| Advisory Lock | 특정 숫자 값 | 설정 방식에 따라 다름 | 애플리케이션이 관리 |

Advisory Lock은 PostgreSQL이 강제하는 것이 아니라, **애플리케이션에서 잘 설계해야 한다**는 점이 핵심입니다.

### 세션 기반(Session-level) vs. 트랜잭션 기반(Transaction-level) Advisory Lock

- **세션 기반 (Session-level)**

  - 명시적으로 `pg_advisory_unlock()`을 호출해야 해제됨
  - 세션이 종료되기 전까지 유지됨
  - 예: `pg_advisory_lock(12345)`

- **트랜잭션 기반 (Transaction-level)**
  - 트랜잭션이 종료되면 자동으로 해제됨
  - 명시적으로 해제할 필요 없음
  - 예: `pg_advisory_xact_lock(12345)`

### Advisory Lock의 특징과 장점

- 숫자 기반 잠금이므로 가볍고 빠름
- 원하는 키(key)를 직접 정의할 수 있어 유연함
- 기존 데이터(테이블/행)에 영향을 주지 않음
- 트랜잭션 기반을 사용하면 자동 해제가 가능하여 관리가 쉬움

---

## 3. Advisory Lock 사용 방법

### 3.1 세션 기반 Advisory Lock

```sql
-- 세션 수준 잠금 설정 (블로킹)
SELECT pg_advisory_lock(12345);

-- 잠금 해제
SELECT pg_advisory_unlock(12345);
```

이 방식은 **명시적으로 해제해야** 하며, 같은 세션에서만 유효합니다.

### 3.2 트랜잭션 기반 Advisory Lock

```sql
-- 트랜잭션 내에서만 유지되는 잠금
SELECT pg_advisory_xact_lock(12345);
```

트랜잭션이 끝나면 자동으로 해제되므로 따로 `pg_advisory_unlock()`을 호출할 필요가 없습니다.

### 3.3 비차단 (Non-blocking) 방식

잠금이 걸려 있을 경우 **즉시 반환**하도록 하려면 `pg_try_advisory_lock()`을 사용합니다.

```sql
SELECT pg_try_advisory_lock(12345);
```

- `true` 반환 → 성공적으로 잠금 획득
- `false` 반환 → 이미 다른 세션이 잠금을 사용 중

---

## 4. Advisory Lock 실습

다음은 두 개의 세션에서 Advisory Lock을 사용하여 동작하는 모습입니다.

| 시간 | 세션 A                            | 세션 B                                         |
| ---- | --------------------------------- | ---------------------------------------------- |
| 1초  | `SELECT pg_advisory_lock(100);`   |                                                |
| 2초  |                                   | `SELECT pg_advisory_lock(100); -- 블로킹 상태` |
| 3초  |                                   | `-- 여전히 대기 중...`                         |
| 4초  | `SELECT pg_advisory_unlock(100);` |                                                |
| 5초  |                                   | `-- 세션 A가 해제됨, 이제 실행됨`              |

---

## 5. Golang에서 Advisory Lock을 활용한 동시성 제어

아래는 Golang에서 PostgreSQL Advisory Lock을 이용하여 동시성을 제어하는 예제 코드입니다.

```go
package main

import (
  "context"
  "database/sql"
  "fmt"
  "log"
  "time"

  _ "github.com/lib/pq"
)

func acquireLock(db *sql.DB, lockID int) bool {
  var success bool
  err := db.QueryRow("SELECT pg_try_advisory_lock($1)", lockID).Scan(&success)
  if err != nil {
    log.Fatalf("Error acquiring lock: %v", err)
  }

  return success
}

func releaseLock(db *sql.DB, lockID int) {
  _, err := db.Exec("SELECT pg_advisory_unlock($1)", lockID)
  if err != nil {
    log.Fatalf("Error releasing lock: %v", err)
  }
}

func main() {
  connStr := "postgres://user:password@localhost:5432/mydb?sslmode=disable"
  db, err := sql.Open("postgres", connStr)
  if err != nil {
    log.Fatalf("Failed to connect to database: %v", err)
  }
  defer db.Close()

  lockID := 12345
  if acquireLock(db, lockID) {
    fmt.Println("Lock acquired, processing...")
    time.Sleep(5 * time.Second)
    releaseLock(db, lockID)
    fmt.Println("Lock released")
  } else {
    fmt.Println("Could not acquire lock, another process is using it.")
  }
}
```

이 코드는 `pg_try_advisory_lock()`을 사용하여 락을 획득하고, 성공하면 5초간 작업 후 락을 해제하는 간단한 예제입니다.
