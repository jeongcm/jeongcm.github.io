---
title: "Kotlin WebFlux의 성능은 어디서 결정될까: Netty, Event Loop, Coroutine 이해하기"
date: 2026-09-08 15:14:40 +0900
categories: [technical-knowledge, backend-knowledge]
tags: [kotlin, spring-webflux, netty, reactor, coroutines, performance]
---

Kotlin과 Spring WebFlux로 API를 만들면 흔히 "적은 스레드로 많은 요청을 처리한다"고 설명한다. 이 문장은 방향은 맞지만, 성능과 안정성을 판단하기에는 부족하다. WebFlux가 요청을 어떤 스레드에서 실행하는지, Netty가 어디까지 담당하는지, `suspend`가 무엇을 멈추고 무엇을 멈추지 않는지 알아야 병목을 찾을 수 있다.

이 글은 Kotlin WebFlux와 Go HTTP 서버의 성능·안정성을 비교하기 전에 필요한 사전 지식을 정리한다. 특정 프레임워크가 언제나 더 빠르다고 결론 내리기보다, WebFlux의 실행 모델과 실패 지점을 먼저 분해하는 것이 목적이다.

## WebFlux, Reactor, Netty는 각각 무엇을 담당할까

세 기술은 같은 것이 아니다.

- **Spring WebFlux**는 HTTP 요청 라우팅, controller와 filter, body codec, 오류 처리, `WebClient` 같은 웹 프로그래밍 모델을 제공한다.
- **Project Reactor**는 `Mono`, `Flux`, operator, scheduler, Reactive Streams backpressure를 제공하는 비동기 실행 기반이다. WebFlux는 내부적으로 Reactor를 사용한다.
- **Netty**는 TCP 연결, socket read/write, event loop, buffer, protocol pipeline을 다루는 비동기 네트워크 프레임워크다. Spring Boot의 WebFlux starter는 기본 서버로 Reactor Netty를 사용하지만, WebFlux 자체는 Tomcat이나 Jetty 같은 다른 서버에서도 실행할 수 있다.

Netty를 조금 더 구체적으로 보면, 연결은 `Channel`로 표현되고 I/O 이벤트는 `EventLoopGroup`의 스레드가 처리한다. 연결에서 읽은 데이터는 `ChannelPipeline`의 handler들을 지나며 HTTP 메시지로 해석된다. Reactor Netty는 이 Netty API 위에 Reactive Streams와 HTTP server/client API를 제공하고, Spring WebFlux는 다시 그 위에 `HttpHandler`, `WebHandler`, controller 같은 애플리케이션 계층을 올린다.

따라서 "WebFlux가 빠르다"는 표현은 정확하지 않다. Netty의 non-blocking I/O, Reactor의 비동기 조합, 애플리케이션이 사용하는 client와 driver, payload 처리 방식이 모두 연결되어 있을 때 작은 수의 스레드로 높은 동시성을 유지할 수 있다.

