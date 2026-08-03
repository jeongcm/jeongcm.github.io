---
title: "Apache Beam 2.75의 Delta Lake 읽기 지원을 데이터 파이프라인 경계로 보기"
date: 2026-08-02 11:19:00 +0900
categories: [data-engineering]
tags: [Apache Beam, Delta Lake, lakehouse, pipeline, data-engineering]
---

데이터 파이프라인에서 자주 생기는 마찰은 처리 엔진보다 저장소 경계에서 나온다. 분석 테이블은 Delta Lake에 있고, 처리 로직은 Apache Beam 모델로 표준화되어 있으며, 실행 환경은 Flink, Spark, Dataflow처럼 여러 runner로 나뉘는 경우다. Apache Beam 2.75.0은 Java SDK에 Delta Lake 읽기 지원을 추가하면서 이 경계를 조금 더 명시적인 Beam transform으로 다룰 수 있게 했다.

Beam 2.75.0 릴리스 노트의 I/O 항목은 Java에서 Delta Lake 읽기 지원이 추가되었다고 설명한다. 현재 Javadoc에는 `org.apache.beam.sdk.io.delta` 패키지가 Delta Lake를 읽기 위한 transform을 제공하며, `DeltaIO`는 Delta Lake table을 읽는 connector로 정리되어 있다. `DeltaReadSchemaTransformProvider`는 `DeltaIO.readRows()`를 schema transform으로 노출하고, 결과를 Beam `Row`의 `PCollection`으로 만든다. 즉 Delta transaction log와 Parquet 파일을 직접 애플리케이션 코드에서 흩어 읽는 대신, Beam pipeline 안의 입력 단계로 캡슐화하는 방향이다.

