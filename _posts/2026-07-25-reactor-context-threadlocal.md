---
title: "Reactor Context와 ThreadLocal: WebFlux에서 요청 정보를 전달하는 기준"
date: 2026-07-25 09:00:00 +0900
categories: [technical-knowledge, backend-knowledge]
tags: [reactor, webflux, context, threadlocal, observability]
---

WebFlux 기반 서비스에서는 요청 ID, 인증 주체, trace 정보처럼 요청 범위의 값을 여러 계층으로 전달해야 한다. 전통적인 Servlet/MVC 코드에서는 `ThreadLocal`이나 MDC가 흔한 선택이지만, Reactor 기반의 비동기 파이프라인에서는 같은 요청이 항상 같은 스레드에서 처리된다는 가정을 둘 수 없다. 따라서 WebFlux에서는 "어떤 값은 Reactor `Context`에 두고, 어떤 경계에서만 `ThreadLocal`과 연결할 것인가"를 명확히 정해야 한다.

## 공식 문서 기준 핵심 개념

Reactor `Context`는 key-value를 담는 불변 컨텍스트이며, 데이터 신호와 별개로 `Subscriber`에 연결된다. 값을 쓰려면 보통 체인의 아래쪽, 즉 구독에 가까운 위치에서 `contextWrite`를 사용하고, 읽을 때는 `deferContextual`이나 `transformDeferredContextual` 같은 API를 사용한다. 위치가 잘못되면 위쪽 연산자가 값을 보지 못할 수 있다.

Spring WebClient 문서도 같은 기준을 따른다. 단일 요청의 필터 체인 안에서만 필요한 값은 request attributes로 충분하지만, `flatMap`으로 이어지는 중첩 요청이나 이후 단계까지 전달해야 하는 값은 Reactor `Context`를 사용해야 한다. WebFlux의 동시성 모델은 작은 event loop 스레드 풀을 전제로 하므로, 스레드에 값을 붙이는 방식은 설계의 기본값이 되기 어렵다.

<svg viewBox="0 0 860 300" role="img" aria-label="Reactor Context and ThreadLocal boundary" xmlns="http://www.w3.org/2000/svg">
  <rect x="20" y="30" width="820" height="240" rx="8" fill="#f8fafc" stroke="#cbd5e1"/>
  <text x="44" y="70" font-size="22" font-family="Arial, sans-serif" fill="#0f172a">WebFlux request context propagation</text>
  <rect x="55" y="105" width="170" height="76" rx="6" fill="#dbeafe" stroke="#60a5fa"/>
  <text x="80" y="138" font-size="15" font-family="Arial, sans-serif" fill="#1e3a8a">HTTP request</text>
  <text x="80" y="160" font-size="13" font-family="Arial, sans-serif" fill="#1e40af">request id</text>
  <path d="M225 143 H330" stroke="#334155" stroke-width="2" marker-end="url(#arrow)"/>
  <rect x="330" y="105" width="190" height="76" rx="6" fill="#dcfce7" stroke="#4ade80"/>
  <text x="365" y="136" font-size="15" font-family="Arial, sans-serif" fill="#14532d">Reactor Context</text>
  <text x="365" y="160" font-size="13" font-family="Arial, sans-serif" fill="#166534">subscriber-scoped values</text>
  <path d="M520 143 H625" stroke="#334155" stroke-width="2" marker-end="url(#arrow)"/>
  <rect x="625" y="105" width="170" height="76" rx="6" fill="#fee2e2" stroke="#f87171"/>
  <text x="655" y="136" font-size="15" font-family="Arial, sans-serif" fill="#7f1d1d">ThreadLocal/MDC</text>
  <text x="655" y="160" font-size="13" font-family="Arial, sans-serif" fill="#991b1b">boundary only</text>
  <path d="M425 181 V226 H710 V181" fill="none" stroke="#64748b" stroke-width="2" stroke-dasharray="6 5"/>
  <text x="455" y="250" font-size="14" font-family="Arial, sans-serif" fill="#334155">Micrometer Context Propagation links context mechanisms at integration boundaries.</text>
  <defs>
    <marker id="arrow" markerWidth="10" markerHeight="10" refX="9" refY="3" orient="auto" markerUnits="strokeWidth">
      <path d="M0,0 L0,6 L9,3 z" fill="#334155"/>
    </marker>
  </defs>
