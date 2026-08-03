---
title: "트랜잭션 격리 수준과 재시도 경계를 정하는 기준"
date: 2026-08-01 09:09:00 +0900
categories: [technical-knowledge, backend-knowledge]
tags: [spring, transaction, postgresql, concurrency, backend]
---

트랜잭션은 `@Transactional`을 붙이면 완료되는 기능이 아니다. 서비스 메서드의 실행 범위, 데이터베이스 격리 수준, 예외와 재시도 정책이 함께 맞아야 동시성 상황에서도 의도한 결과를 유지할 수 있다. 특히 주문, 정산, 재고, 포인트처럼 같은 행이나 같은 집합을 여러 요청이 동시에 갱신하는 도메인에서는 "어디까지를 한 트랜잭션으로 묶을 것인가"와 "실패 시 어디서 다시 시도할 것인가"가 핵심 판단 지점이 된다.

Spring Framework의 선언적 트랜잭션은 AOP proxy와 `TransactionManager`가 메서드 호출 주변에 트랜잭션 경계를 만드는 방식이다. 기본 설정의 `@Transactional`은 `PROPAGATION_REQUIRED`, `ISOLATION_DEFAULT`, read-write 트랜잭션이며, 별도 rollback 규칙이 없으면 `RuntimeException`과 `Error`에서 rollback한다. checked exception까지 rollback 대상으로 삼아야 한다면 `rollbackFor` 같은 규칙을 명시해야 한다.

