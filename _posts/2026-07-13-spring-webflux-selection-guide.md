---
title: "Spring WebFlux는 언제 MVC보다 좋은 선택일까"
date: 2026-07-13 09:00:00 +0900
categories: [technical-knowledge, backend-knowledge]
tags: [spring-webflux, reactor, kotlin, reactive-streams, backend]
---

Spring WebFlux는 "MVC보다 빠른 프레임워크"라기보다, 요청 처리 모델을 동기 블로킹에서 비동기 논블로킹으로 바꾸는 선택지에 가깝다. Spring Framework 문서 기준으로 WebFlux는 Spring 5.0에서 추가된 reactive-stack 웹 프레임워크이며, 완전한 논블로킹 처리와 Reactive Streams backpressure를 지원한다. Spring Boot 문서도 WebFlux가 Servlet API에 의존하지 않고 Reactor를 통해 Reactive Streams를 구현한다고 설명한다.

따라서 WebFlux 선택의 핵심 질문은 "트래픽이 많은가"가 아니라 "요청 처리 중 대기 시간이 많은가"에 있다. 외부 API 호출, 메시지 브로커 연동, 스트리밍 응답, SSE, WebSocket, 긴 폴링, 비동기 I/O가 많은 서비스라면 적은 스레드로 많은 동시 연결을 유지하는 WebFlux 모델이 의미가 있다. 반대로 대부분의 작업이 짧은 DB 트랜잭션과 CPU 연산이고, 사용하는 라이브러리도 JDBC처럼 블로킹 중심이라면 MVC가 더 단순하고 예측 가능하다.

<figure class="post-diagram">
  <svg viewBox="0 0 980 300" role="img" aria-labelledby="webflux-flow-title webflux-flow-desc" xmlns="http://www.w3.org/2000/svg">
    <title id="webflux-flow-title">Spring WebFlux request flow</title>
    <desc id="webflux-flow-desc">Client request enters a reactive server, passes through a WebFlux handler and reactive service, waits for external I/O, and returns without blocking the event-loop thread.</desc>
    <defs>
      <marker id="arrow" markerWidth="12" markerHeight="12" refX="10" refY="6" orient="auto">
        <path d="M2,2 L10,6 L2,10 Z" fill="#4f6f52" />
      </marker>
      <style>
        .box { fill: #f8f5ea; stroke: #4f6f52; stroke-width: 2; rx: 14; }
        .accent { fill: #e8f0dd; stroke: #4f6f52; stroke-width: 2; rx: 14; }
        .label { font: 700 18px -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; fill: #2f3329; }
        .small { font: 14px -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; fill: #5e6455; }
        .line { stroke: #4f6f52; stroke-width: 3; fill: none; marker-end: url(#arrow); }
        .return { stroke: #8a6f3d; stroke-width: 3; fill: none; stroke-dasharray: 8 8; marker-end: url(#arrow); }
      </style>
    </defs>

    <rect x="30" y="72" width="140" height="86" class="box" />
    <text x="100" y="108" text-anchor="middle" class="label">Client</text>
    <text x="100" y="134" text-anchor="middle" class="small">HTTP request</text>

    <rect x="220" y="72" width="150" height="86" class="accent" />
    <text x="295" y="108" text-anchor="middle" class="label">Reactive Server</text>
    <text x="295" y="134" text-anchor="middle" class="small">event loop</text>

    <rect x="420" y="72" width="145" height="86" class="box" />
    <text x="492" y="108" text-anchor="middle" class="label">Handler</text>
    <text x="492" y="134" text-anchor="middle" class="small">Mono / Flux</text>

    <rect x="615" y="72" width="145" height="86" class="box" />
    <text x="688" y="108" text-anchor="middle" class="label">Service</text>
    <text x="688" y="134" text-anchor="middle" class="small">reactive chain</text>

    <rect x="810" y="72" width="140" height="86" class="accent" />
    <text x="880" y="108" text-anchor="middle" class="label">External I/O</text>
    <text x="880" y="134" text-anchor="middle" class="small">API / broker / DB</text>

    <path d="M170 115 H220" class="line" />
    <path d="M370 115 H420" class="line" />
    <path d="M565 115 H615" class="line" />
    <path d="M760 115 H810" class="line" />

    <path d="M880 158 C880 238 100 238 100 158" class="return" />
    <text x="490" y="260" text-anchor="middle" class="small">response resumes when I/O completes; event-loop threads are not held while waiting</text>
  </svg>
  <figcaption>WebFlux는 요청을 기다리는 동안 스레드를 점유하지 않는 흐름을 만들 때 효과적이다.</figcaption>
</figure>

WebFlux에서 중요한 것은 체인 중간을 블로킹 호출로 막지 않는 것이다. 컨트롤러 반환 타입이 `Mono`나 `Flux`라고 해서 전체 서비스가 자동으로 논블로킹이 되는 것은 아니다. 내부에서 블로킹 JDBC, 파일 I/O, 오래 걸리는 동기 SDK를 그대로 호출하면 이벤트 루프 스레드를 붙잡아 장점을 잃는다. 이 경우에는 블로킹 작업을 별도 스케줄러로 격리하거나, 애초에 MVC와 명시적인 스레드 풀 모델을 선택하는 편이 더 나을 수 있다.

```kotlin
// 개념 예시: 외부 API 호출과 저장을 논블로킹 체인으로 연결하는 흐름
fun createOrder(command: OrderCommand): Mono<OrderResult> {
    return validate(command)
        .flatMap { validCommand -> pricingClient.quote(validCommand.items) }
        .flatMap { quote -> orderRepository.save(command.toOrder(quote)) }
        .map { saved -> OrderResult(saved.id, saved.status) }
        .timeout(Duration.ofSeconds(2))
}
```

운영 관점에서는 timeout, retry, backpressure, 관측 가능성을 함께 봐야 한다. WebFlux 서비스는 하나의 요청이 여러 비동기 경계를 지나가기 때문에 로그 MDC, trace context, 에러 전파 방식이 MVC보다 복잡해질 수 있다. 또한 팀이 Reactor 연산자와 scheduler 모델에 익숙하지 않으면 단순한 로직도 디버깅 비용이 커진다.

정리하면 WebFlux는 "모든 백엔드의 기본값"이 아니라, 논블로킹 I/O와 스트리밍 응답이 서비스의 핵심일 때 효과적인 선택이다. Spring MVC는 여전히 대부분의 CRUD API에서 좋은 기본값이고, WebClient만 MVC 애플리케이션에 함께 쓰는 조합도 가능하다. 좋은 판단 기준은 프레임워크 선호가 아니라 호출 그래프다. 요청 하나가 어떤 외부 시스템을 기다리는지, 그 대기 시간이 얼마나 많은지, 체인의 어느 지점이 블로킹인지 먼저 그려보면 선택이 훨씬 선명해진다.

## 참고 링크

- [Spring Framework Reference: Spring WebFlux](https://docs.spring.io/spring-framework/reference/web/webflux.html)
- [Spring Boot Reference: Reactive Web Applications](https://docs.spring.io/spring-boot/reference/web/reactive.html)
- [Project Reactor Reference Guide](https://projectreactor.io/docs/core/release/reference/)
