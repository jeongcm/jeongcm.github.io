---
title: "Kafka 설계 패턴은 왜 벤치마크와 같이 봐야 할까"
date: 2026-07-11 09:35:00 +0900
categories: [data-engineering]
tags: [apache-kafka, event-streaming, benchmarking, cdc, exactly-once]
---

Kafka 기반 이벤트 스트리밍 시스템을 설계할 때 자주 등장하는 말이 있다. log compaction, CDC, exactly-once pipeline, stream-table join, event sourcing, tiered storage 같은 패턴들이다. 이 표현들은 익숙하지만, 실제 시스템에 적용할 때는 이름만으로 충분하지 않다. 같은 패턴이라도 topic 수, partition 수, 메시지 크기, ack 설정, consumer 처리 방식, state store 사용 여부에 따라 성능과 장애 양상이 크게 달라지기 때문이다.

최근 arXiv에 공개된 Kafka 설계 패턴과 벤치마크 연구는 이 지점을 잘 짚는다. 이 논문은 2015년부터 2025년까지의 peer-reviewed 연구 42개를 분석해 Kafka 이벤트 스트리밍에서 반복적으로 등장하는 9가지 패턴을 정리한다. 여기에는 log compaction, CQRS bus, exactly-once pipeline, change data capture, stream-table join, saga orchestration, tiered storage, multi-tenant topic, event sourcing replay가 포함된다. 흥미로운 점은 패턴 분류 자체보다, 벤치마크 조건 공개가 일관되지 않아 결과를 실무에 그대로 옮기기 어렵다는 문제 제기다.

Kafka는 단순 메시지 큐가 아니라 durable log를 중심으로 동작한다. 공식 문서도 Kafka를 topic, partition, consumer group, offset, producer/consumer API, Kafka Connect, Kafka Streams 같은 개념으로 설명한다. 따라서 성능을 판단할 때는 "Kafka가 빠른가"보다 "어떤 log 사용 패턴에서 어떤 보장 수준을 택했는가"를 봐야 한다. 예를 들어 exactly-once pipeline은 중복 방지와 정합성 측면에서 강하지만, transaction 경계와 commit 비용이 생긴다. CDC는 데이터베이스 변경을 이벤트로 연결하기 좋지만, schema evolution, snapshot 이후 증분 처리, downstream idempotency를 함께 설계해야 한다.

아래는 CDC 이벤트를 Kafka로 흘려보낸 뒤 downstream에서 멱등 처리와 집계를 나누어 보는 단순한 구조다. 핵심은 Kafka topic 하나가 아니라, 각 단계가 어떤 정합성 책임을 갖는지 분리해서 보는 것이다.

