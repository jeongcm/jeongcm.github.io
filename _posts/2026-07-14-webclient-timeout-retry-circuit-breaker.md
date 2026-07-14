---
title: "WebClient timeout과 retry는 어디에 걸어야 할까"
date: 2026-07-14 09:03:27 +0900
categories: [technical-knowledge, backend-knowledge]
tags: [spring-webflux, webclient, reactor, timeout, retry, circuit-breaker]
---

`WebClient` 장애 처리는 `timeout` 하나를 붙이는 문제가 아니다. 외부 API 호출은 연결 수립, 커넥션 풀 대기, 요청 전송, 응답 대기, 본문 처리, 재시도, 차단까지 서로 다른 실패 지점을 가진다. Spring Framework 문서 기준으로 `WebClient`는 Reactor 기반의 논블로킹 HTTP 클라이언트이며, Reactor Netty, JDK HttpClient, Jetty Reactive HttpClient 같은 HTTP 클라이언트 구현 위에서 동작한다. 따라서 운영 설정은 `WebClient` API뿐 아니라 실제 커넥터의 timeout 모델까지 함께 봐야 한다.

핵심은 timeout을 가능한 한 실패 지점에 가깝게 두는 것이다. Reactor Netty 문서는 `Mono`/`Flux`의 `timeout` 연산자가 전체 작업에 적용될 수 있지만, 연결이나 응답 대기처럼 목적이 분명한 경우에는 Reactor Netty의 더 구체적인 timeout 설정이 더 적합하다고 설명한다. 예를 들어 연결 수립이 늦는 문제는 `CONNECT_TIMEOUT_MILLIS`, 응답 첫 바이트가 늦는 문제는 `responseTimeout`으로 분리해 보는 편이 원인 파악과 알림 설계에 유리하다.

<figure class="post-diagram">
  <svg viewBox="0 0 940 260" role="img" aria-labelledby="webclient-resilience-title webclient-resilience-desc" xmlns="http://www.w3.org/2000/svg">
    <title id="webclient-resilience-title">WebClient resilience boundary</title>
    <desc id="webclient-resilience-desc">A WebClient request passes through connection timeout, response timeout, retry filter, circuit breaker, and fallback decision points.</desc>
    <defs>
      <marker id="arrow-webclient" markerWidth="12" markerHeight="12" refX="10" refY="6" orient="auto">
        <path d="M2,2 L10,6 L2,10 Z" fill="#3d6b66" />
      </marker>
      <style>
        .box { fill: #f8f5ea; stroke: #3d6b66; stroke-width: 2; rx: 10; }
        .guard { fill: #e7f1ef; stroke: #3d6b66; stroke-width: 2; rx: 10; }
        .fail { fill: #f5e5df; stroke: #945540; stroke-width: 2; rx: 10; }
        .label { font: 700 17px -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; fill: #26332f; }
        .small { font: 13px -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; fill: #59625d; }
        .line { stroke: #3d6b66; stroke-width: 3; fill: none; marker-end: url(#arrow-webclient); }
      </style>
    </defs>
    <rect x="24" y="78" width="130" height="78" class="box" />
    <text x="89" y="111" text-anchor="middle" class="label">Caller</text>
    <text x="89" y="135" text-anchor="middle" class="small">business request</text>

    <rect x="190" y="78" width="130" height="78" class="guard" />
    <text x="255" y="111" text-anchor="middle" class="label">Connect</text>
    <text x="255" y="135" text-anchor="middle" class="small">connect timeout</text>

    <rect x="356" y="78" width="130" height="78" class="guard" />
    <text x="421" y="111" text-anchor="middle" class="label">Response</text>
    <text x="421" y="135" text-anchor="middle" class="small">response timeout</text>

    <rect x="522" y="78" width="130" height="78" class="guard" />
    <text x="587" y="111" text-anchor="middle" class="label">Retry</text>
    <text x="587" y="135" text-anchor="middle" class="small">filtered backoff</text>

    <rect x="688" y="78" width="130" height="78" class="fail" />
    <text x="753" y="111" text-anchor="middle" class="label">Circuit</text>
    <text x="753" y="135" text-anchor="middle" class="small">open / half-open</text>

    <rect x="356" y="188" width="296" height="46" class="box" />
    <text x="504" y="217" text-anchor="middle" class="small">fallback, degraded response, or explicit failure</text>

    <path d="M154 117 H190" class="line" />
    <path d="M320 117 H356" class="line" />
    <path d="M486 117 H522" class="line" />
    <path d="M652 117 H688" class="line" />
    <path d="M753 156 C753 206 658 211 652 211" class="line" />
  </svg>
  <figcaption>외부 호출 안정성은 하나의 timeout이 아니라 단계별 실패 경계의 조합으로 설계한다.</figcaption>
</figure>

Retry는 모든 예외에 기계적으로 적용하면 안 된다. Reactor의 `retryWhen`과 `Retry` 계열은 재시도 횟수, 예외 필터, backoff, 재시도 소진 시 예외 변환을 조정할 수 있다. 이 기능은 일시적인 네트워크 오류나 503처럼 재시도 가치가 있는 실패에만 걸어야 한다. 400 계열 검증 오류, 인증 실패, 멱등성이 없는 주문 생성 요청처럼 재시도하면 부작용이 커지는 호출은 제외해야 한다.

```kotlin
// 개념 예시: WebClient 호출의 timeout, retry, fallback 배치
fun fetchPrice(productId: String): Mono<Price> {
    return priceClient.get()
        .uri("/prices/{id}", productId)
        .retrieve()
        .bodyToMono<Price>()
        .retryWhen(
            Retry.backoff(2, Duration.ofMillis(200))
                .filter { error -> error is IOException || error is TimeoutException }
        )
        .timeout(Duration.ofSeconds(2))
        .onErrorResume { error -> Mono.error(PriceUnavailableException(error)) }
}
```

Circuit breaker는 retry와 역할이 다르다. Resilience4j 문서 기준으로 circuit breaker는 `CLOSED`, `OPEN`, `HALF_OPEN` 같은 상태를 가진 상태 머신이며, 호출 결과를 sliding window로 집계한다. retry가 "이번 요청을 다시 시도할지"의 판단이라면, circuit breaker는 "이 의존성을 지금 계속 호출해도 되는지"를 판단한다. 장애가 길어질 때 계속 재시도하면 장애 전파와 스레드/커넥션 고갈을 키울 수 있으므로, 실패율과 지연 시간이 기준을 넘으면 빠르게 실패시키는 경계가 필요하다.

운영 기준은 세 가지로 정리할 수 있다. 첫째, 연결 timeout과 응답 timeout을 분리해 어떤 단계가 느린지 관측한다. 둘째, retry는 멱등성과 예외 종류를 기준으로 좁게 건다. 셋째, circuit breaker와 fallback은 사용자에게 반환할 degraded response가 있는지까지 포함해 설계한다. `WebClient`의 안정성은 특정 연산자 암기가 아니라, 외부 의존성의 실패를 어디서 자르고 어떤 신호로 남길지 결정하는 문제다.

## 참고 링크

- [Spring Framework Reference: WebClient](https://docs.spring.io/spring-framework/reference/web/webflux-webclient.html)
- [Reactor Netty Reference: HTTP Client Timeout](https://projectreactor.io/docs/netty/release/reference/http-client.html#_httpclient_timeout)
- [Project Reactor Reference: Handling Errors and retryWhen](https://projectreactor.io/docs/core/release/reference/coreFeatures/error-handling.html)
- [Resilience4j Reference: CircuitBreaker](https://resilience4j.readme.io/docs/circuitbreaker)
