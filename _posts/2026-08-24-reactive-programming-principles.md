---
title: "Reactive는 어떤 원리로 동작할까: signal, demand, scheduler 이해하기"
date: 2026-08-24 11:03:21 +0900
categories: [technical-knowledge, backend-knowledge]
tags: [reactive-programming, reactor, reactive-streams, spring-webflux, backpressure]
---

Reactive programming은 "비동기로 코드를 짜는 방식" 정도로 설명되곤 하지만, 핵심은 더 좁고 구체적이다. Reactive Streams 기준에서 중요한 문제는 빠른 생산자가 느린 소비자를 압도하지 않도록, 비동기 경계에서 데이터 흐름과 수요를 명시적으로 조절하는 것이다. Reactor나 Spring WebFlux가 다루는 `Mono`, `Flux`, `Publisher`, `Subscriber`, `Subscription`은 이 계약을 코드로 표현하는 도구다.

가장 먼저 봐야 할 단위는 signal이다. Reactive Streams의 기본 흐름은 `onSubscribe`로 시작하고, 0개 이상의 `onNext`가 온 뒤, `onError` 또는 `onComplete` 중 하나로 끝난다. 이 terminal signal 이후에는 더 이상 signal이 오면 안 된다. 그래서 reactive pipeline에서 error는 일반 예외처럼 중간에서 잡고 이어가는 값이 아니라, 기본적으로 현재 sequence를 종료시키는 사건이다. Reactor의 `onErrorResume`, `retry`, `timeout` 같은 연산자는 이 종료 의미를 바탕으로 새 흐름을 만들거나 대체 흐름으로 전환한다.

<svg viewBox="0 0 820 300" role="img" aria-label="Reactive Streams signal and demand flow" xmlns="http://www.w3.org/2000/svg">
  <rect x="40" y="50" width="160" height="72" rx="8" fill="#e8f3ff" stroke="#3178c6"/>
  <text x="120" y="82" text-anchor="middle" font-size="16" font-family="sans-serif">Publisher</text>
  <text x="120" y="104" text-anchor="middle" font-size="12" font-family="sans-serif">source / upstream</text>
  <rect x="330" y="50" width="160" height="72" rx="8" fill="#f2f7e8" stroke="#6a994e"/>
  <text x="410" y="82" text-anchor="middle" font-size="16" font-family="sans-serif">Subscription</text>
  <text x="410" y="104" text-anchor="middle" font-size="12" font-family="sans-serif">demand contract</text>
  <rect x="620" y="50" width="160" height="72" rx="8" fill="#fff2d8" stroke="#d08c00"/>
  <text x="700" y="82" text-anchor="middle" font-size="16" font-family="sans-serif">Subscriber</text>
  <text x="700" y="104" text-anchor="middle" font-size="12" font-family="sans-serif">consumer / downstream</text>
  <path d="M200 82 H330" stroke="#555" stroke-width="2" marker-end="url(#arrow)"/>
  <text x="265" y="70" text-anchor="middle" font-size="12" font-family="sans-serif">onSubscribe</text>
  <path d="M620 105 H490" stroke="#555" stroke-width="2" marker-end="url(#arrow)"/>
  <text x="555" y="128" text-anchor="middle" font-size="12" font-family="sans-serif">request(n)</text>
  <path d="M200 158 C300 210 520 210 620 158" fill="none" stroke="#555" stroke-width="2" marker-end="url(#arrow)"/>
  <text x="410" y="225" text-anchor="middle" font-size="12" font-family="sans-serif">onNext up to requested demand</text>
  <path d="M200 252 H620" stroke="#555" stroke-width="2" marker-end="url(#arrow)"/>
  <text x="410" y="274" text-anchor="middle" font-size="12" font-family="sans-serif">onComplete or onError terminates the sequence</text>
  <defs>
    <marker id="arrow" markerWidth="10" markerHeight="10" refX="8" refY="3" orient="auto">
      <path d="M0,0 L0,6 L9,3 z" fill="#555"/>
    </marker>
  </defs>
</svg>

두 번째 원리는 demand다. `Subscriber`는 `Subscription.request(n)`으로 받을 수 있는 개수를 알리고, `Publisher`는 요청된 개수보다 많은 `onNext`를 보내면 안 된다. 이것이 backpressure의 가장 단순한 형태다. blocking queue처럼 생산자를 스레드 대기로 막는 것이 아니라, "앞으로 몇 개까지 받을 수 있다"는 수요 신호를 비동기 프로토콜에 포함한다. 소비자가 `Long.MAX_VALUE`를 요청하면 사실상 무제한 요청이 되므로, backpressure가 있다고 해서 항상 자동으로 안전해지는 것은 아니다.

세 번째 원리는 assembly와 subscription의 분리다. Reactor에서 `Flux`나 `Mono`를 만든다고 바로 실행되는 것은 아니다. 대부분의 pipeline은 선언만 된 상태이고, `subscribe`가 일어나야 upstream으로 구독이 전파된다. 이때 operator마다 중간 subscriber가 만들어지고, `map`, `flatMap`, `filter`, `timeout` 같은 단계가 signal을 변환한다. 이 구조 때문에 reactive 코드는 "위에서 아래로 실행된다"보다 "구독 시점에 signal graph가 동작한다"로 이해해야 한다.

개념 예시는 아래처럼 읽을 수 있다.

```kotlin
// 개념 예시: demand와 signal을 보여주기 위한 단순 흐름
Flux.range(1, 100)
    .map { it * 10 }
    .limitRate(10)
    .doOnNext { value -> println("next=$value") }
    .doOnError { error -> println("error=${error.message}") }
    .doOnComplete { println("complete") }
    .subscribe()
```

네 번째 원리는 scheduler다. Reactive는 자동으로 모든 연산을 다른 스레드에서 실행한다는 뜻이 아니다. Reactor 문서 기준으로 대부분의 operator는 별도 scheduler를 지정하지 않으면 이전 단계가 실행된 스레드에서 계속 동작한다. `subscribeOn`은 upstream 구독과 source 실행 위치에 영향을 주고, `publishOn`은 그 지점 이후 downstream 실행 위치를 바꾼다. blocking 호출을 event loop 위에서 실행하면 reactive 구조를 쓰더라도 병목이 생긴다. 필요한 경우 `boundedElastic` 같은 별도 scheduler로 blocking 경계를 격리해야 한다.

실무에서 Reactive를 선택할 때는 처리량만 보지 말고 경계를 봐야 한다. HTTP 요청, WebClient 호출, 메시지 발행, DB 드라이버, 파일 I/O가 모두 non-blocking 흐름에 맞을 때 장점이 분명하다. 반대로 중간에 `block()`이 섞이거나, transaction/context/offset 같은 운영 경계를 이해하지 못한 채 operator만 늘리면 디버깅이 더 어려워진다. Reactive의 원리는 마법이 아니라 signal, demand, subscription, scheduler라는 계약을 끝까지 지키는 데 있다.

## 참고 링크

- [Reactive Streams JVM Specification](https://github.com/reactive-streams/reactive-streams-jvm)
- [Project Reactor Documentation](https://projectreactor.io/docs)
- [Project Reactor Reference Guide](https://projectreactor.io/docs/core/release/reference/)
- [Spring Framework WebFlux](https://docs.spring.io/spring-framework/reference/web/webflux.html)
- [Spring Framework Reactive Core](https://docs.spring.io/spring-framework/reference/web/webflux/reactive-spring.html)