<figure class="post-diagram">
  <svg viewBox="0 0 1040 360" role="img" aria-labelledby="kafka-flow-title kafka-flow-desc" xmlns="http://www.w3.org/2000/svg">
    <title id="kafka-flow-title">Kafka CDC benchmark responsibility flow</title>
    <desc id="kafka-flow-desc">Operational database changes flow through a CDC connector and Kafka topic into an idempotent consumer, state store, analytics table, and benchmark metrics.</desc>
    <defs>
      <marker id="kafka-arrow" markerWidth="12" markerHeight="12" refX="10" refY="6" orient="auto">
        <path d="M2,2 L10,6 L2,10 Z" fill="#3f6473" />
      </marker>
      <style>
        .kafka-box { fill: #f8f5ea; stroke: #3f6473; stroke-width: 2; rx: 14; }
        .kafka-store { fill: #e7f0ee; stroke: #3f6473; stroke-width: 2; rx: 14; }
        .kafka-metric { fill: #f2ead9; stroke: #8a6f3d; stroke-width: 2; rx: 14; }
        .kafka-label { font: 700 17px -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; fill: #263238; }
        .kafka-small { font: 13px -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; fill: #5a6670; }
        .kafka-line { stroke: #3f6473; stroke-width: 3; fill: none; marker-end: url(#kafka-arrow); }
        .kafka-dash { stroke: #8a6f3d; stroke-width: 3; fill: none; stroke-dasharray: 8 8; marker-end: url(#kafka-arrow); }
      </style>
    </defs>

    <rect x="30" y="80" width="140" height="82" class="kafka-store" />
    <text x="100" y="114" text-anchor="middle" class="kafka-label">Operational DB</text>
    <text x="100" y="138" text-anchor="middle" class="kafka-small">source of truth</text>

    <rect x="220" y="80" width="140" height="82" class="kafka-box" />
    <text x="290" y="114" text-anchor="middle" class="kafka-label">CDC Connector</text>
    <text x="290" y="138" text-anchor="middle" class="kafka-small">snapshot + changes</text>

    <rect x="410" y="80" width="140" height="82" class="kafka-store" />
    <text x="480" y="114" text-anchor="middle" class="kafka-label">Kafka Topic</text>
    <text x="480" y="138" text-anchor="middle" class="kafka-small">durable log</text>

    <rect x="600" y="80" width="160" height="82" class="kafka-box" />
    <text x="680" y="114" text-anchor="middle" class="kafka-label">Idempotent</text>
    <text x="680" y="138" text-anchor="middle" class="kafka-small">consumer boundary</text>

    <rect x="830" y="26" width="160" height="74" class="kafka-store" />
    <text x="910" y="58" text-anchor="middle" class="kafka-label">State Store</text>
    <text x="910" y="80" text-anchor="middle" class="kafka-small">dedupe / offsets</text>

    <rect x="830" y="138" width="160" height="74" class="kafka-store" />
    <text x="910" y="170" text-anchor="middle" class="kafka-label">Analytics Table</text>
    <text x="910" y="192" text-anchor="middle" class="kafka-small">upsert result</text>

    <rect x="410" y="260" width="220" height="70" class="kafka-metric" />
    <text x="520" y="290" text-anchor="middle" class="kafka-label">Benchmark Metrics</text>
    <text x="520" y="312" text-anchor="middle" class="kafka-small">lag / latency / recovery</text>

    <path d="M170 121 H220" class="kafka-line" />
    <path d="M360 121 H410" class="kafka-line" />
    <path d="M550 121 H600" class="kafka-line" />
    <path d="M760 110 C790 88 805 72 830 64" class="kafka-line" />
    <path d="M760 132 C790 148 805 164 830 174" class="kafka-line" />

    <path d="M480 162 V260" class="kafka-dash" />
    <path d="M910 100 C910 244 650 236 575 260" class="kafka-dash" />
    <path d="M910 212 C910 292 700 292 630 295" class="kafka-dash" />
  </svg>
  <figcaption>Kafka 패턴은 topic 하나가 아니라 각 단계의 정합성 책임과 운영 지표를 함께 봐야 한다.</figcaption>
</figure>

코드 예시도 실제 운영 코드를 복사하기보다 개념을 작게 드러내는 편이 안전하다. 예를 들어 CDC 이벤트를 처리할 때는 event id 또는 source offset을 기준으로 이미 처리한 이벤트인지 먼저 확인하고, 처리 결과와 offset commit의 경계를 명확히 둔다.

```kotlin
// 개념 예시: CDC 이벤트를 멱등 처리할 때 확인할 최소 흐름
fun handle(event: CdcEvent) {
    if (processedStore.exists(event.id)) {
        return
    }

    val normalized = normalize(event)
    analyticsSink.upsert(normalized)
    processedStore.mark(event.id)
}
```

벤치마크도 마찬가지다. 초당 처리량 하나만 보면 설계 판단이 왜곡될 수 있다. p95/p99 latency, rebalance 시간, 장애 후 복구 시간, consumer lag, compaction 이후 read amplification, tiered storage 사용 시 원격 읽기 지연 같은 지표가 함께 필요하다. 특히 multi-tenant topic이나 event sourcing replay처럼 운영 기간이 길수록 데이터가 누적되는 패턴은 초기 부하 테스트보다 장기 보관, 재처리, backfill, quota 정책이 더 중요해질 수 있다.

Kafka 패턴을 선택할 때는 세 가지 질문을 먼저 던지는 편이 안전하다. 첫째, 이 패턴이 해결하려는 핵심 문제가 순서 보장인지, 재처리인지, 중복 제거인지, 저장 비용인지 명확히 해야 한다. 둘째, 벤치마크를 볼 때 producer/consumer 설정, partition 수, 메시지 크기, replication factor, storage 계층, 장애 주입 여부를 확인해야 한다. 셋째, 처리량 지표와 함께 운영 지표를 본다. 좋은 Kafka 설계는 최대 TPS를 찍는 구성이 아니라, 장애와 재처리 상황에서도 예측 가능한 지연과 정합성 경계를 유지하는 구성에 가깝다.

결국 Kafka 설계 패턴은 레시피가 아니라 선택지다. 패턴 이름은 대화를 시작하게 해주지만, 실제 설계 판단은 벤치마크 조건과 운영 지표를 함께 볼 때 가능하다. 이벤트 스트리밍 시스템을 설계할 때는 "이 패턴을 썼다"보다 "이 패턴을 어떤 부하와 실패 조건에서 검증했는가"가 더 중요한 질문이다.

## 참고 링크

- [Analysis of Design Patterns and Benchmark Practices in Apache Kafka Event-Streaming Systems](https://arxiv.org/abs/2512.16146)
- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