<figure class="post-diagram">
  <svg viewBox="0 0 860 320" role="img" aria-labelledby="tx-title tx-desc" xmlns="http://www.w3.org/2000/svg">
    <title id="tx-title">Transaction boundary, isolation, and retry flow</title>
    <desc id="tx-desc">A request enters a service transaction, reads and writes through a database isolation level, and retries the whole unit only after serialization failure.</desc>
    <defs>
      <marker id="arrow-tx" markerWidth="10" markerHeight="10" refX="9" refY="5" orient="auto">
        <path d="M1,1 L9,5 L1,9 Z" fill="#475569" />
      </marker>
      <style>
        .box { fill: #f8fafc; stroke: #cbd5e1; stroke-width: 2; rx: 8; }
        .svc { fill: #eef6ff; stroke: #3b82f6; stroke-width: 2; rx: 8; }
        .db { fill: #ecfdf5; stroke: #10b981; stroke-width: 2; rx: 8; }
        .warn { fill: #fff7ed; stroke: #f97316; stroke-width: 2; rx: 8; }
        .label { font: 700 16px -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; fill: #0f172a; }
        .small { font: 13px -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; fill: #475569; }
        .line { stroke: #475569; stroke-width: 3; fill: none; marker-end: url(#arrow-tx); }
      </style>
    </defs>
    <rect x="24" y="28" width="812" height="264" class="box" />
    <text x="54" y="65" class="label">Transaction design boundary</text>
    <rect x="62" y="118" width="158" height="84" class="svc" />
    <text x="141" y="151" text-anchor="middle" class="label">Request</text>
    <text x="141" y="176" text-anchor="middle" class="small">idempotent command</text>
    <path d="M220 160 H312" class="line" />
    <rect x="312" y="92" width="230" height="136" class="svc" />
    <text x="427" y="128" text-anchor="middle" class="label">Service transaction</text>
    <text x="427" y="154" text-anchor="middle" class="small">business invariant</text>
    <text x="427" y="178" text-anchor="middle" class="small">rollback rule</text>
    <text x="427" y="202" text-anchor="middle" class="small">timeout boundary</text>
    <path d="M542 160 H634" class="line" />
    <rect x="634" y="96" width="158" height="128" class="db" />
    <text x="713" y="132" text-anchor="middle" class="label">Database</text>
    <text x="713" y="158" text-anchor="middle" class="small">isolation level</text>
    <text x="713" y="182" text-anchor="middle" class="small">locks, MVCC</text>
    <path d="M640 242 C540 292 322 292 224 214" class="line" />
    <rect x="306" y="248" width="238" height="34" class="warn" />
    <text x="425" y="270" text-anchor="middle" class="small">Retry the whole transaction after serialization failure.</text>
  </svg>
  <figcaption>재시도는 SQL 한 문장이나 부분 로직이 아니라, 불변식을 판단한 트랜잭션 단위 전체에 걸어야 한다.</figcaption>
</figure>

격리 수준은 Spring 설정만으로 판단하면 안 된다. 실제 보장은 데이터베이스가 제공한다. PostgreSQL 문서는 `READ COMMITTED`를 각 statement가 시작되기 전에 commit된 행만 보는 기본 수준으로 설명한다. `REPEATABLE READ`는 트랜잭션의 첫 쿼리 시점 snapshot을 유지하며, PostgreSQL에서는 phantom read도 허용하지 않는다. `SERIALIZABLE`은 동시에 commit된 트랜잭션들의 결과가 어떤 직렬 실행 순서와도 맞지 않으면 하나를 실패시켜 정합성을 지킨다.

따라서 기본값을 무조건 높이는 방식은 좋은 기준이 아니다. 단순 조회와 단일 행 갱신은 `READ COMMITTED`와 명시적인 조건절, unique constraint, optimistic locking으로 충분한 경우가 많다. 반대로 여러 행의 합계, 존재 여부, 상태 전이 조건을 읽고 그 결과로 새 행을 쓰는 로직은 `REPEATABLE READ`나 `SERIALIZABLE`, 또는 명시적 lock을 검토해야 한다. 격리 수준이 높아질수록 개발자는 덜 복잡한 불변식 검증을 기대할 수 있지만, serialization failure와 retry 비용을 받아들여야 한다.

다음은 설명을 위한 Kotlin 개념 예시다. 핵심은 재시도가 트랜잭션 내부의 일부 쿼리가 아니라 서비스 명령 전체를 다시 실행한다는 점이다.

```kotlin
class PaymentService(
    private val repository: PaymentRepository
) {
    @Transactional(
        isolation = Isolation.SERIALIZABLE,
        rollbackFor = [PaymentConflictException::class]
    )
    fun approve(command: ApprovePaymentCommand): PaymentResult {
        val payment = repository.findForDecision(command.paymentId)
        payment.assertApprovable(command.amount)
        repository.markApproved(payment.id, command.requestId)
        return PaymentResult(payment.id, "APPROVED")
    }
}
```

운영 관점에서는 세 가지를 분리해야 한다. 첫째, 트랜잭션 timeout은 외부 API 호출 대기 시간을 감추는 장치가 아니다. DB connection을 잡은 상태에서 긴 네트워크 호출을 섞으면 lock 보유 시간이 늘고 pool 고갈로 번질 수 있다. 둘째, retry는 serialization failure, deadlock, transient connection 오류처럼 다시 시도할 의미가 있는 실패에 제한해야 한다. 비즈니스 충돌이나 검증 실패를 retry하면 부하만 키운다. 셋째, reactive transaction은 ThreadLocal이 아니라 Reactor context를 사용하므로 같은 reactive pipeline 안에서 참여해야 한다. Spring 문서도 `ReactiveTransactionManager` 사용 시 트랜잭션 메서드가 reactive pipeline을 반환해야 한다고 설명한다.

트랜잭션 설계의 좋은 출발점은 "데이터 불변식이 깨질 수 있는 읽기-쓰기 범위"를 먼저 찾는 것이다. 그 범위를 서비스 트랜잭션으로 묶고, 데이터베이스 격리 수준과 constraint로 보강한 뒤, 재시도 가능한 실패만 경계 바깥에서 전체 명령으로 다시 실행해야 한다.

## 참고 링크

- [Spring Framework - Declarative Transaction Management](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative.html)
- [Spring Framework - Understanding Declarative Transaction Implementation](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-decl-explained.html)
- [Spring Framework `@Transactional` API](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/annotation/Transactional.html)
- [Spring Framework - Programmatic Transaction Management](https://docs.spring.io/spring-framework/reference/data-access/transaction/programmatic.html)
- [PostgreSQL 18 - Transaction Isolation](https://www.postgresql.org/docs/18/transaction-iso.html)
