---
title: "PostgreSQL SKIP LOCKED: 작업 큐의 선점과 재실행을 설계하는 기준"
date: 2026-10-10 09:00:00 +0900
categories: [technical-knowledge, backend-knowledge]
tags: [PostgreSQL, concurrency, transaction, job-queue, skip-locked]
---

PostgreSQL의 `SKIP LOCKED`는 여러 워커가 잠긴 작업을 기다리지 않고 다른 행을 선점하게 한다.
선점 트랜잭션을 짧게 끝내려면 처리 중 상태를 저장하고, 장애 후 회수 규칙을 별도로 설계해야 한다.
행 잠금만으로 외부 API 호출의 중복 실행까지 막을 수는 없다.

## 잠긴 작업을 기다릴 것인가, 건너뛸 것인가

여러 워커가 같은 대기 테이블을 조회하면 동일한 작업을 가져갈 수 있다. `SELECT FOR UPDATE`는 선택한 행을 잠그고, `SKIP LOCKED`는 즉시 잠글 수 없는 행을 건너뛴다. `NOWAIT`가 잠금 충돌을 오류로 돌려주는 것과 구별된다.

공식 문서는 이 방식이 일관된 전체 조회에는 부적합하지만 큐 형태의 테이블에서 경합을 줄이는 데 유용하다고 설명한다. 잔액 검증이나 정산 조회에 적용하면 잠긴 행이 결과에서 빠질 수 있다. 아래 내용은 2026년 10월 10일 확인한 PostgreSQL 18 문서 기준의 기존 기능 해설이다. [SELECT의 잠금 절](https://www.postgresql.org/docs/18/sql-select.html#SQL-FOR-UPDATE-SHARE)

## 선점과 상태 변경을 한 트랜잭션에 묶는다

아래는 직접 작성한 **개념 예시**다. `jobs(id, state, claimed_at)` 테이블과 `id` 기본 키를 가정하며, 스키마 생성·회수·완료 처리는 생략했다. 기본 격리 수준인 Read Committed를 전제로 한다.

```sql
BEGIN;
WITH picked AS (
    SELECT id
    FROM jobs
    WHERE state = 'ready'
    ORDER BY id
    LIMIT 10
    FOR UPDATE SKIP LOCKED
)
UPDATE jobs AS j
SET state = 'running', claimed_at = CURRENT_TIMESTAMP
FROM picked
WHERE j.id = picked.id
RETURNING j.id;
COMMIT;
```

워커는 반환된 ID를 보관하고 커밋 성공을 확인한 뒤 실제 작업을 시작한다. 단순 조회와 상태 갱신 사이에서 잠금을 풀면 선점 경쟁이 다시 생긴다. 행 잠금은 트랜잭션 종료 때 풀리므로, 커밋 뒤에는 `running` 상태가 다음 선점 대상에서 제외하는 역할을 한다. [명시적 잠금](https://www.postgresql.org/docs/18/explicit-locking.html#LOCKING-ROWS)

<figure>
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 760 260" role="img" aria-labelledby="queue-title queue-desc" style="width:100%;height:auto">
<title id="queue-title">두 워커의 행 선점과 커밋 이후 처리</title>
<desc id="queue-desc">워커 A가 행 1을 잠그면 워커 B는 행 1을 건너뛰고 행 2를 잠근다. 각자 running 상태를 커밋한 뒤 외부 작업을 수행한다.</desc>
<rect width="760" height="260" rx="16" fill="#f8fafc"/>
<g font-family="sans-serif" text-anchor="middle" fill="#0f172a">
<text x="380" y="30" font-size="18" font-weight="bold">짧은 선점 트랜잭션 → 커밋 → 실제 작업</text>
<rect x="20" y="55" width="210" height="135" rx="10" fill="#dbeafe"/>
<text x="125" y="86" font-size="17">행 잠금 획득</text>
<text x="125" y="119" font-size="15">A: 행 1 잠금</text>
<text x="125" y="151" font-size="15">B: 1 건너뛰고 2 잠금</text>
<rect x="280" y="55" width="200" height="135" rx="10" fill="#fef3c7"/>
<text x="380" y="86" font-size="17">running 저장</text>
<text x="380" y="119" font-size="15">각 트랜잭션 커밋</text>
<text x="380" y="151" font-size="15">행 잠금 해제</text>
<rect x="530" y="55" width="210" height="135" rx="10" fill="#d1fae5"/>
<text x="635" y="86" font-size="17">외부 작업 수행</text>
<text x="635" y="119" font-size="15">성공 후 완료 기록</text>
<text x="635" y="151" font-size="15">중단 시 별도 회수</text>
<text x="380" y="230" font-size="16">DB 선점의 배타성과 외부 효과의 중복 방지는 별도 문제</text>
</g>
<g stroke="#475569" stroke-width="3" fill="none"><path d="M237 120 H267 M257 111 L267 120 L257 129"/><path d="M487 120 H517 M507 111 L517 120 L507 129"/></g>
</svg>
<figcaption>직접 작성한 개념도. 배치 크기를 1로 줄여 두 워커의 선점 과정을 표현했다.</figcaption>
</figure>

## 커밋 이후의 실패는 별도 문제다

다음은 이 잠금 동작에서 도출한 설계 기준이다. 커밋 직후 워커가 죽으면 작업이 `running`에 남는다. 소유자와 만료 시각, heartbeat를 두고 회수할 수 있지만, 느린 기존 워커가 계속 실행 중일 수도 있다. 회수 후에는 새 소유권 토큰과 일치하는 워커만 완료 상태를 쓰도록 조건부 갱신한다.

이 조건은 DB 상태 보호용이다. 외부 API가 성공한 직후 완료 기록 전에 죽으면 재실행이 같은 외부 효과를 만들 수 있다. 수신 측 멱등성 키나 중복 제거 계약도 필요하다. 만료 시간만으로 중복 실행이 사라지지는 않는다.

## 적용 범위와 운영 지표

소규모 비동기 작업을 기존 DB에서 처리할 때는 검토할 만하다. 엄격한 FIFO, 대규모 팬아웃, 장기 재생이 필요하면 전용 메시징 시스템과 비교한다. `ORDER BY id`가 있어도 잠긴 앞 행을 건너뛰므로 전역 처리 순서는 보장되지 않는다.

`SKIP LOCKED`는 행 잠금에 적용되며 테이블 잠금 대기까지 없애지 않는다. 외부 호출 동안 트랜잭션을 열어두지 말고, 배치 크기와 동시 워커 수를 제한한다. 대기 작업의 최대 나이, 오래된 `running`, 회수·재시도 횟수, 선점 쿼리 지연을 함께 관찰한다. 빈 조회 결과도 전체 큐가 비었다는 확정 신호로 해석하지 않는다.

## 참고 링크

- [PostgreSQL 18 SELECT — SKIP LOCKED와 잠금 절](https://www.postgresql.org/docs/18/sql-select.html#SQL-FOR-UPDATE-SHARE)
- [PostgreSQL 18 Explicit Locking — 행 잠금의 수명](https://www.postgresql.org/docs/18/explicit-locking.html#LOCKING-ROWS)
- [PostgreSQL 18 Transaction Isolation — Read Committed](https://www.postgresql.org/docs/18/transaction-iso.html#XACT-READ-COMMITTED)
