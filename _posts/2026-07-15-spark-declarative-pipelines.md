---
title: "Spark Declarative Pipelines는 데이터 파이프라인 설계를 어떻게 바꾸나"
date: 2026-07-15 09:03:50 +0900
categories: [data-engineering]
tags: [apache-spark, data-pipeline, structured-streaming, spark-41, orchestration]
---

Apache Spark 4.1.0에서 눈에 띄는 변화 중 하나는 Spark Declarative Pipelines(SDP)다. Spark 릴리스 노트는 SDP를 데이터셋과 쿼리를 정의하면 실행 그래프, 의존성 순서, 병렬성, 체크포인트, 재시도를 Spark가 처리하는 선언형 프레임워크로 설명한다. 이는 Spark를 단순한 실행 엔진으로만 쓰는 방식에서 한 걸음 더 나아가, 파이프라인 정의와 실행 제어 일부를 Spark 생태계 안으로 가져오는 변화다.

기존 Spark 파이프라인은 보통 Airflow, Dagster, Argo Workflows 같은 외부 오케스트레이터가 작업 순서를 관리하고, 각 작업 내부에서 Spark SQL 또는 DataFrame 코드를 실행하는 형태가 많았다. 이 구조는 범용성이 높지만, 데이터셋 간 의존성, 증분 처리, 체크포인트 위치, 재처리 범위를 여러 계층에 나눠 관리해야 한다. SDP는 `flow`, `dataset`, `pipeline`을 명시적으로 정의하게 해 이 경계를 좁힌다.

<svg viewBox="0 0 760 250" role="img" aria-label="Spark Declarative Pipelines flow diagram" xmlns="http://www.w3.org/2000/svg">
  <rect x="24" y="30" width="150" height="70" rx="8" fill="#e8f3ff" stroke="#3178c6"/>
  <text x="99" y="60" text-anchor="middle" font-size="15" font-family="sans-serif">Source</text>
  <text x="99" y="82" text-anchor="middle" font-size="12" font-family="sans-serif">Kafka / S3 / Table</text>
  <rect x="238" y="30" width="150" height="70" rx="8" fill="#f2f7e8" stroke="#6a994e"/>
  <text x="313" y="60" text-anchor="middle" font-size="15" font-family="sans-serif">Flow</text>
  <text x="313" y="82" text-anchor="middle" font-size="12" font-family="sans-serif">Transform logic</text>
  <rect x="452" y="30" width="150" height="70" rx="8" fill="#fff2d8" stroke="#d08c00"/>
  <text x="527" y="60" text-anchor="middle" font-size="15" font-family="sans-serif">Dataset</text>
  <text x="527" y="82" text-anchor="middle" font-size="12" font-family="sans-serif">Streaming table / MV</text>
  <rect x="300" y="150" width="170" height="70" rx="8" fill="#f7e8ff" stroke="#8e44ad"/>
  <text x="385" y="180" text-anchor="middle" font-size="15" font-family="sans-serif">Spark Pipeline</text>
  <text x="385" y="202" text-anchor="middle" font-size="12" font-family="sans-serif">order, checkpoint, retry</text>
  <path d="M174 65 H238" stroke="#555" stroke-width="2" marker-end="url(#arrow)"/>
  <path d="M388 65 H452" stroke="#555" stroke-width="2" marker-end="url(#arrow)"/>
  <path d="M527 100 C527 150 470 172 470 185" fill="none" stroke="#555" stroke-width="2" marker-end="url(#arrow)"/>
  <path d="M313 100 C313 145 300 160 300 185" fill="none" stroke="#555" stroke-width="2" marker-end="url(#arrow)"/>
  <defs>
    <marker id="arrow" markerWidth="10" markerHeight="10" refX="8" refY="3" orient="auto">
      <path d="M0,0 L0,6 L9,3 z" fill="#555"/>
    </marker>
  </defs>
</svg>

공식 문서 기준으로 SDP의 핵심 단위는 세 가지다. `flow`는 소스에서 읽고 변환한 뒤 대상 데이터셋에 쓰는 처리 단위다. `dataset`은 파이프라인 안에서 조회 가능한 산출물이며, 스트리밍 테이블, 머티리얼라이즈드 뷰, 임시 뷰로 나뉜다. `pipeline`은 이런 정의들을 분석해 실행 순서와 병렬화를 정한다. 특히 Spark 4.1 계열 문서는 SDP가 배치와 스트리밍 모두를 대상으로 하며, Kafka 같은 메시지 버스와 클라우드 스토리지 ingestion을 주요 사용 사례로 둔다고 설명한다.

실무적으로 중요한 지점은 오케스트레이터를 완전히 대체하느냐가 아니다. SDP가 유리한 영역은 Spark 안에서 데이터셋 의존성이 촘촘하고, 배치와 스트리밍 변환을 같은 모델로 관리하려는 경우다. 예를 들어 원천 이벤트를 스트리밍 테이블로 수집하고, 고객/상품 같은 기준 데이터를 머티리얼라이즈드 뷰로 붙인 뒤, 집계 테이블을 순차적으로 갱신하는 구조에서는 파이프라인 그래프를 코드와 가까운 곳에 둘 수 있다.

반대로 여러 시스템을 호출하는 승인 워크플로, 장시간 외부 API 연동, Spark 외부 작업이 많은 플랫폼에서는 기존 오케스트레이터가 여전히 필요하다. SDP는 Spark 내부 데이터 처리 그래프를 깔끔하게 만드는 도구이지, 조직 전체의 워크플로 엔진으로 확대 해석하면 안 된다. 체크포인트 저장소, 카탈로그/데이터베이스 경계, full refresh와 incremental refresh의 운영 정책도 별도로 정해야 한다.

개념 예시는 아래처럼 읽을 수 있다. 핵심은 "어떻게 실행할지"보다 "어떤 데이터셋이 어떤 입력에서 만들어지는지"를 먼저 선언하는 것이다.

```python
# concept example
from pyspark import pipelines as dp
from pyspark.sql.functions import col

@dp.table
def orders_raw():
    return spark.readStream.table("source_orders")

@dp.materialized_view
def daily_order_amount():
    return (
        spark.table("orders_raw")
        .groupBy(col("order_date"))
        .sum("amount")
    )
```

Spark 4.1.0은 SDP 외에도 Structured Streaming Real-Time Mode, RocksDB state store 개선, SQL Scripting GA, VARIANT GA 같은 변화를 함께 담고 있다. 따라서 업그레이드 검토 시에는 기능 목록을 나열하기보다, 파이프라인 정의를 Spark 안으로 얼마나 가져올지, 낮은 지연 시간이 실제 요구사항인지, 기존 오케스트레이터와 책임을 어떻게 나눌지를 먼저 정하는 편이 안전하다.

## 참고 링크

- [Apache Spark 4.1.0 release notes](https://spark.apache.org/releases/spark-release-4.1.0.html)
- [Spark Declarative Pipelines Programming Guide](https://spark.apache.org/docs/latest/declarative-pipelines-programming-guide.html)
- [Structured Streaming Programming Guide](https://spark.apache.org/docs/latest/streaming/index.html)