</svg>

## 언제 적합하고 언제 과한가

Reactor `Context`는 요청 단위 메타데이터를 라이브러리, 필터, WebClient 호출까지 전달해야 할 때 적합하다. 예를 들어 correlation ID를 로그나 하위 HTTP 요청 헤더에 반영해야 한다면, 컨텍스트 키를 하나 정하고 경계 계층에서만 읽고 쓰는 편이 명확하다. 반대로 도메인 로직의 필수 입력값이라면 숨은 컨텍스트보다 명시적 파라미터가 낫다. 값이 없으면 비즈니스 결과가 달라지는 정보는 함수 시그니처에 드러나야 테스트와 코드 리뷰가 쉬워진다.

`ThreadLocal`을 완전히 금지할 필요는 없다. 로깅 MDC, tracing, 일부 관측성 라이브러리처럼 imperative API가 `ThreadLocal`을 전제로 하는 경우가 있다. 이때 Micrometer Context Propagation은 `ThreadLocalAccessor`, `ContextSnapshot`, `ContextRegistry` 같은 추상화로 Reactor `Context`와 `ThreadLocal` 사이의 값을 옮기는 역할을 한다. Reactor는 `io.micrometer:context-propagation` SPI를 지원하며, 기본 모드와 `Hooks.enableAutomaticContextPropagation()`으로 켜는 automatic mode를 제공한다.

## 개념 예시

아래 코드는 실행 가능한 완성 코드가 아니라 WebClient 필터에서 요청 ID를 Reactor `Context`로 전달하고 읽는 흐름을 보여주는 개념 예시다.

```kotlin
private const val REQUEST_ID = "requestId"

val client = WebClient.builder()
    .filter { request, next ->
        Mono.deferContextual { ctx ->
            val requestId = ctx.getOrDefault(REQUEST_ID, "unknown")
            val decorated = ClientRequest.from(request)
                .header("X-Request-Id", requestId)
                .build()
            next.exchange(decorated)
        }
    }
    .build()

fun callDownstream(requestId: String): Mono<String> {
    return client.get()
        .uri("https://example.org/orders")
        .retrieve()
        .bodyToMono(String::class.java)
        .contextWrite { it.put(REQUEST_ID, requestId) }
}
```

## 운영 시 주의할 점

첫째, `Context` 키는 문자열 남발보다 상수나 전용 타입으로 관리하는 편이 안전하다. 여러 라이브러리가 같은 문자열 키를 사용하면 의도하지 않은 충돌이 생길 수 있다. 둘째, `contextWrite`의 위치를 테스트해야 한다. Reactor 문서의 예시처럼 같은 체인 안에서도 쓰는 위치에 따라 읽는 연산자가 보는 값이 달라진다. 셋째, automatic context propagation은 편하지만 공짜가 아니다. Reactor 문서는 `ThreadLocal` 접근이 reactive pipeline 성능에 의미 있는 영향을 줄 수 있다고 설명한다. 고성능 경로에서는 로그용 메타데이터만 컨텍스트에 두고, 핵심 도메인 값은 명시적으로 전달하는 기준이 더 예측 가능하다.

정리하면 WebFlux의 요청 범위 정보 전달은 `ThreadLocal`을 기본 저장소로 보는 방식에서 탈피해야 한다. Reactor `Context`를 요청 메타데이터의 기본 통로로 두고, MDC나 tracing처럼 imperative 생태계와 접하는 경계에서만 context propagation을 적용하는 방식이 운영과 테스트 모두에서 명확하다.

## 참고 링크

- [Project Reactor Documentation](https://projectreactor.io/docs)
- [Reactor Core Reference Guide - Adding a Context to a Reactive Sequence](https://docs.spring.io/projectreactor/reactor-core/docs/current/reference/html/#context)
- [Reactor Core Reference Guide - Context-Propagation Support](https://docs.spring.io/projectreactor/reactor-core/docs/current/reference/html/#context-propagation)
- [Spring Framework - WebClient Context](https://docs.spring.io/spring-framework/reference/web/webflux-webclient/client-context.html)
- [Spring Framework - WebFlux Concurrency Model](https://docs.spring.io/spring-framework/reference/web/webflux/new-framework.html#webflux-concurrency-model)
- [Micrometer Context Propagation - Purpose](https://docs.micrometer.io/context-propagation/reference/purpose.html)
