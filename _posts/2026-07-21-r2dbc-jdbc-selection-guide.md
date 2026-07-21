---
title: "R2DBC와 JDBC 선택 기준: 논블로킹만으로 결정하지 않기"
date: 2026-07-21 09:26:54 +0900
categories: [technical-knowledge, backend-knowledge]
tags: [spring, r2dbc, jdbc, webflux, transaction]
---

Spring 기반 백엔드에서 R2DBC와 JDBC 선택은 "WebFlux면 R2DBC, MVC면 JDBC"처럼 단순하게 끝나지 않는다. R2DBC는 Reactive Streams 기반으로 SQL 데이터베이스에 접근하는 SPI이고, JDBC는 오래 검증된 명령형 데이터 접근 모델이다. 중요한 판단 기준은 API의 모양보다 요청 처리 모델, 트랜잭션 범위, 드라이버 성숙도, 운영 복잡도다.

## 오늘 다룰 백엔드 판단 문제

R2DBC의 핵심은 관계형 데이터베이스 접근을 `Publisher` 기반으로 지연 실행하고, 구독 시점에 실제 SQL을 실행하며, backpressure-aware 흐름에 맞춘다는 점이다. Spring Data R2DBC도 `R2dbcEntityTemplate`과 repository 지원을 제공하지만, 터미널 메서드는 `Mono`나 `Flux`를 반환하고 실제 DB 작업은 subscription 이후에 수행된다.

반대로 JDBC는 스레드가 DB 호출을 기다리는 명령형 모델이다. Spring Framework의 JDBC 지원은 `JdbcClient`, `JdbcTemplate` 같은 추상화로 반복 코드를 줄이고 예외 변환, 리소스 관리를 제공한다. blocking I/O라는 특성은 단점이 될 수 있지만, 트랜잭션, 드라이버, 툴링, 운영 사례가 넓게 축적되어 있다는 강점도 있다.

<svg viewBox="0 0 760 310" role="img" aria-label="R2DBC와 JDBC 선택 기준" xmlns="http://www.w3.org/2000/svg">
  <rect width="760" height="310" rx="14" fill="#111827"/>
  <text x="34" y="42" fill="#f9fafb" font-size="22" font-family="Arial, sans-serif" font-weight="700">R2DBC vs JDBC 판단 흐름</text>
  <g font-family="Arial, sans-serif" font-size="14">
    <rect x="44" y="86" width="190" height="78" rx="10" fill="#1f2937" stroke="#60a5fa"/>
    <text x="74" y="118" fill="#e5e7eb">요청 처리 모델</text>
    <text x="65" y="142" fill="#bfdbfe">Reactive end-to-end?</text>
    <path d="M244 125 H332" stroke="#9ca3af" stroke-width="3" marker-end="url(#arrow)"/>
    <rect x="342" y="72" width="170" height="58" rx="10" fill="#0f172a" stroke="#34d399"/>
    <text x="384" y="106" fill="#d1fae5">R2DBC 후보</text>
    <rect x="342" y="154" width="170" height="58" rx="10" fill="#0f172a" stroke="#fbbf24"/>
    <text x="386" y="188" fill="#fde68a">JDBC 후보</text>
    <path d="M522 101 H620" stroke="#64748b" stroke-width="3" marker-end="url(#arrow)"/>
    <path d="M522 183 H620" stroke="#64748b" stroke-width="3" marker-end="url(#arrow)"/>
    <rect x="630" y="82" width="86" height="122" rx="10" fill="#1f2937" stroke="#a78bfa"/>
    <text x="650" y="116" fill="#ede9fe">검증</text>
    <text x="647" y="145" fill="#c4b5fd">driver</text>
    <text x="646" y="169" fill="#c4b5fd">tx scope</text>
    <text x="651" y="193" fill="#c4b5fd">ops</text>
    <rect x="84" y="226" width="590" height="42" rx="10" fill="#0f172a" stroke="#475569"/>
    <text x="109" y="252" fill="#cbd5e1">선택은 논블로킹 여부 + 트랜잭션 경계 + 드라이버/운영 성숙도를 함께 본다</text>
  </g>
  <defs>
    <marker id="arrow" markerWidth="10" markerHeight="10" refX="8" refY="3" orient="auto">
      <path d="M0,0 L0,6 L9,3 z" fill="#9ca3af"/>
    </marker>
  </defs>
</svg>

## 언제 R2DBC가 적합한가