<figure class="post-diagram">
  <svg viewBox="0 0 940 440" role="img" aria-labelledby="webflux-netty-title webflux-netty-desc" xmlns="http://www.w3.org/2000/svg">
    <title id="webflux-netty-title">Kotlin WebFlux request execution flow</title>
    <desc id="webflux-netty-desc">A request moves from a network socket through a Netty event loop, Reactor Netty, Spring WebFlux, and a Kotlin suspending handler. Non-blocking downstream I/O suspends and later resumes the handler without occupying the event-loop thread.</desc>
    <defs>
      <marker id="arrow-webflux-netty" markerWidth="10" markerHeight="10" refX="9" refY="5" orient="auto">
        <path d="M0,0 L10,5 L0,10 Z" fill="#475569" />
      </marker>
      <style>
        .stage { stroke-width: 2; rx: 6; }
        .network { fill: #e0f2fe; stroke: #0284c7; }
        .runtime { fill: #ecfccb; stroke: #65a30d; }
        .spring { fill: #dcfce7; stroke: #16a34a; }
        .kotlin { fill: #f3e8ff; stroke: #9333ea; }
        .external { fill: #fff7ed; stroke: #ea580c; }
        .title { font: 700 17px -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; fill: #0f172a; }
        .text { font: 13px -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; fill: #334155; }
        .line { stroke: #475569; stroke-width: 2.5; fill: none; marker-end: url(#arrow-webflux-netty); }
        .wait { stroke: #9333ea; stroke-width: 2.5; stroke-dasharray: 7 6; fill: none; marker-end: url(#arrow-webflux-netty); }
      </style>
    </defs>

    <rect x="28" y="48" width="150" height="82" class="stage network" />
    <text x="103" y="80" text-anchor="middle" class="title">Socket / Channel</text>
    <text x="103" y="105" text-anchor="middle" class="text">readiness event</text>

    <rect x="218" y="48" width="150" height="82" class="stage runtime" />
    <text x="293" y="80" text-anchor="middle" class="title">Netty EventLoop</text>
    <text x="293" y="105" text-anchor="middle" class="text">small worker set</text>

    <rect x="408" y="48" width="150" height="82" class="stage runtime" />
    <text x="483" y="80" text-anchor="middle" class="title">Reactor Netty</text>
    <text x="483" y="105" text-anchor="middle" class="text">HTTP + Publisher</text>

    <rect x="598" y="48" width="150" height="82" class="stage spring" />
    <text x="673" y="80" text-anchor="middle" class="title">Spring WebFlux</text>
    <text x="673" y="105" text-anchor="middle" class="text">route, codec, filter</text>

    <rect x="788" y="48" width="124" height="82" class="stage kotlin" />
    <text x="850" y="80" text-anchor="middle" class="title">suspend handler</text>
    <text x="850" y="105" text-anchor="middle" class="text">application flow</text>

    <path d="M178 89 H218" class="line" />
    <path d="M368 89 H408" class="line" />
    <path d="M558 89 H598" class="line" />
    <path d="M748 89 H788" class="line" />

    <rect x="350" y="258" width="240" height="92" class="stage external" />
    <text x="470" y="292" text-anchor="middle" class="title">Non-blocking downstream I/O</text>
    <text x="470" y="318" text-anchor="middle" class="text">WebClient, R2DBC, reactive driver</text>

    <path d="M850 130 C850 210 690 245 590 285" class="wait" />
    <text x="750" y="222" text-anchor="middle" class="text">suspend and release thread</text>
    <path d="M350 320 C260 350 258 210 293 130" class="wait" />
    <text x="186" y="314" class="text">I/O ready signal</text>

    <rect x="640" y="274" width="272" height="66" class="stage" fill="#fee2e2" stroke="#dc2626" />
    <text x="776" y="302" text-anchor="middle" class="title">Blocking call on EventLoop</text>
    <text x="776" y="325" text-anchor="middle" class="text">other connections wait behind it</text>
    <path d="M850 130 V274" stroke="#dc2626" stroke-width="3" marker-end="url(#arrow-webflux-netty)" />
  </svg>
  <figcaption>non-blocking I/O는 기다리는 동안 event-loop 스레드를 반환한다. 반대로 같은 스레드에서 blocking 호출을 실행하면 다른 연결의 진행까지 늦어진다.</figcaption>
</figure>

## Event Loop는 요청을 어떻게 처리할까

전통적인 thread-per-request 모델은 요청을 처리하는 스레드가 DB나 외부 API의 응답을 기다리는 동안에도 그 스레드를 점유할 수 있다. 이 모델은 이해하기 쉽지만, 동시 연결이 많고 I/O 대기 시간이 길면 더 많은 스레드와 stack memory, context switching이 필요하다.

Event loop 모델은 다르게 동작한다. 소켓이 읽기나 쓰기 가능한 상태가 되면 소수의 worker가 짧은 작업을 수행하고, 다음 준비된 I/O로 이동한다. 외부 응답을 기다리는 동안 해당 요청 전용 스레드를 붙잡아 두지 않는다. Reactor Netty 문서에 따르면 기본 event loop worker 수는 런타임이 사용할 수 있는 processor 수를 기준으로 하며 최소값은 4다. 서버와 client가 같은 JVM에서 기본 Reactor Netty 자원을 사용하면 event loop와 connection 자원을 공유할 수 있다.

이 구조의 장점은 스레드 수가 트래픽과 함께 무한히 늘지 않아, I/O 대기가 많은 부하에서 자원 사용을 비교적 예측하기 쉽다는 점이다. 반면 worker 하나가 오래 막히면 그 worker가 담당하는 여러 연결이 함께 지연된다. 작은 thread pool은 WebFlux의 장점인 동시에 반드시 지켜야 하는 제약이다.

## Kotlin `suspend`는 스레드를 만드는 기능이 아니다

Kotlin의 suspending function은 실행을 중단했다가 결과가 준비되면 이어갈 수 있는 함수다. `suspend`가 붙었다고 새 스레드가 생기거나 내부 코드가 자동으로 non-blocking으로 변하는 것은 아니다.

WebFlux controller에서 `suspend fun`을 사용하면 Spring은 Coroutines와 Reactor 사이를 연결한다. `WebClient.awaitBody()`처럼 취소 가능한 suspending API를 기다릴 때 coroutine은 중단되고 event-loop 스레드는 다른 작업을 처리할 수 있다. 결과가 도착하면 continuation이 다시 실행되어 응답을 완성한다. `Flow`는 여러 값을 비동기로 전달하며, 소비 속도 조절은 suspension으로 표현한다.

다음은 실행 흐름을 설명하기 위한 최소 개념 예시다.

```kotlin
// 개념 예시: WebClient 호출을 기다리는 동안 event-loop thread를 점유하지 않는다.
@RestController
class ProductController(
    private val webClient: WebClient
) {
    @GetMapping("/products/{id}")
    suspend fun product(@PathVariable id: String): ProductView {
        val product = webClient.get()
            .uri("https://catalog.example/products/{id}", id)
            .retrieve()
            .awaitBody<Product>()

        return ProductView.from(product)
    }
}
```

이 코드가 non-blocking인 이유는 `suspend`라는 키워드 자체가 아니라, `WebClient`와 `awaitBody()`가 Reactor subscription 및 coroutine cancellation과 연결된 비동기 API이기 때문이다. 같은 함수 안에서 JDBC, `Thread.sleep`, 동기 HTTP client를 호출하면 해당 스레드는 그대로 막힌다.

## 성능은 언제 좋아지고 언제 나빠질까

WebFlux의 주된 이점은 단일 요청의 실행 시간을 마법처럼 줄이는 것이 아니다. Spring 문서도 reactive와 non-blocking이 일반적으로 애플리케이션을 더 빠르게 만드는 것은 아니며, 핵심 이점은 작은 고정 스레드 수와 적은 메모리로 확장하는 능력이라고 설명한다.

다음 조건에서는 장점이 드러나기 쉽다.

- 요청 하나가 여러 외부 API, message broker, reactive database를 기다리는 I/O 중심 작업이다.
- SSE, streaming response, long-lived connection처럼 연결 유지 시간이 길다.
- 호출 경로의 client와 driver가 처음부터 끝까지 non-blocking이다.
- timeout, connection pool, concurrency limit으로 느린 downstream의 영향을 제한한다.

반대로 짧고 CPU 집약적인 연산은 event loop라고 빨라지지 않는다. JSON 변환, 압축, 암호화, 이미지 처리처럼 CPU를 오래 쓰는 작업을 event loop에서 실행하면 I/O 처리 기회가 줄어든다. JDBC나 동기 SDK를 별도 pool로 옮기는 것도 임시 격리일 뿐이다. blocking 작업의 처리량이 pool 크기보다 많으면 queue가 늘고 tail latency가 악화된다.

불가피한 blocking API는 event loop 밖의 제한된 실행 자원으로 격리해야 한다. 다음은 완성된 운영 코드가 아니라 경계를 보여주는 pseudocode다.

```kotlin
// pseudocode: 실제 서비스에서는 전용 dispatcher의 크기, queue, 종료 수명주기를 관리해야 한다.
class LegacyRepository(
    private val blockingDispatcher: CoroutineDispatcher
) {
    suspend fun find(id: String): LegacyRecord =
        withContext(blockingDispatcher) {
            jdbcTemplate.queryForObject(/* ... */)
        }
}
```

격리했다고 문제가 사라지는 것은 아니다. pool이 포화됐을 때 요청을 얼마나 기다리게 할지, 거부할지, timeout을 어디에 둘지까지 정해야 안정적인 경계가 된다. 장기적으로는 호출 비중과 트랜잭션 요구사항을 보고 MVC/JDBC를 유지할지, R2DBC 같은 reactive driver로 바꿀지를 판단해야 한다.

## WebFlux를 쓰면서 조심해야 할 점

### 1. Blocking 호출은 코드 깊은 곳에도 숨어 있다

`block()`, `Thread.sleep`, JDBC뿐 아니라 파일 접근, DNS, 인증 SDK, 압축, template rendering, 오래 걸리는 logging appender도 event loop를 막을 수 있다. controller 반환 타입이 `Mono`이거나 함수가 `suspend`라는 사실만으로 호출 그래프 전체가 non-blocking임을 보장하지 않는다. dependency별 I/O 모델을 확인하고, 부하 테스트에서 `reactor-http-nio-*` thread의 긴 task와 event-loop pending task를 관찰해야 한다.

### 2. 동시성은 반드시 상한이 있어야 한다

Reactor의 `flatMap`이나 Coroutine의 `async`를 입력 개수만큼 무제한 생성하면 downstream connection pool보다 많은 작업이 쌓일 수 있다. 비동기 작업은 값싼 대기가 가능하지만 공짜 작업은 아니다. 각 요청은 buffer, continuation, timeout, trace context와 connection 대기 상태를 소비한다. fan-out 개수, in-flight request, queue 길이에 제한을 두고 포화 시 빠르게 실패하거나 backpressure를 적용해야 한다.

### 3. Server와 WebClient가 같은 자원을 공유할 수 있다

기본 Reactor Netty 구성에서는 server와 client가 event loop 자원을 공유한다. 정상적인 non-blocking 흐름에서는 효율적이지만, client callback이나 응답 가공이 event loop를 오래 점유하면 inbound 요청 처리에도 영향을 줄 수 있다. 무조건 자원을 분리하기보다 먼저 blocking 구간과 CPU 작업을 제거하고, 실제 contention이 측정될 때 `LoopResources`와 connection pool 분리를 검토하는 편이 낫다.

### 4. Timeout 하나로 모든 대기를 제어할 수 없다

연결 수립, connection pool acquire, response 수신, 전체 요청 처리에는 서로 다른 시간 제한이 필요하다. timeout이 없으면 느린 downstream 때문에 in-flight 요청이 계속 쌓일 수 있고, 너무 짧으면 정상적인 지연도 실패로 바뀐다. retry를 추가할 때는 멱등성, backoff, jitter와 전체 deadline을 함께 설계해야 한다. 취소가 전파돼도 이미 downstream에 전달된 쓰기 작업이 자동으로 rollback되는 것은 아니다.

### 5. 큰 body를 한 번에 모으면 메모리 이점이 사라진다

WebFlux codec은 애플리케이션의 memory 문제를 막기 위해 body buffering 한도를 둔다. `DataBufferLimitException`을 피하려고 `maxInMemorySize`만 크게 올리면 동시 요청 수에 비례해 heap 사용량이 증가한다. 큰 JSON, 업로드, 다운로드는 가능한 한 streaming으로 처리하고, 반드시 모아야 한다면 payload 상한과 동시 처리량을 함께 제한해야 한다. 낮은 수준에서 pooled `DataBuffer`를 직접 다룰 때는 release 책임도 확인해야 한다.

### 6. Coroutine 취소와 업무 트랜잭션은 같은 개념이 아니다

클라이언트 연결 종료나 timeout으로 coroutine이 취소되면 하위 reactive subscription도 취소될 수 있다. 그러나 외부 API가 이미 요청을 받았거나 DB commit이 시작됐다면 업무 효과까지 취소된다고 가정할 수 없다. 결제, 주문, 메시지 발행 같은 쓰기 작업은 idempotency key, 상태 조회, 보상 처리로 불확실한 결과를 다뤄야 한다.

### 7. ThreadLocal 기반 문맥은 비동기 경계를 그대로 넘지 않는다

하나의 요청이 항상 같은 스레드에서 실행되지 않으므로 MDC, 보안 주체, tracing 정보를 일반 `ThreadLocal`에만 두면 문맥이 유실될 수 있다. Reactor `Context`, coroutine context, Micrometer context propagation 사이의 경계를 명시하고, controller에서 WebClient까지 같은 trace와 request ID가 유지되는지 통합 테스트로 확인해야 한다.

### 8. 평균 지연시간만 보면 과부하 신호를 놓친다

Event loop가 잠깐씩 막히거나 connection pool이 포화되면 평균보다 p95·p99가 먼저 나빠진다. 운영에서는 throughput과 평균 응답 시간 외에도 다음을 함께 봐야 한다.

- p50, p95, p99 latency와 timeout·cancel 비율
- event-loop pending task와 blocked thread 징후
- WebClient connection pool의 active, idle, pending acquire
- heap, direct memory, GC pause, process RSS
- in-flight request, queue, retry 횟수와 downstream 오류율
- CPU 사용률, thread 수, 열린 connection 수

## Go와 비교하기 전에 고정해야 할 조건

Go의 `net/http`는 connection과 request 처리를 goroutine으로 표현하고, Kotlin WebFlux는 Reactor Netty event loop와 continuation으로 대기를 표현한다. 겉으로 보이는 동시성 단위는 다르지만 둘 다 운영체제의 non-blocking I/O와 runtime scheduler 위에서 동작한다. 단순한 hello-world 요청 수치만으로는 실제 선택 기준을 만들기 어렵다.

후속 비교에서는 최소한 다음 조건을 같게 해야 한다.

1. 같은 JSON schema와 직렬화 비용
2. 같은 downstream latency와 connection reuse 조건
3. 같은 timeout, 최대 in-flight 요청, connection pool 상한
4. CPU-bound, 빠른 I/O, 느린 I/O, downstream 장애를 분리한 workload
5. warm-up 이후 throughput뿐 아니라 p95·p99, 오류율, RSS, CPU, thread·goroutine 수 측정
6. 정상 부하를 넘겼을 때 queue가 어디에 쌓이고 어떻게 회복하는지 확인

Kotlin WebFlux의 안정성은 event loop 자체보다 애플리케이션이 그 계약을 얼마나 일관되게 지키는지에 달려 있다. Netty는 많은 연결의 준비 이벤트를 효율적으로 전달하고, Reactor와 Coroutines는 기다림을 비동기로 표현한다. 하지만 blocking 호출, 무제한 fan-out, 큰 buffer, 불명확한 timeout이 섞이면 작은 스레드 수라는 장점이 병목 증폭기로 바뀔 수 있다. 성능 비교의 출발점은 언어별 최고 수치가 아니라, 같은 부하에서 어느 런타임이 자원을 어떻게 제한하고 과부하에서 어떻게 실패하는지를 관찰하는 것이다.

## 참고 링크

- [Spring Framework - Spring WebFlux](https://docs.spring.io/spring-framework/reference/web/webflux.html)
- [Spring Framework - WebFlux Overview, Performance and Concurrency Model](https://docs.spring.io/spring-framework/reference/web/webflux/new-framework.html)
- [Spring Framework - Reactive Core](https://docs.spring.io/spring-framework/reference/web/webflux/reactive-spring.html)
- [Spring Framework - Kotlin Coroutines](https://docs.spring.io/spring-framework/reference/languages/kotlin/coroutines.html)
- [Spring Framework - WebClient Configuration](https://docs.spring.io/spring-framework/reference/web/webflux-webclient/client-builder.html)
- [Reactor Netty Reference Guide](https://projectreactor.io/docs/netty/release/reference/index.html)
- [Project Reactor - Schedulers](https://projectreactor.io/docs/core/release/api/reactor/core/scheduler/Schedulers.html)
- [Netty User Guide for 4.x](https://netty.io/wiki/user-guide-for-4.x.html)
- [Kotlin Coroutines Guide](https://kotlinlang.org/docs/coroutines-guide.html)
