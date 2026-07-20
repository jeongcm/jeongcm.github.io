---
title: "Spark 4.2의 CDC와 Auto CDC를 어떻게 볼 것인가"
date: 2026-07-20 14:28:08 +0900
categories: [data-engineering]
tags: [spark, cdc, lakehouse, declarative-pipelines, streaming]
---

Apache Spark 4.2.0은 Spark SQL, Spark Connect, Structured Streaming, PySpark 개선과 함께 데이터 파이프라인 관점에서 눈에 띄는 변화도 포함한다. 특히 `CHANGES` 절을 통한 Change Data Capture(CDC) 지원과 Declarative Pipelines의 Auto CDC API는 레이크하우스 테이블을 단순 조회 대상이 아니라 변경 흐름의 입력으로 다루려는 방향을 보여준다.

## 오늘 본 변화의 핵심

Spark 4.2.0 릴리스 노트는 Spark SQL의 새 기능 중 하나로 `CHANGES` 절을 소개한다. 이는 테이블의 변경분을 쿼리할 수 있게 하는 CDC 기능이다. 같은 릴리스에서 Declarative Pipelines는 Auto CDC API를 추가했다. 파이프라인 작성자는 전체 테이블을 매번 다시 계산하는 방식 대신, 변경 이벤트를 기준으로 downstream 테이블을 갱신하는 모델을 선택할 수 있다.

데이터 플랫폼에서 CDC는 보통 Debezium, Kafka, Flink, Iceberg 같은 도구 조합으로 설명된다. Spark 4.2의 의미는 Spark가 이 흐름 전체를 대체한다는 뜻이 아니라, 배치/스트리밍/SQL 중심의 Spark 작업 안에서도 변경 데이터가 더 직접적인 설계 요소가 된다는 점에 있다.

<svg viewBox="0 0 760 270" role="img" aria-label="Spark 4.2 CDC 흐름" xmlns="http://www.w3.org/2000/svg">
  <rect width="760" height="270" rx="14" fill="#111827"/>
  <text x="34" y="42" fill="#f9fafb" font-size="22" font-family="Arial, sans-serif" font-weight="700">CDC 중심 Spark 파이프라인</text>
  <g font-family="Arial, sans-serif" font-size="14">
    <rect x="34" y="86" width="150" height="76" rx="10" fill="#1f2937" stroke="#60a5fa"/>
    <text x="60" y="118" fill="#e5e7eb">Source Table</text>
    <text x="58" y="142" fill="#93c5fd">insert/update/delete</text>
    <path d="M194 124 H292" stroke="#9ca3af" stroke-width="3" marker-end="url(#arrow)"/>
    <rect x="302" y="86" width="156" height="76" rx="10" fill="#1f2937" stroke="#34d399"/>
    <text x="330" y="118" fill="#e5e7eb">CHANGES Query</text>
    <text x="335" y="142" fill="#a7f3d0">changed rows only</text>
    <path d="M468 124 H566" stroke="#9ca3af" stroke-width="3" marker-end="url(#arrow)"/>
    <rect x="576" y="86" width="150" height="76" rx="10" fill="#1f2937" stroke="#fbbf24"/>
    <text x="604" y="118" fill="#e5e7eb">Auto CDC</text>
    <text x="598" y="142" fill="#fde68a">apply to targets</text>
    <rect x="302" y="188" width="156" height="46" rx="10" fill="#0f172a" stroke="#475569"/>
    <text x="326" y="216" fill="#cbd5e1">watermark / ordering</text>
    <path d="M380 162 V184" stroke="#64748b" stroke-width="2" marker-end="url(#arrow)"/>
  </g>
  <defs>
    <marker id="arrow" markerWidth="10" markerHeight="10" refX="8" refY="3" orient="auto">
      <path d="M0,0 L0,6 L9,3 z" fill="#9ca3af"/>
    </marker>
  </defs>
</svg>

## 왜 실무적으로 중요한가

전체 재처리는 단순하지만 비용과 지연 시간이 커진다. 특히 주문, 결제, 계정, 재고처럼 변경이 잦은 도메인에서는 변경분만 읽고 필요한 테이블만 갱신하는 방식이 더 자연스럽다. Spark SQL의 CDC 지원은 분석 SQL과 파이프라인 SQL 사이의 간격을 줄인다. Declarative Pipelines의 Auto CDC는 변경 이벤트를 대상 테이블에 반영하는 반복 코드를 줄이는 방향이다.

다만 CDC는 "변경분을 읽는다"만으로 끝나지 않는다. update와 delete의 순서, 중복 이벤트, late arrival, snapshot과 incremental 구간의 경계, target table의 idempotency가 함께 설계돼야 한다. Auto CDC API를 쓰더라도 기본 키, 정렬 기준, 삭제 처리 정책을 명확히 하지 않으면 downstream 테이블의 신뢰도를 보장하기 어렵다.

## 개념 예시

아래 SQL은 문법 검증용 완성 예제가 아니라, Spark 4.2에서 기대할 수 있는 CDC 설계 형태를 보여주는 개념 예시다.

```sql
-- 개념 예시: 변경분을 기준으로 downstream 테이블을 갱신하는 흐름
CREATE OR REFRESH STREAMING TABLE order_changes AS
SELECT *
FROM CHANGES(order_events, START => '2026-07-20T00:00:00Z');

CREATE OR REFRESH MATERIALIZED VIEW daily_order_summary AS
SELECT
  order_date,
  count(*) AS order_count,
  sum(order_amount) AS total_amount
FROM order_changes
WHERE change_type <> 'delete'
GROUP BY order_date;
```

핵심은 CDC를 "빠른 재계산" 정도로만 보지 않는 것이다. 변경 로그는 사실상 데이터 계약의 일부가 된다. 어떤 컬럼이 변경 이벤트에 포함되는지, delete가 tombstone인지 hard delete인지, 같은 key의 여러 변경이 어떤 순서로 적용되는지를 소비자가 알아야 한다.

## 적용 시 주의할 점

첫째, CDC를 도입하기 전 원천 테이블의 변경 보존 기간을 확인해야 한다. 변경 로그가 짧게 보존되면 장기 장애 후 재시작이나 backfill이 어려워진다.

둘째, 변경 이벤트의 순서 기준을 명시해야 한다. 이벤트 시간, 커밋 시간, 버전 번호가 섞이면 결과 테이블이 재실행마다 달라질 수 있다.

셋째, CDC 기반 파이프라인도 모니터링 대상이다. 변경량 급증, delete 비율 증가, 지연 이벤트 누적은 단순 처리량보다 중요한 운영 신호가 될 수 있다.

Spark 4.2의 CDC 관련 기능은 Spark를 스트리밍 엔진 하나로 보는 관점보다, SQL 기반 레이크하우스 파이프라인의 변경 처리 계층으로 보는 관점에서 더 의미가 크다.

## 참고 링크

- [Apache Spark 4.2.0 release notes](https://spark.apache.org/releases/spark-release-4-2-0.html)
- [Apache Spark 4.2.0 released](https://spark.apache.org/news/spark-4-2-0-released.html)
- [Spark Declarative Pipelines Programming Guide](https://spark.apache.org/docs/latest/declarative-pipelines-programming-guide.html)
- [Spark Structured Streaming Programming Guide](https://spark.apache.org/docs/latest/streaming/index.html)
