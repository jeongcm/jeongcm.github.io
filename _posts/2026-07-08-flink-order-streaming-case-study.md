---
title: "내가 한 주문 스트리밍 작업에서 본 Flink 사용 관점"
date: 2026-07-08 16:21:00 +0900
categories: [technical-knowledge, data-engineering]
tags: [apache-flink, kafka, iceberg, cdc, exactly-once, case-study]
---

Flink를 설명할 때 흔히 "실시간 집계 엔진"이라고 말한다. 맞는 말이지만, 내가 주문 스트리밍 작업에서 Flink를 사용한 관점은 조금 더 구체적이었다. 핵심은 **Kafka로 들어오는 주문 변경 이벤트를 Iceberg에서 믿고 조회할 수 있는 상태로 정규화하는 것**이었다.

주문 이벤트는 생각보다 지저분하다. 같은 주문 아이템이 Kafka 재처리로 다시 들어올 수 있고, 실제 삭제 이벤트는 별도 토픽에서 들어올 수 있다. 어떤 이벤트는 최신 upsert이고, 어떤 이벤트는 과거 delete일 수 있다. API/NFT 주문은 아이템별 `결제금액`은 있는데 주문 전체 금액이 비어 있거나 취소 이벤트 때문에 다시 계산해야 할 수 있다. 이런 문제는 단순히 메시지를 하나씩 sink에 쓰는 방식으로 풀기 어렵다.

첫 번째 관점은 **중복을 sink에 맡기지 않는 것**이다. Iceberg upsert는 유용하지만, 같은 checkpoint 안에서 완전히 같은 레코드가 반복되는 문제까지 깔끔하게 해결해주지는 않는다. 그래서 주문 아이템을keyBy하고, Flink의 ValueState에 마지막 레코드 hash와 updatedAt을 저장한다. 완전히 같은 이벤트면 downstream으로 보내지 않고, 내용이 바뀐 경우에만 다음 단계로 흘린다.

두 번째 관점은 **delete와 upsert의 승자를 명시적으로 정하는 것**이다. 주문 삭제 이벤트는 일반 주문 이벤트와 다른 Kafka source에서 들어온다. 이 둘을 keyBy 로 connect하고, KeyedCoProcessFunction에서 최신 updatedAt과 마지막 op를 상태로 관리한다. 더 과거의 delete는 무시하고, 같은 timestamp에서는 delete가 이기게 둔다. 이 정책이 코드에 들어가 있어야 Iceberg에는 "마지막으로 살아 있는 상태"가 안정적으로 쌓인다.

세 번째 관점은 **업무 규칙을 상태ful 보정으로 처리하는 것**이다. API/NFT 주문에서는 같은 주문 안의 여러 아이템을 보고 `주문 결제금액`을 다시 계산해야 한다. 이때 외부 DB를 매번 조회하는 대신 key 단위로 MapState를 유지한다. upsert, cancel, delete가 들어오면 state의 활성 아이템을 기준으로 금액을 다시 합산하고, 같은 주문의 아이템들을 재발행한다. 이건 window aggregation은 아니지만, 실무적으로는 작은 materialized view에 가깝다.

네 번째 관점은 **최신 상태 테이블과 관측용 로그 테이블을 분리하는 것**이다. Iceberg 테이블은 identifier fields를 가진 upsert 테이블로 만든다. upsert 테이블만 있으면 같은 주문이 같은 상태로 몇 번 수집됐는지 사라진다. 

다섯 번째 관점은 **복구 가능한 상태로 운영하는 것**이다. 공통 Flink 환경은 RocksDB state backend, S3 checkpoint/savepoint, EXACTLY_ONCE checkpoint mode를 사용한다. 즉 중복 제거 상태, delete 승자 상태, 주문 금액 보정 MapState가 checkpoint와 함께 복구된다. 장애가 났을 때 중요한 것은 "잡이 다시 떴다"가 아니라 "잡이 기억하던 주문 상태가 함께 돌아왔다"는 점이다.

이 구조에서 Flink의 장점은 낮은 latency 하나로 설명되지 않는다. Flink는 이벤트를 순서대로 흘려보내는 파이프가 아니라, key별 업무 상태를 들고 있는 실행 환경이다. Kafka는 이벤트 로그를 제공하고, Flink는 그 이벤트를 해석해 현재 상태를 만든다. Iceberg는 그 결과를 장기적으로 조회 가능한 테이블로 보존한다. 이 세 가지가 맞물려야 실시간 주문 파이프라인이 운영 가능한 시스템이 된다.

그래서 이 작업에서 Flink를 쓴 이유를 한 문장으로 정리하면 이렇다.

> 주문 이벤트의 중복, 삭제, 보정, 복구 문제를 애플리케이션 코드와 외부 DB 호출로 흩뜨리지 않고, key별 상태를 가진 스트리밍 런타임 안에서 해결하기 위해서.

이 관점으로 보면 Flink의 TaskManager, JobManager, checkpoint, state backend, Iceberg sink는 각각 따로 떨어진 기능이 아니다. TaskManager는 key별 상태를 가진 subtask를 실행하고, JobManager는 checkpoint와 복구를 조율한다. RocksDB와 S3는 상태를 오래 들고 갈 수 있게 해준다. Iceberg upsert와 append 테이블은 각각 최신 상태와 운영 관측을 담당한다.

Flink를 도입할 때 "빠른가?"만 물으면 답이 좁아진다. 더 좋은 질문은 "우리 이벤트에는 어떤 충돌이 있고, 그 충돌을 어디에서 결정할 것인가?"다. 이 작업에서 내 답은 Flink였다. 주문 이벤트를 그대로 저장하는 것이 아니라, 상태를 가진 스트리밍 잡 안에서 신뢰 가능한 주문 상태로 바꿔야 했기 때문이다.

## 참고 링크

- [Apache Flink Architecture](https://nightlies.apache.org/flink/flink-docs-release-2.1/docs/concepts/flink-architecture/)
- [Apache Flink Stateful Stream Processing](https://nightlies.apache.org/flink/flink-docs-release-2.1/docs/concepts/stateful-stream-processing/)
- [Apache Flink Fault Tolerance](https://nightlies.apache.org/flink/flink-docs-release-2.1/docs/learn-flink/fault_tolerance/)
- [Apache Iceberg Flink Writes](https://iceberg.apache.org/docs/latest/flink-writes/)
