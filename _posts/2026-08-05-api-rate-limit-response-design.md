---
title: "HTTP API Rate Limit 응답을 설계하는 기준"
date: 2026-08-05 09:03:00 +0900
categories: [technical-knowledge, backend-knowledge]
tags: [http, api-design, rate-limit, webflux, backend]
---

Rate limit은 단순히 요청을 막는 장치가 아니다. 서버가 보호해야 할 자원, 클라이언트가 재시도해야 할 시점, 사용자나 애플리케이션별 공정성을 HTTP 계약으로 드러내는 설계 문제다. 같은 제한이라도 응답 코드와 헤더가 불명확하면 클라이언트는 즉시 재시도하거나, 과도하게 대기하거나, 장애를 일반 오류로 오해할 수 있다.

RFC 6585는 `429 Too Many Requests`를 rate limiting 상황에 맞는 상태 코드로 정의한다. 응답 본문에는 제한 조건을 설명할 수 있고, `Retry-After` 헤더로 언제 다시 요청할 수 있는지 알려줄 수 있다. 다만 이 RFC는 서버가 사용자를 어떻게 식별하고 요청을 어떻게 집계하는지는 정의하지 않는다. 따라서 rate limit 정책은 인증 주체, IP, client credential, tenant, route 비용 같은 서비스 기준으로 별도 설계해야 한다.

<figure class="post-diagram">
  <svg viewBox="0 0 820 300" role="img" aria-labelledby="rate-limit-title rate-limit-desc" xmlns="http://www.w3.org/2000/svg">
    <title id="rate-limit-title">API rate limit response flow</title>
    <desc id="rate-limit-desc">A request passes identity and quota checks. Allowed requests continue to handlers, while exhausted quota returns 429 with Retry-After and rate limit hints.</desc>
    <defs>
      <marker id="arrow-rate-limit" markerWidth="10" markerHeight="10" refX="9" refY="5" orient="auto">
        <path d="M1,1 L9,5 L1,9 Z" fill="#475569" />
      </marker>
      <style>
        .box { rx: 8; stroke-width: 2; }
        .client { fill: #eef6ff; stroke: #3b82f6; }
        .quota { fill: #ecfdf5; stroke: #10b981; }
        .handler { fill: #fff7ed; stroke: #f97316; }
        .deny { fill: #fef2f2; stroke: #ef4444; }
        .label { font: 700 15px -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; fill: #0f172a; }
        .small { font: 13px -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; fill: #475569; }
        .line { stroke: #475569; stroke-width: 3; fill: none; marker-end: url(#arrow-rate-limit); }
      </style>
    </defs>
    <rect x="34" y="52" width="150" height="74" class="box client" />
    <text x="109" y="84" text-anchor="middle" class="label">Client</text>
    <text x="109" y="106" text-anchor="middle" class="small">request</text>
    <path d="M184 89 H270" class="line" />
    <rect x="270" y="42" width="210" height="94" class="box quota" />
    <text x="375" y="76" text-anchor="middle" class="label">Quota boundary</text>
    <text x="375" y="100" text-anchor="middle" class="small">identity, policy, cost</text>
    <path d="M480 89 H575" class="line" />
    <rect x="575" y="52" width="170" height="74" class="box handler" />
    <text x="660" y="84" text-anchor="middle" class="label">Handler</text>
    <text x="660" y="106" text-anchor="middle" class="small">2xx response</text>
    <path d="M375 136 V202" class="line" />
    <rect x="252" y="202" width="246" height="70" class="box deny" />
    <text x="375" y="231" text-anchor="middle" class="label">429 Too Many Requests</text>
    <text x="375" y="253" text-anchor="middle" class="small">Retry-After + safe hints</text>
  </svg>
  <figcaption>rate limit은 handler 내부 예외보다 인증·정책·비용 계산 이후의 공통 경계에서 다루는 편이 일관적이다.</figcaption>
</figure>

최근 IETF HTTPAPI 작업 그룹의 `RateLimit` 헤더 Internet-Draft는 `RateLimit-Policy`와 `RateLimit` 필드로 quota 정책과 남은 한도를 전달하는 방향을 제안한다. 2026년 5월 기준 최신 문서는 아직 RFC가 아니므로, 공개 API에서 표준 헤더처럼 단정하기보다 문서화된 호환 계약으로 다루는 것이 안전하다. 특히 `Retry-After`와 함께 내려갈 때는 `Retry-After`를 우선한다는 규칙, 캐시된 응답의 quota 정보는 낡았을 수 있다는 점, quota 수치가 서비스 내부 용량을 과하게 노출할 수 있다는 보안 고려사항이 중요하다.

Spring WebFlux에서는 `WebFilter`가 이런 횡단 관심사를 두기 좋은 위치다. `WebFilter`는 요청 처리 체인 앞에서 `ServerWebExchange`를 보고 다음 필터나 handler로 넘길지 결정한다. 인증 이후의 사용자 식별자, route별 비용, distributed quota store 조회가 필요하다면 필터 순서와 실패 시 응답 형식을 명확히 잡아야 한다.

다음은 설명을 위한 Kotlin 개념 예시다. 실제 quota 저장소와 동시성 제어는 Redis, gateway, sidecar, 별도 rate limit 서비스 등 운영 구조에 맞게 구현해야 한다.

```kotlin
class RateLimitWebFilter(
    private val limiter: ReactiveRateLimiter
) : WebFilter {
    override fun filter(exchange: ServerWebExchange, chain: WebFilterChain): Mono<Void> {
        val key = exchange.request.headers.getFirst("X-Client-Id") ?: "anonymous"

        return limiter.consume(key, cost = 1)
            .flatMap { decision ->
                if (decision.allowed) {
                    exchange.response.headers.add("RateLimit", "\"default\";r=${decision.remaining};t=${decision.resetSeconds}")
                    chain.filter(exchange)
                } else {
                    exchange.response.statusCode = HttpStatus.TOO_MANY_REQUESTS
                    exchange.response.headers.add("Retry-After", decision.resetSeconds.toString())
                    exchange.response.setComplete()
                }
            }
    }
}
```

운영 설계에서는 세 가지를 먼저 결정해야 한다. 첫째, 제한 단위는 사용자, 조직, client credential, IP 중 무엇인지 정해야 한다. 둘째, 모든 요청을 같은 비용으로 볼지, 무거운 route에 더 높은 cost를 둘지 정해야 한다. 셋째, 초과 응답을 캐시하거나 중간 프록시가 재시도하지 않도록 `429` 응답의 캐시 정책과 클라이언트 SDK의 backoff/jitter 정책을 함께 맞춰야 한다.

Rate limit 응답은 서버 방어 로직이면서 클라이언트와의 협업 신호다. 제한을 숨기면 클라이언트는 추측으로 재시도하고, 제한을 과하게 공개하면 공격자에게 용량 정보를 줄 수 있다. 좋은 API는 `429`, `Retry-After`, 제한 힌트, 문제 설명을 필요한 만큼만 제공하고, 정책 자체는 문서와 관측 지표로 검증 가능하게 만든다.

## 참고 링크

- [RFC 6585 - Additional HTTP Status Codes](https://www.rfc-editor.org/rfc/rfc6585.html)
- [RFC 9110 - HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)
- [IETF Datatracker - RateLimit header fields for HTTP Internet-Draft](https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/)
- [Spring Framework WebFlux](https://docs.spring.io/spring-framework/reference/web/webflux.html)
- [Spring Framework `WebFilter` API](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/server/WebFilter.html)
