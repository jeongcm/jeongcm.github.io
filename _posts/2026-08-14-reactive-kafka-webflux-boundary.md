---
title: "Reactive Programming과 Kafka가 잘 맞는 이유: Kotlin WebFlux와 Reactor Kafka 기준"
date: 2026-08-14 11:45:00 +0900
categories: [technical-knowledge, backend-knowledge]
tags: [spring-webflux, reactor, reactor-kafka, kafka, kotlin]
---

Kotlin WebFlux 기반 서비스에서 Kafka를 연동할 때 Reactor Kafka가 자주 언급되는 이유는 단순히 "비동기 라이브러리끼리 조합하기 쉽다"가 아니다. Reactive programming은 데이터가 언제 도착할지 모르는 비동기 흐름을 `Publisher`, `Subscriber`, demand, signal로 모델링한다. Kafka도 producer가 record를 비동기로 전송하고, consumer가 partition에서 record를 계속 가져와 처리 위치를 offset으로 남기는 스트리밍 시스템이다. 둘 다 핵심 관심사가 "요청 하나를 즉시 끝내는 것"보다 "계속 흐르는 이벤트를 어느 속도로 받고, 처리하고, 실패 지점을 어디에 둘 것인가"에 가깝다.

Spring WebFlux 문서는 WebFlux가 Reactive Streams back pressure를 지원하는 non-blocking 웹 스택이라고 설명한다. Reactor Kafka도 Kafka sender와 receiver를 Reactor API로 노출한다. `KafkaSender`는 thread-safe라 공유할 수 있지만 하나의 Kafka producer에 연결되고, `KafkaReceiver`는 내부 Kafka consumer 특성상 thread-safe가 아니므로 consumer 인스턴스 경계를 조심해야 한다. 즉 Reactive와 Kafka의 관계는 "Kafka가 자동으로 reactive해진다"가 아니라, Kafka client의 비동기 작업을 Reactor 타입으로 감싸 HTTP, 외부 API, DB, 메시지 브로커 사이의 비동기 경계를 한 모델로 맞추는 데 가깝다.

Reactive programming이 Kafka와 잘 맞는 첫 번째 이유는 흐름 제어를 코드 구조에 드러낼 수 있기 때문이다. HTTP 요청을 받아 Kafka로 발행하는 producer 경로에서는 `send` 결과가 도착하기 전까지 성공 응답을 보류하거나, 발행 요청 자체는 빠르게 받고 결과는 별도 이벤트로 분리할 수 있다. consumer 경로에서는 `concatMap`으로 partition 처리 순서를 보수적으로 지키거나, `flatMap(concurrency = n)`으로 외부 API 호출을 제한된 병렬성 안에서 처리할 수 있다. 이 선택이 코드 연산자로 보이기 때문에 "어디서 병렬화하고 어디서 순서를 지킬 것인가"를 리뷰하기 쉽다.

두 번째 이유는 backpressure를 이야기할 수 있는 공통 언어가 생긴다는 점이다. Reactive Streams의 backpressure는 downstream이 upstream에 demand를 전달하는 프로토콜이다. Kafka consumer의 poll/fetch, `max.poll.records`, partition assignment, offset commit은 Reactive Streams backpressure와 같은 것은 아니지만, 둘 다 처리 속도와 유입 속도 사이의 경계를 다룬다. Reactor Kafka를 쓰면 Kafka에서 가져온 record를 무제한으로 외부 API 호출에 밀어 넣는 대신, `limitRate`, `concatMap`, `flatMap` concurrency, receiver option을 함께 조합해 애플리케이션 내부 압력을 제한할 수 있다.

세 번째 이유는 실패와 완료의 위치를 명확히 표현할 수 있기 때문이다. Reactor에서 error는 terminal signal이고, `retryWhen`은 실패한 흐름을 이어 붙이는 것이 아니라 다시 subscribe하는 동작이다. Kafka consumer에서는 처리 실패 후 offset을 commit하지 않으면 재처리 가능성이 생기고, 처리 후 commit 전에 프로세스가 죽으면 중복 처리가 생긴다. Reactive pipeline 안에서 "처리 완료 -> acknowledge" 순서를 코드로 묶으면, 메시지 유실과 중복의 경계를 눈에 보이게 만들 수 있다.