R2DBC는 애플리케이션의 상단부터 하단까지 reactive 흐름을 유지할 때 의미가 크다. WebFlux handler, Reactor 기반 외부 호출, 비동기 메시징, R2DBC driver가 같은 흐름 안에서 backpressure와 non-blocking I/O를 유지한다면 스레드 점유를 줄일 수 있다. 요청 수가 많고 DB 호출 대기 시간이 길며, 각 요청이 짧은 쿼리 중심으로 구성되는 서비스에서는 이 장점이 드러난다.

다만 R2DBC는 JDBC의 비동기 버전이 아니다. 드라이버별 기능 차이, SQL dialect 차이, 트랜잭션 전파 방식, connection pool 관측 지표를 별도로 확인해야 한다. JPA의 영속성 컨텍스트, lazy loading, 복잡한 ORM 기능을 기대하는 구조라면 R2DBC로 옮기는 순간 애플리케이션 설계가 같이 바뀐다.

## 언제 JDBC가 더 나은가

JDBC는 blocking 모델이지만 단순한 CRUD, 복잡한 트랜잭션, 기존 JPA/JdbcTemplate 기반 코드, 검증된 드라이버 기능이 중요한 서비스에 적합하다. 특히 요청 처리량보다 데이터 정합성, 운영 예측 가능성, 팀의 디버깅 경험이 더 중요한 내부 API나 관리성 작업에서는 JDBC가 더 실용적인 선택일 수 있다.

WebFlux 애플리케이션에서도 JDBC를 완전히 금지해야 하는 것은 아니다. 다만 event loop에서 blocking JDBC를 직접 호출하면 안 된다. 필요한 경우 별도 scheduler나 worker pool로 격리해야 하고, 그 선택이 thread pool, connection pool, timeout을 두 겹으로 관리하는 비용을 만든다는 점을 계산해야 한다.

## 개념 예시

아래 코드는 완성된 운영 코드가 아니라, 선택 기준을 드러내기 위한 개념 예시다.

```kotlin
// 개념 예시: reactive 흐름을 끝까지 유지할 때 R2DBC가 자연스럽다.
fun findOrder(id: String): Mono<OrderView> =
    r2dbcTemplate.selectOne(
        query(where("id").`is`(id)),
        OrderRow::class.java
    ).map { row -> OrderView(row.id, row.status) }

// 개념 예시: 명령형 트랜잭션과 기존 JDBC 생태계가 중요하면 JDBC가 단순하다.
@Transactional
fun markPaid(id: String) {
    jdbcClient.sql("update orders set status = ? where id = ?")
        .params("PAID", id)
        .update()
}
```

## 운영 시 주의할 점

첫째, R2DBC를 도입할 때는 "논블로킹"보다 "blocking 호출이 남아 있는가"를 먼저 확인해야 한다. 같은 흐름 안에 blocking HTTP client, 파일 I/O, JDBC 호출이 섞이면 R2DBC의 장점은 쉽게 줄어든다.

둘째, 트랜잭션 경계를 명확히 해야 한다. reactive transaction은 subscription과 context 전파에 민감하다. 중간에 `block()`을 호출하거나 다른 실행 모델로 넘어가면 예상한 트랜잭션 경계가 깨질 수 있다.

셋째, pool 크기와 timeout을 별도로 검증해야 한다. R2DBC는 스레드를 덜 점유할 수 있지만 DB connection이 무한히 증가하는 것은 아니다. connection pool, statement timeout, lock wait timeout, slow query 지표는 JDBC와 동일하게 운영 기준에 포함되어야 한다.

결론적으로 R2DBC는 reactive stack의 완성도를 높이는 선택이고, JDBC는 검증된 명령형 데이터 접근을 단순하게 유지하는 선택이다. 둘 중 하나가 항상 우월한 것이 아니라, 서비스의 호출 모델과 트랜잭션 복잡도에 맞춰 선택해야 한다.

## 참고 링크

- [R2DBC 1.0 Specification](https://r2dbc.io/spec/1.0.0.RELEASE/spec/html/)
- [Spring Framework: Data Access with R2DBC](https://docs.spring.io/spring-framework/reference/data-access/r2dbc.html)
- [Spring Framework: Data Access with JDBC](https://docs.spring.io/spring-framework/reference/data-access/jdbc.html)
- [Spring Framework: Transaction Management](https://docs.spring.io/spring-framework/reference/data-access/transaction.html)
- [Spring Data Relational: R2DBC](https://docs.spring.io/spring-data/relational/reference/r2dbc.html)
