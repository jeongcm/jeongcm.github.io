---
title: "Spring WebFlux에서 Kotlin Coroutines를 쓰는 기준"
date: 2026-07-28 10:58:16 +0900
categories: [technical-knowledge, backend-knowledge]
tags: [spring-webflux, kotlin, coroutines, reactor, backend]
---

Spring WebFlux는 Reactor의 `Mono`, `Flux`를 중심으로 비동기 흐름을 표현한다. Kotlin 기반 서비스에서는 같은 WebFlux 위에서 `suspend fun`, `Flow`, `coRouter` 같은 Coroutines API를 함께 사용할 수 있다. 중요한 판단은 "Reactor를 버리고 Coroutines로 바꿀 것인가"가 아니라, 어떤 경계에서 Coroutines의 명령형 스타일을 쓰고 어떤 경계에서는 Reactor의 연산자와 `Context`를 유지할 것인가다.

## 공식 문서 기준 핵심 개념

Spring Framework의 Kotlin Coroutines 문서는 `kotlinx-coroutines-core`와 `kotlinx-coroutines-reactor`가 classpath에 있을 때 Coroutines 지원이 활성화된다고 설명한다. WebFlux annotated `@Controller`에서는 `suspend fun`과 `Flow` 반환값을 사용할 수 있고, 함수형 엔드포인트에서는 Coroutines 전용 `coRouter { }` DSL과 suspending handler를 사용할 수 있다.

Reactive 타입과 Coroutines 타입의 대응도 명확하다. `Mono<Void>`는 값을 반환하지 않는 `suspend fun`으로, `Mono<T>`는 `suspend fun(): T` 또는 nullable 반환으로, `Flux<T>`는 `Flow<T>`로 옮길 수 있다. 단, 이 대응은 표현 방식의 변화이지 실행 모델이 blocking으로 바뀐다는 뜻은 아니다. WebFlux의 기본 전제는 여전히 event loop를 막지 않는 non-blocking 처리다.