<figure class="post-diagram">
  <svg viewBox="0 0 880 350" role="img" aria-labelledby="rk-title rk-desc" xmlns="http://www.w3.org/2000/svg">
    <title id="rk-title">WebFlux and Reactor Kafka boundary</title>
    <desc id="rk-desc">A WebFlux request can publish to Kafka through KafkaSender, while KafkaReceiver consumes records in a separate stream with explicit offset acknowledgement.</desc>
    <defs>
      <marker id="arrow-rk" markerWidth="10" markerHeight="10" refX="9" refY="5" orient="auto">
        <path d="M1,1 L9,5 L1,9 Z" fill="#475569" />
      </marker>
      <style>
        .box { rx: 8; stroke-width: 2; }
        .web { fill: #eef6ff; stroke: #2563eb; }
        .reactor { fill: #ecfdf5; stroke: #059669; }
        .kafka { fill: #fff7ed; stroke: #ea580c; }
        .warn { fill: #fef2f2; stroke: #dc2626; }
        .label { font: 700 15px -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; fill: #0f172a; }
        .small { font: 13px -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; fill: #475569; }
        .line { stroke: #475569; stroke-width: 3; fill: none; marker-end: url(#arrow-rk); }
      </style>
    </defs>
    <rect x="36" y="45" width="150" height="82" class="box web" />
    <text x="111" y="80" text-anchor="middle" class="label">WebFlux API</text>
    <text x="111" y="104" text-anchor="middle" class="small">Mono response</text>
    <path d="M186 86 H278" class="line" />
    <rect x="278" y="45" width="190" height="82" class="box reactor" />
    <text x="373" y="80" text-anchor="middle" class="label">KafkaSender</text>
    <text x="373" y="104" text-anchor="middle" class="small">send result as Flux</text>
    <path d="M468 86 H570" class="line" />
    <rect x="570" y="45" width="235" height="82" class="box kafka" />
    <text x="687" y="80" text-anchor="middle" class="label">Kafka topic</text>
    <text x="687" y="104" text-anchor="middle" class="small">acks, ordering, partitions</text>
    <path d="M687 127 V205 H570" class="line" />
    <rect x="570" y="205" width="235" height="92" class="box reactor" />
    <text x="687" y="242" text-anchor="middle" class="label">KafkaReceiver</text>
    <text x="687" y="266" text-anchor="middle" class="small">process then acknowledge</text>
    <path d="M570 251 H468" class="line" />
    <rect x="278" y="205" width="190" height="92" class="box warn" />
    <text x="373" y="242" text-anchor="middle" class="label">Offset boundary</text>
    <text x="373" y="266" text-anchor="middle" class="small">commit after side effect</text>
  </svg>
  <figcaption>Reactor Kafka는 WebFlux와 Kafka를 같은 reactive 타입으로 연결해 흐름 제어와 실패 경계를 코드로 드러내지만, 메시지 처리 보장은 offset과 producer 설정으로 따로 설계해야 한다.</figcaption>
</figure>

좋은 점은 첫째, HTTP 요청에서 메시지 발행 결과를 `Mono`로 자연스럽게 반환할 수 있다는 것이다. controller가 Kafka send 결과를 기다려 `202 Accepted`를 반환하거나, 실패 시 명확한 오류로 매핑할 수 있다. 둘째, consumer 처리에서도 `flatMap`, `concatMap`, `retryWhen`, `timeout` 같은 Reactor 연산자를 사용해 병렬성, 순서, 재시도 정책을 드러낼 수 있다. 셋째, WebFlux, WebClient, Reactor Kafka가 같은 Reactor 생태계에 있으므로 context propagation과 Micrometer Observation 같은 관측성 연동을 한 흐름으로 맞추기 쉽다. 넷째, thread-per-request 방식처럼 대기 중인 외부 호출마다 스레드를 붙잡는 구조를 피하고, Kafka broker 응답과 downstream 응답을 non-blocking signal로 조합할 수 있다.

단점도 분명하다. Reactive pipeline은 Kafka의 delivery semantics를 자동으로 보장하지 않는다. producer 쪽에서는 `acks=all`, idempotence, key 설계, partition 선택을 따로 봐야 한다. consumer 쪽에서는 offset acknowledge 시점이 핵심이다. 처리 전에 acknowledge하면 장애 시 메시지를 잃을 수 있고, 처리 후 acknowledge하면 장애 시 중복 처리가 발생할 수 있다. 따라서 consumer 로직은 멱등 처리나 중복 제거 키를 전제로 두는 편이 안전하다. 또한 reactive 체인은 익숙하지 않으면 실패가 숨어 보일 수 있다. `subscribe()` 위치가 분산되거나, hot/cold publisher 차이를 이해하지 못하거나, consumer lifecycle을 애플리케이션 시작/종료와 제대로 묶지 않으면 운영 중 멈춘 스트림을 알아차리기 어렵다.

다음은 Kotlin WebFlux에서 요청을 Kafka로 발행하고, 별도 consumer가 처리 후 offset을 acknowledge하는 개념 예시다. 실제 서비스에서는 serializer, schema, 보안 설정, DLQ, metric 태그를 별도로 구성해야 한다. 예시의 목적은 API, producer, consumer를 모두 Reactor 타입으로 맞추되, offset acknowledge는 side effect 뒤에 둔다는 경계를 보여주는 것이다.

```kotlin
// 개념 예시: WebFlux controller에서 Reactor Kafka sender를 사용하는 최소 흐름
data class OrderCommand(val orderId: String, val amount: Long)

@RestController
class OrderController(private val sender: KafkaSender<String, OrderCommand>) {
    @PostMapping("/orders")
    fun create(@RequestBody command: Mono<OrderCommand>): Mono<ResponseEntity<Void>> =
        command.flatMap { cmd ->
            val record = ProducerRecord("order-commands", cmd.orderId, cmd)
            sender.send(Mono.just(SenderRecord.create(record, cmd.orderId)))
                .single()
                .thenReturn(ResponseEntity.accepted().build())
        }
}

class OrderConsumer(private val receiver: KafkaReceiver<String, OrderCommand>) {
    fun start(): Disposable =
        receiver.receive()
            .concatMap { record ->
                handle(record.value())
                    .timeout(Duration.ofSeconds(3))
                    .then(Mono.fromRunnable { record.receiverOffset().acknowledge() })
            }
            .retryWhen(Retry.backoff(Long.MAX_VALUE, Duration.ofSeconds(1)))
            .subscribe()

    private fun handle(command: OrderCommand): Mono<Void> =
        Mono.empty()
}
```

이 예시에서 `concatMap`은 한 consumer stream 안에서 처리 순서를 보수적으로 지키는 선택이다. 처리량을 높이려고 `flatMap`을 쓰면 동시성은 늘지만 partition별 순서와 offset acknowledge 순서를 더 세밀하게 검토해야 한다. Kafka consumer 설정의 `max.poll.records`, `max.poll.interval.ms`, 수동 acknowledge 정책도 같이 맞춰야 한다. 오래 걸리는 외부 호출을 consumer pipeline 안에 넣으면 poll 간격과 rebalance에도 영향을 줄 수 있다.

조금 더 깊게 보면 WebFlux와 Kafka의 결합에서 가장 중요한 설계 질문은 "HTTP 요청과 Kafka record의 완료 의미가 같은가"이다. API가 Kafka에 record를 성공적으로 append하면 바로 `202 Accepted`를 반환하는 모델이라면, HTTP 성공은 비즈니스 처리 완료가 아니라 접수 완료를 뜻한다. 이때 클라이언트에는 비동기 처리 상태 조회나 후속 이벤트 계약이 필요할 수 있다. 반대로 HTTP 응답 전에 downstream 처리까지 끝내야 한다면 Kafka는 비동기 완충재가 아니라 동기 요청 경로의 일부가 되고, timeout과 장애 전파 모델이 완전히 달라진다.

Kafka의 partition도 reactive 설계와 연결된다. 같은 key를 가진 record는 같은 partition으로 가기 때문에 순서가 중요하면 key 선택이 먼저다. Reactor에서 `flatMap`을 크게 열어 처리량을 올려도, 같은 entity의 이벤트 순서를 깨면 최종 상태가 틀어질 수 있다. 이 경우에는 partition 단위 순서를 유지하거나, key별 serialization, idempotent update, version check 같은 도메인 방어가 필요하다. Reactive 연산자는 처리 형태를 표현할 뿐, key 설계와 상태 전이를 대신 정해 주지 않는다.

Reactor Kafka를 쓰기 좋은 경우는 WebFlux 서비스가 이미 Reactor 타입을 중심으로 설계돼 있고, Kafka 발행/소비가 HTTP나 WebClient 호출과 같은 비동기 흐름 안에서 연결되는 경우다. 예를 들어 API 요청을 검증한 뒤 이벤트를 발행하거나, Kafka 이벤트를 받아 non-blocking downstream API를 호출하는 서비스라면 코드 모델이 잘 맞는다. 반대로 단순 배치 소비, 복잡한 stateful stream processing, join/window/aggregation이 핵심이라면 Reactor Kafka보다 Kafka Streams, Flink, Spark Structured Streaming 같은 처리 엔진이 더 적합할 수 있다.

운영 기준은 세 가지로 정리할 수 있다. 첫째, WebFlux의 backpressure와 Kafka의 fetch/commit은 같은 개념이 아니므로 둘을 혼동하지 않는다. 둘째, send 성공은 broker append와 producer 설정 기준의 성공이지 비즈니스 처리가 끝났다는 뜻이 아니다. 셋째, consumer offset은 side effect 완료 후 acknowledge하되, 그 결과 중복 가능성을 멱등성 키와 재처리 정책으로 흡수한다. Reactive와 Kafka를 함께 쓰는 핵심은 "전부 비동기로 만든다"가 아니라 요청, 메시지, offset, 부작용의 경계를 코드에 명확히 드러내는 것이다.

정리하면 Kafka는 지속적인 이벤트 흐름, partition 기반 순서, offset 기반 재처리라는 모델을 가진다. Reactive programming은 비동기 데이터 흐름, demand, signal, composition이라는 모델을 가진다. 두 모델은 모두 "흐름을 어떻게 제어할 것인가"를 중심에 둔다는 점에서 잘 맞는다. 다만 잘 맞는다는 말은 보장까지 자동으로 맞춰진다는 뜻이 아니다. WebFlux와 Reactor Kafka를 선택할 때는 non-blocking end-to-end 흐름, 명시적인 병렬성, 관측성 통합을 얻는 대신, offset commit, 중복 처리, lifecycle, partition 순서를 더 엄격하게 설계해야 한다.

## 참고 링크

- [Reactor Kafka Reference Guide](https://docs.spring.io/projectreactor/reactor-kafka/docs/current-SNAPSHOT/reference/html/)
- [Reactor Kafka API Overview](https://docs.spring.io/projectreactor/reactor-kafka/docs/current/api/overview-summary.html)
- [Spring Framework Reference - Spring WebFlux](https://docs.spring.io/spring/reference/web/webflux.html)
- [Spring Framework Reference - Reactive Core](https://docs.spring.io/spring-framework/reference/web/webflux/reactive-spring.html)
- [Apache Kafka Producer Configs](https://kafka.apache.org/40/configuration/producer-configs/)
- [Apache Kafka Consumer Configs](https://kafka.apache.org/26/configuration/consumer-configs/)
