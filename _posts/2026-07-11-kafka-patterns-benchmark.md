---
title: "Kafka 설계 패턴은 왜 벤치마크와 같이 봐야 할까"
date: 2026-07-11 09:35:00 +0900
categories: [technical-knowledge, data-engineering]
tags: [apache-kafka, event-streaming, benchmarking, cdc, exactly-once]
---

Kafka 기반 이벤트 스트리밍 시스템을 설계할 때 자주 등장하는 말이 있다. log compaction, CDC, exactly-once pipeline, stream-table join, event sourcing, tiered storage 같은 패턴들이다. 이 표현들은 익숙하지만, 실제 시스템에 적용할 때는 이름만으로 충분하지 않다. 같은 패턴이라도 topic 수, partition 수, 메시지 크기, ack 설정, consumer 처리 방식, state store 사용 여부에 따라 성능과 장애 양상이 크게 달라지기 때문이다.

최근 arXiv에 공개된 Kafka 설계 패턴과 벤치마크 연구는 이 지점을 잘 짚는다. 이 논문은 2015년부터 2025년까지의 peer-reviewed 연구 42개를 분석해 Kafka 이벤트 스트리밍에서 반복적으로 등장하는 9가지 패턴을 정리한다. 여기에는 log compaction, CQRS bus, exactly-once pipeline, change data capture, stream-table join, saga orchestration, tiered storage, multi-tenant topic, event sourcing replay가 포함된다. 흥미로운 점은 패턴 분류 자체보다, 벤치마크 조건 공개가 일관되지 않아 결과를 실무에 그대로 옮기기 어렵다는 문제 제기다.

Kafka는 단순 메시지 큐가 아니라 durable log를 중심으로 동작한다. 공식 문서도 Kafka를 topic, partition, consumer group, offset, producer/consumer API, Kafka Connect, Kafka Streams 같은 개념으로 설명한다. 따라서 성능을 판단할 때는 "Kafka가 빠른가"보다 "어떤 log 사용 패턴에서 어떤 보장 수준을 택했는가"를 봐야 한다. 예를 들어 exactly-once pipeline은 중복 방지와 정합성 측면에서 강하지만, transaction 경계와 commit 비용이 생긴다. CDC는 데이터베이스 변경을 이벤트로 연결하기 좋지만, schema evolution, snapshot 이후 증분 처리, downstream idempotency를 함께 설계해야 한다.

벤치마크도 마찬가지다. 초당 처리량 하나만 보면 설계 판단이 왜곡될 수 있다. p95/p99 latency, rebalance 시간, 장애 후 복구 시간, consumer lag, compaction 이후 read amplification, tiered storage 사용 시 원격 읽기 지연 같은 지표가 함께 필요하다. 특히 multi-tenant topic이나 event sourcing replay처럼 운영 기간이 길수록 데이터가 누적되는 패턴은 초기 부하 테스트보다 장기 보관, 재처리, backfill, quota 정책이 더 중요해질 수 있다.

Kafka 패턴을 선택할 때는 세 가지 질문을 먼저 던지는 편이 안전하다. 첫째, 이 패턴이 해결하려는 핵심 문제가 순서 보장인지, 재처리인지, 중복 제거인지, 저장 비용인지 명확히 해야 한다. 둘째, 벤치마크를 볼 때 producer/consumer 설정, partition 수, 메시지 크기, replication factor, storage 계층, 장애 주입 여부를 확인해야 한다. 셋째, 처리량 지표와 함께 운영 지표를 본다. 좋은 Kafka 설계는 최대 TPS를 찍는 구성이 아니라, 장애와 재처리 상황에서도 예측 가능한 지연과 정합성 경계를 유지하는 구성에 가깝다.

결국 Kafka 설계 패턴은 레시피가 아니라 선택지다. 패턴 이름은 대화를 시작하게 해주지만, 실제 설계 판단은 벤치마크 조건과 운영 지표를 함께 볼 때 가능하다. 이벤트 스트리밍 시스템을 설계할 때는 "이 패턴을 썼다"보다 "이 패턴을 어떤 부하와 실패 조건에서 검증했는가"가 더 중요한 질문이다.

## 참고 링크

- [Analysis of Design Patterns and Benchmark Practices in Apache Kafka Event-Streaming Systems](https://arxiv.org/abs/2512.16146)
- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
