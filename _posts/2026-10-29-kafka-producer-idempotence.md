---
title: "Kafka Producer의 멱등성과 exactly-once 전달"
date: 2026-10-29 09:00:00 +09:00
categories: [data-engineering]
tags: [kafka, messaging, exactly-once, idempotence, producer]
---

Kafka는 0.11 릴리스부터 생산자(Producer)에 멱등성(Idempotence) 기능을 지원한다. 이 기능은 전달 시맨틱스를 "최소 한 번(At-least-once)"에서 "정확히 한 번(Exactly-once)"으로 끌어올린다. 핵심은 재시도(Retry) 시 중복이 발생하는 문제를 제거한다는 점이다.

기존 최소 한 번 전달에서는 클라이언트가 요청 실패를 감지하면 재전송을 수행했다. 재전송은 서버 중복 기록을 유발할 수 있어, 소비자(Consumer) 쪽이 중복 메시지를 걸러야 했다. 멱등성 생산자는 서버에서 프로듀서 ID를 기준으로 시퀀스 번호를 검증하여, 같은 메시지가 한 번만 기록되도록 보장한다.

멱등성을 켜려면 `enable.idempotence`를 `true`로 설정한다. Kafka 3.0부터 이 값은 기본 `true`로 변경되었다. 함께 따라오는 기본값이 두 가지 있다. 첫째, `retries`는 `Integer.MAX_VALUE`로 설정된다. 둘째, `acks`는 `all`로 설정된다. 세 조합이 하나의 보장이 된다. 깊은 복제를 전제로 한 전송, 무한 재시도, 리더 복제 확인 이후의 기록. 이 조건이 갖춰지면 서버 중복 없이 안전하게 재전송할 수 있다.

멱등성에는 전제 조건이 있다. `max.in.flight.requests.per.connection`은 5 이하여야 한다. 이 값을 넘으면 메시지 순서가 깨질 수 있어, 멱등성은 비활성된다. 즉, 순서 보장이 깨지는 동시성 환경에서는 멱등성이 자동으로 꺼지는 것이다. 또한 멱등성은 단일 생산자 세션 안에서만 보장된다. 브로커 재시작이나 프로듀서 재시작으로 세션이 바뀌면 같은 키라도 이전 세션의 시퀀스 정보가 남아 있지 않다. 세션 경계를 넘는 원자적 쓰기는 트랜잭션 생산자가 담당한다.

트랜잭션 생산자는 `transactional.id`를 설정하여 사용한다. 이 값을 지정하면 멱등성이 자동으로 활성화되며, 여러 파티션과 토픽에 걸친 원자적 커밋을 지원한다. 배치 단위 커밋으로 소비자 오프셋과 메시지를 함께 트랜잭션에 넣는 방식도 가능하다. 다만 트랜잭션은 리더 대기, 트랜잭션 커밋 대기, 조정(Coordinator) 통신이 추가되어 지연과 처리량 비용이 따른다.

실무에서는 대부분 `enable.idempotence=true`만으로 충분한다. 트랜잭션까지 필요한 경우는 이벤트 발행 후 소비자 오프셋 저장까지 원자적으로 처리해야 하는 파이프라인, 혹은 여러 토픽에 걸친 상태 갱신과 같이 "실패 시 전체 롤백"이 필요한 케이스다.

요약하면, 멱등성 생산자는 송신 측의 재시도로 인한 중복을 제거하는 장치다. 설정 변경은 간단하다. 기본값이 켜져 있는 상태에서 값만 유지하면, 재시도 없는 것처럼 동작하는 전송을 얻을 수 있다.

---

## 멱등 전송과 최소 한 번 전달 비교

| 특성 | 최소 한 번 | 멱등성(생산자) |
| --- | --- | --- |
| 재시도 시 중복 | 발생 가능 | 발생 없음 |
| 보장 수준 | 최소 한 번 | 정확히 한 번 |
| 필요 설정 | — | `enable.idempotence=true`, `acks=all` |
| 세션 경계 보장 | 해당 없음 | 단일 세션 내 |
| 크로스 세션 원자성 | 해당 없음 | 트랜잭션 생산자 필요 |

---

[참고]
- Apache Kafka 공식 문서 – Producer Configs: https://kafka.apache.org/43/configuration/producer-configs/
- Apache Kafka Javadoc – KafkaProducer: https://kafka.apache.org/43/javadoc/org/apache/kafka/clients/producer/KafkaProducer.html
