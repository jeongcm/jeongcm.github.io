---
title: "Apache Flink 집계가 강한 이유: 상태, 이벤트 시간, 체크포인트"
date: 2026-07-08 16:21:00 +0900
categories: [technical-knowledge, data-engineering]
tags: [apache-flink, aggregation, event-time, stateful-streaming, checkpoint]
---

데이터 엔지니어링에서 집계는 단순해 보이지만 운영에서는 까다롭다. "최근 5분 주문 수", "사용자별 일별 클릭 수", "상품별 실시간 매출", "비정상 이벤트 비율" 같은 지표는 모두 여러 이벤트를 묶어서 하나의 값으로 만드는 작업이다. 배치에서는 원본 테이블을 읽고 `GROUP BY`를 수행하면 되지만, 스트리밍에서는 데이터가 끝나지 않는다. Flink의 집계가 강한 이유는 이 끝나지 않는 입력을 다루기 위한 세 가지 축을 런타임에 깊게 갖고 있기 때문이다. **상태**, **이벤트 시간**, **체크포인트**다.

내가 한 주문 스트리밍 작업에서 Flink를 쓴 이유도 결국 이 세 가지와 맞닿아 있다. 이 파이프라인은 주문 이벤트를 단순히 받아서 저장하는 작업이 아니었다. Kafka 재처리로 같은 주문 아이템이 다시 들어올 수 있고, 삭제 이벤트가 별도 토픽으로 늦게 올 수 있고, API/NFT 주문은 아이템별 금액을 다시 합산해 같은 주문의 모든 row에 반영해야 했다. 이런 문제는 "한 번 읽고 한 번 쓰는 ETL"이 아니라, key별 상태를 오래 들고 판단해야 하는 스트리밍 애플리케이션에 가깝다.

첫 번째 축은 상태다. 스트리밍 집계는 과거 값을 기억해야 한다. `user_id`별 카운트를 세려면 각 user의 현재 카운트가 필요하고, 10분 window의 합계를 내려면 window별 중간 결과가 필요하다. Flink는 이런 값을 operator state 또는 keyed state로 관리한다. 특히 `keyBy` 이후의 keyed state는 key별로 분산되어 TaskManager들에 나뉘어 저장된다. 한 머신이 모든 user의 상태를 들고 있는 것이 아니라, key space가 병렬 subtask에 나뉘어 올라간다.

이 구조는 집계에서 큰 장점이 된다. key별 상태를 로컬에서 갱신할 수 있기 때문에 매 이벤트마다 외부 DB를 조회하지 않아도 된다. 상태가 메모리나 RocksDB 같은 state backend에 있고, Flink 연산자가 그 상태를 직접 읽고 갱신한다. 그래서 실시간 대시보드나 알림처럼 낮은 지연시간이 중요한 집계에 잘 맞는다. 물론 상태가 커질수록 checkpoint 시간, RocksDB I/O, compaction, 메모리 설정을 함께 봐야 한다.

그 작업의 금액 보정 로직은 이 장점을 잘 보여준다. 특정 키 단위로 주문 아이템들을 MapState에 보관하고, 결제 금액이 비어 있거나 취소/삭제 이벤트가 들어오면 활성 아이템들의 결제 금액을 다시 합산한다. 그리고 그 합계를 같은 주문의 모든 아이템에 다시 emit한다. 외부 DB를 매번 조회해서 맞추는 방식이 아니라, Flink operator 안의 keyed state가 현재 주문의 부분 집계를 들고 있는 구조다.

두 번째 축은 이벤트 시간이다. 실시간 시스템에서 "지금 도착한 시간"과 "이 이벤트가 실제로 발생한 시간"은 다르다. 모바일 앱 로그, CDC, Kafka 재처리, 네트워크 지연을 생각하면 이벤트는 늦게 도착할 수 있다. Flink는 event time과 watermark를 이용해 "어느 시점까지의 데이터가 대체로 도착했는지"를 추정하고, window를 닫거나 늦은 데이터를 처리한다. 이 덕분에 단순히 처리 시간 기준으로 집계하는 것보다 비즈니스 시간에 가까운 결과를 만들 수 있다.

예를 들어 10:00부터 10:05까지의 매출을 집계한다고 하자. 10:04:59에 발생한 주문 이벤트가 네트워크 문제로 10:06에 도착할 수 있다. 처리 시간 기준이면 이 이벤트는 다음 구간에 들어가거나 누락될 수 있다. 이벤트 시간과 watermark를 쓰면 시스템은 어느 정도의 지연을 허용하고, 해당 이벤트를 원래 window에 반영할 수 있다. 실시간 집계의 품질은 여기서 많이 갈린다. 빠르기만 한 집계보다, 늦게 온 데이터를 어떻게 해석할지 정한 집계가 운영에서 더 믿을 만하다.