<figure class="post-diagram">
  <svg viewBox="0 0 860 300" role="img" aria-labelledby="beam-delta-title beam-delta-desc" xmlns="http://www.w3.org/2000/svg">
    <title id="beam-delta-title">Beam Delta Lake read boundary</title>
    <desc id="beam-delta-desc">Delta Lake transaction log and data files are read by DeltaIO, converted to Beam Rows, transformed, and written to downstream sinks by a runner.</desc>
    <defs>
      <marker id="arrow-beam-delta" markerWidth="10" markerHeight="10" refX="9" refY="5" orient="auto">
        <path d="M1,1 L9,5 L1,9 Z" fill="#475569" />
      </marker>
      <style>
        .outer { fill: #f8fafc; stroke: #cbd5e1; stroke-width: 2; rx: 8; }
        .lake { fill: #eef6ff; stroke: #3b82f6; stroke-width: 2; rx: 8; }
        .beam { fill: #ecfdf5; stroke: #10b981; stroke-width: 2; rx: 8; }
        .run { fill: #fff7ed; stroke: #f97316; stroke-width: 2; rx: 8; }
        .label { font: 700 16px -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; fill: #0f172a; }
        .small { font: 13px -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; fill: #475569; }
        .line { stroke: #475569; stroke-width: 3; fill: none; marker-end: url(#arrow-beam-delta); }
      </style>
    </defs>
    <rect x="24" y="28" width="812" height="244" class="outer" />
    <text x="54" y="64" class="label">Lakehouse read path as a Beam boundary</text>
    <rect x="58" y="112" width="180" height="88" class="lake" />
    <text x="148" y="144" text-anchor="middle" class="label">Delta Lake</text>
    <text x="148" y="168" text-anchor="middle" class="small">transaction log</text>
    <text x="148" y="190" text-anchor="middle" class="small">Parquet data files</text>
    <path d="M238 156 H326" class="line" />
    <rect x="326" y="98" width="206" height="116" class="beam" />
    <text x="429" y="132" text-anchor="middle" class="label">DeltaIO.readRows()</text>
    <text x="429" y="158" text-anchor="middle" class="small">snapshot-aware read</text>
    <text x="429" y="180" text-anchor="middle" class="small">Beam Row PCollection</text>
    <path d="M532 156 H620" class="line" />
    <rect x="620" y="112" width="178" height="88" class="run" />
    <text x="709" y="144" text-anchor="middle" class="label">Runner</text>
    <text x="709" y="168" text-anchor="middle" class="small">Flink / Spark / Dataflow</text>
    <text x="709" y="190" text-anchor="middle" class="small">sink or transform</text>
  </svg>
  <figcaption>Delta Lake의 저장 형식과 Beam의 처리 모델 사이에 읽기 transform을 둬서 pipeline 경계를 분리한다.</figcaption>
</figure>

이 변화가 중요한 이유는 lakehouse 테이블을 batch pipeline의 원천으로 다룰 때 코드의 책임을 줄여주기 때문이다. Delta Lake는 transaction log를 통해 table snapshot을 정의하고, 실제 데이터는 여러 data file에 나뉘어 있다. 처리 코드가 파일 목록과 snapshot 해석을 직접 다루면 재처리, schema 처리, runner별 분할 방식이 애플리케이션 로직으로 새기 쉽다. Beam transform으로 읽기 경계를 두면 이후 단계는 `PCollection<Row>`를 변환하는 일반 Beam 코드로 좁아진다.

다음은 설명을 위한 Java 개념 예시다. 실제 옵션 이름과 지원 범위는 사용하는 Beam 버전의 Javadoc과 release note를 기준으로 확인해야 한다.

```java
Pipeline pipeline = Pipeline.create(options);

PCollection<Row> events =
    pipeline.apply(
        "Read Delta table",
        DeltaIO.readRows()
            .fromTablePath("s3://analytics-lake/events_delta"));

events
    .apply("Keep successful events", Filter.by(row -> "SUCCESS".equals(row.getString("status"))))
    .apply("Write curated rows", SomeWarehouseIO.writeRows());

pipeline.run();
```

운영 관점에서는 세 가지를 먼저 확인해야 한다. 첫째, 이 기능은 2.75.0 기준 Java I/O로 소개되므로 Python이나 YAML pipeline에서 같은 표면을 기대하기 전에 cross-language transform 제공 여부를 확인해야 한다. 둘째, Beam 2.75.0은 Flink 2.1과 2.2 runner 지원을 추가했지만, Flink 2.1 이상과 Kafka client 계열 의존성을 함께 쓸 때 `lz4-java` capability 충돌이 발생할 수 있다고 알려져 있다. 셋째, Dataflow Runner v2 명칭이 Dataflow Portable Runner로 바뀌고 일부 Dataflow Streaming Engine 상태 태그 인코딩 기본값도 바뀌었으므로, runner 변경과 SDK 업그레이드를 같은 배포에 묶을 때는 rollback 가능성을 따로 검토해야 한다.

설계 기준은 간단하다. Delta Lake를 분석 저장소로 유지하면서 Beam으로 정제, 품질 검사, 재적재 pipeline을 운영하려면 `DeltaIO` 같은 공식 I/O transform을 먼저 검토한다. 반대로 streaming CDC 처리나 table commit을 직접 제어해야 하는 경우에는 Delta connector 하나로 모든 의미론이 해결된다고 보면 안 된다. 읽기 snapshot, schema evolution, runner 호환성, 의존성 충돌을 배포 체크리스트에 올려야 lakehouse와 portable pipeline의 장점을 함께 가져갈 수 있다.

## 참고 링크

- [Apache Beam 2.75.0 릴리스 발표](https://beam.apache.org/blog/beam-2.75.0/)
- [Apache Beam downloads - current release 2.75.0](https://beam.apache.org/get-started/downloads/)
- [Apache Beam Javadoc - org.apache.beam.sdk.io.delta](https://beam.apache.org/releases/javadoc/current/org/apache/beam/sdk/io/delta/package-summary.html)
- [Apache Beam Javadoc - DeltaReadSchemaTransformProvider](https://beam.apache.org/releases/javadoc/current/org/apache/beam/sdk/io/delta/DeltaReadSchemaTransformProvider.html)