<figure class="post-diagram">
  <svg viewBox="0 0 860 330" role="img" aria-labelledby="coroutines-title coroutines-desc" xmlns="http://www.w3.org/2000/svg">
    <title id="coroutines-title">WebFlux Reactor and Kotlin Coroutines boundary</title>
    <desc id="coroutines-desc">A WebFlux request can use coroutine controllers for application flow while adapting to Reactor at infrastructure boundaries such as WebClient, filters, and context propagation.</desc>
    <defs>
      <marker id="arrow-coroutine" markerWidth="11" markerHeight="11" refX="9" refY="5.5" orient="auto">
        <path d="M1,1 L10,5.5 L1,10 Z" fill="#315f72" />
      </marker>
      <style>
        .panel { fill: #f8fafc; stroke: #cbd5e1; stroke-width: 2; rx: 8; }
        .reactor { fill: #e0f2fe; stroke: #0284c7; stroke-width: 2; rx: 8; }
        .kotlin { fill: #ede9fe; stroke: #7c3aed; stroke-width: 2; rx: 8; }
        .ops { fill: #dcfce7; stroke: #16a34a; stroke-width: 2; rx: 8; }
        .label { font: 700 16px -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; fill: #0f172a; }
        .small { font: 13px -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; fill: #475569; }
        .line { stroke: #315f72; stroke-width: 3; fill: none; marker-end: url(#arrow-coroutine); }
      </style>
    </defs>
    <rect x="24" y="28" width="812" height="274" class="panel" />
    <text x="54" y="67" class="label">WebFlux + Kotlin Coroutines boundary</text>
    <rect x="60" y="114" width="168" height="86" class="reactor" />
    <text x="144" y="148" text-anchor="middle" class="label">WebFlux runtime</text>
    <text x="144" y="173" text-anchor="middle" class="small">event loop, Reactor</text>
    <path d="M228 157 H320" class="line" />
    <rect x="320" y="94" width="200" height="126" class="kotlin" />
    <text x="420" y="130" text-anchor="middle" class="label">Application flow</text>
    <text x="420" y="157" text-anchor="middle" class="small">suspend fun</text>
    <text x="420" y="181" text-anchor="middle" class="small">Flow, coroutineScope</text>
    <path d="M520 157 H612" class="line" />
    <rect x="612" y="114" width="168" height="86" class="reactor" />
    <text x="696" y="148" text-anchor="middle" class="label">Reactive boundary</text>
    <text x="696" y="173" text-anchor="middle" class="small">WebClient, filters</text>
    <rect x="268" y="244" width="304" height="38" class="ops" />
    <text x="420" y="269" text-anchor="middle" class="small">Context propagation and cancellation must cross the boundary deliberately.</text>
  </svg>
  <figcaption>Coroutines는 WebFlux 애플리케이션 코드의 표현 방식을 단순화하지만, Reactor 기반 런타임과 통합 경계는 그대로 남는다.</figcaption>
</figure>

## 언제 적합하고 언제 과한가

Coroutines는 요청 처리 흐름이 순차적 비즈니스 로직에 가까울 때 적합하다. 여러 외부 호출을 조합하거나, 조건 분기와 예외 처리가 많은 컨트롤러에서 `flatMap` 체인이 깊어지는 경우 `suspend fun`은 읽기 쉬운 흐름을 만든다. `coroutineScope`와 `async`를 사용하면 병렬 호출도 명시적으로 표현할 수 있다.

반대로 데이터 스트림의 backpressure, 재시도, window, merge, group 연산처럼 Reactor 연산자가 문제를 더 정확히 표현하는 구간에서는 `Flux`를 무리하게 `Flow`로 감싸지 않는 편이 낫다. WebClient filter, WebFilter, library integration처럼 Reactor `Context`와 구독 시점의 의미가 중요한 경계도 Reactor 모델을 이해한 상태에서 다뤄야 한다.

## 개념 예시

아래 코드는 실행 가능한 완성 코드가 아니라, WebClient의 Reactor 기반 API를 Coroutines 스타일의 요청 흐름 안에서 사용하는 개념 예시다.

```kotlin
@RestController
class OrderViewController(
    private val client: WebClient
) {
    @GetMapping("/orders/{id}/view")
    suspend fun view(@PathVariable id: String): OrderView = coroutineScope {
        val order = async {
            client.get()
                .uri("/internal/orders/{id}", id)
                .retrieve()
                .awaitBody<Order>()
        }

        val payments = async {
            client.get()
                .uri("/internal/orders/{id}/payments", id)
                .retrieve()
                .bodyToFlow<Payment>()
                .toList()
        }

        OrderView(order.await(), payments.await())
    }
}
```

이런 형태에서는 컨트롤러의 읽기 흐름은 imperative하게 보이지만, `awaitBody`와 `bodyToFlow`는 `kotlinx-coroutines-reactor`가 제공하는 adapter를 통해 reactive publisher와 연결된다. 호출을 기다리는 동안 event loop를 blocking하지 않는다는 점이 핵심이다.

## 운영 시 주의할 점

첫째, blocking API를 `suspend fun` 안에 넣는다고 non-blocking이 되지는 않는다. JDBC, 파일 I/O, 동기 HTTP client처럼 스레드를 점유하는 작업은 WebFlux event loop 밖에서 실행되도록 별도 dispatcher나 bounded thread pool 기준을 정해야 한다.

둘째, 취소 전파를 설계해야 한다. `awaitSingle` 계열 API는 coroutine이 취소되면 subscription을 취소하도록 동작한다. 클라이언트 연결 종료, timeout, 상위 coroutine 취소가 하위 HTTP 호출과 스트림 수집에 어떤 의미인지 테스트로 확인하는 편이 안전하다.

셋째, 관측성 컨텍스트를 놓치기 쉽다. Spring 문서는 tracing의 현재 observation이 blocking code에서는 `ThreadLocal`, reactive pipeline에서는 Reactor `Context`, suspended function에서는 coroutine execution context에 맞게 전달되어야 한다고 설명한다. Micrometer Context Propagation의 `PropagationContextElement`나 `Hooks.enableAutomaticContextPropagation()`을 사용할 수 있지만, 자동 전파는 모든 경계 문제를 없애는 마법이 아니다. 로깅, tracing, WebClient filter가 같은 trace를 보는지 통합 테스트로 확인해야 한다.

정리하면 Kotlin Coroutines는 WebFlux를 더 동기식 코드처럼 쓰게 해주는 문법 선택이 아니라, non-blocking 런타임 위에서 애플리케이션 흐름을 표현하는 다른 모델이다. 컨트롤러와 서비스 흐름에는 `suspend`와 `Flow`를 사용하되, Reactor 연산자와 `Context`가 강한 구간은 경계로 남겨두는 기준이 유지보수에 유리하다.

## 참고 링크

- [Spring Framework - Kotlin Coroutines](https://docs.spring.io/spring-framework/reference/languages/kotlin/coroutines.html)
- [Spring Framework - Spring WebFlux Overview](https://docs.spring.io/spring-framework/reference/web/webflux/new-framework.html)
- [Spring Framework - Reactive Libraries](https://docs.spring.io/spring-framework/reference/web/webflux-reactive-libraries.html)
- [Spring Framework - Kotlin Web](https://docs.spring.io/spring-framework/reference/languages/kotlin/web.html)
- [kotlinx.coroutines.reactor API](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-reactor/kotlinx.coroutines.reactor/)
- [kotlinx.coroutines awaitSingle API](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-reactive/kotlinx.coroutines.reactive/await-single.html)