세 번째 축은 체크포인트다. 집계는 상태를 갖기 때문에 장애 복구가 중요하다. Flink는 상태와 소스 위치를 주기적으로 스냅샷으로 남긴다. 장애가 발생하면 마지막으로 성공한 체크포인트에서 상태를 복원하고, 소스도 그 지점부터 다시 읽는다. Kafka 같은 소스와 정확히 맞물리면 "어디까지 읽고 어떤 상태를 만들었는가"를 함께 복구할 수 있다. 이 모델은 exactly-once 처리 보장의 기반이 된다.

이 파이프라인에서는 RocksDB state backend와 S3 checkpoint/savepoint를 사용하고, checkpoint consistency mode를 EXACTLY_ONCE로 둔다. 중복 제거 상태, 삭제 이벤트의 최신 timestamp 상태, 주문 아이템 MapState가 모두 복구 가능한 대상이 된다. 그래서 장애 후에는 단순히 Kafka offset만 맞추는 것이 아니라, "어떤 주문 아이템을 이미 보았고, 어떤 delete가 이겼고, 어떤 금액 합산 상태였는지"까지 함께 복원하는 것이 중요하다.

집계에서 이 세 가지가 합쳐지면 다음과 같은 이점이 생긴다.

1. key별 상태를 분산해서 큰 규모의 집계를 지속적으로 유지할 수 있다.
2. 이벤트 시간과 watermark로 늦게 도착한 데이터까지 비즈니스 시간 기준으로 다룰 수 있다.
3. 체크포인트로 장애 후에도 집계 상태를 재구성할 수 있다.
4. window, trigger, allowed lateness 같은 설정으로 "빠른 중간 결과"와 "정확한 최종 결과" 사이의 균형을 잡을 수 있다.
5. SQL과 DataStream API 양쪽에서 상태 기반 집계를 표현할 수 있다.

그렇다고 Flink 집계가 항상 정답은 아니다. 상태가 너무 커지면 운영 난도가 올라간다. key skew가 있으면 특정 subtask만 뜨거워진다. 외부 sink가 느리면 backpressure가 전체 잡으로 전파된다. 이벤트 시간 기준을 잘못 잡으면 지표가 사용자의 기대와 다르게 보인다. Flink는 강력하지만, 그 강력함은 설정과 운영 책임을 함께 가져온다.

내가 Flink 집계를 설계한다면 먼저 질문을 쪼갠다. 이 집계는 처리 시간 기준이어도 되는가, 이벤트 시간 기준이어야 하는가. 늦은 데이터는 몇 분까지 받아야 하는가. window가 닫힌 뒤 수정 결과를 내보내도 downstream이 감당할 수 있는가. 상태 크기는 얼마나 커질 수 있고, checkpoint SLA는 얼마인가. key skew가 생길 가능성은 없는가. 이 질문에 답할 수 있으면 Flink 집계는 꽤 탄탄한 무기가 된다.

Flink의 장점은 집계를 "주기적으로 다시 계산하는 작업"이 아니라 "계속 살아 있는 상태ful computation"으로 다룬다는 데 있다. 그래서 이벤트가 들어오는 즉시 상태가 갱신되고, 장애가 나도 그 상태를 복원하며, 늦은 이벤트도 시간 의미에 맞게 처리할 수 있다. 실시간 집계에서 이 차이는 생각보다 크다.

내가 이 작업에서 배운 것은 Flink 집계가 꼭 `window().aggregate()`처럼 보이지 않아도 된다는 점이다. 중복 제거도 집계의 한 형태이고, delete와 upsert 중 최신 이벤트를 고르는 것도 상태 기반 집계이며, 주문 아이템들의 결제 금액을 다시 계산하는 것도 작은 materialized view다. Flink의 강점은 이런 비즈니스 상태를 이벤트가 들어올 때마다 계속 갱신하고, 그 결과를 Iceberg나 Kafka 같은 downstream에 안정적으로 흘려보내는 데 있다.

## 참고 링크

- [Apache Flink Stateful Stream Processing](https://nightlies.apache.org/flink/flink-docs-release-2.1/docs/concepts/stateful-stream-processing/)
- [Apache Flink Windows](https://nightlies.apache.org/flink/flink-docs-release-2.1/docs/dev/datastream/operators/windows/)
- [Apache Flink Fault Tolerance](https://nightlies.apache.org/flink/flink-docs-release-2.1/docs/learn-flink/fault_tolerance/)
