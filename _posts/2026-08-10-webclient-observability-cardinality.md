---
title: "WebClient 관측성에서 URI 태그를 설계하는 기준"
date: 2026-08-10 09:10:00 +0900
categories: [technical-knowledge, backend-knowledge]
tags: [spring-webflux, webclient, observability, micrometer, opentelemetry]
---

`WebClient` 관측성은 "HTTP 호출 시간이 얼마나 걸렸는가"를 보는 기능에 그치지 않는다. 어떤 외부 API가 느려졌는지, timeout과 retry가 어느 경로에서 늘어나는지, 특정 downstream 장애가 전체 요청 지연으로 번지는지를 빠르게 좁히기 위한 계약이다. 이때 가장 자주 문제가 되는 지점은 metric tag, 특히 URI 값을 얼마나 구체적으로 넣을지에 대한 판단이다.

Spring Framework의 WebClient 문서는 client request 관측을 지원하며, `ClientRequestObservationConvention`을 통해 observation 이름과 low-cardinality key values를 구성할 수 있다고 설명한다. Micrometer Observation 모델도 tag를 low-cardinality와 high-cardinality로 나누어 본다. low-cardinality 값은 metric 차원으로 집계해도 폭발하지 않는 값이고, high-cardinality 값은 trace나 event에는 유용하지만 metric label로 쓰기에는 위험한 값이다.

<figure class="post-diagram">
  <svg viewBox="0 0 860 320" role="img" aria-labelledby="obs-title obs-desc" xmlns="http://www.w3.org/2000/svg">
    <title id="obs-title">WebClient observation cardinality boundary</title>
    <desc id="obs-desc">A WebClient request creates metrics with low-cardinality tags and traces with high-cardinality details.</desc>
    <defs>
      <marker id="arrow-obs" markerWidth="10" markerHeight="10" refX="9" refY="5" orient="auto">
        <path d="M1,1 L9,5 L1,9 Z" fill="#475569" />
      </marker>
      <style>
        .box { rx: 8; stroke-width: 2; }
        .client { fill: #eef6ff; stroke: #2563eb; }
        .metric { fill: #ecfdf5; stroke: #059669; }
        .trace { fill: #fff7ed; stroke: #ea580c; }
        .warn { fill: #fef2f2; stroke: #dc2626; }
        .label { font: 700 15px -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; fill: #0f172a; }
        .small { font: 13px -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; fill: #475569; }
        .line { stroke: #475569; stroke-width: 3; fill: none; marker-end: url(#arrow-obs); }
      </style>
    </defs>
    <rect x="34" y="66" width="165" height="82" class="box client" />
    <text x="116" y="100" text-anchor="middle" class="label">WebClient</text>
    <text x="116" y="124" text-anchor="middle" class="small">outbound request</text>
    <path d="M199 107 H282" class="line" />
    <rect x="282" y="36" width="228" height="112" class="box metric" />
    <text x="396" y="73" text-anchor="middle" class="label">Metrics</text>
    <text x="396" y="98" text-anchor="middle" class="small">method=GET</text>
    <text x="396" y="120" text-anchor="middle" class="small">uri=/users/{id}</text>
    <path d="M510 92 H622" class="line" />
    <rect x="622" y="36" width="200" height="112" class="box warn" />
    <text x="722" y="73" text-anchor="middle" class="label">Avoid label blow-up</text>
    <text x="722" y="98" text-anchor="middle" class="small">/users/1001</text>
    <text x="722" y="120" text-anchor="middle" class="small">/users/1002 ...</text>
    <path d="M116 148 V221 H282" class="line" />
    <rect x="282" y="184" width="228" height="94" class="box trace" />
    <text x="396" y="220" text-anchor="middle" class="label">Traces / logs</text>
    <text x="396" y="244" text-anchor="middle" class="small">request id, full url</text>
    <path d="M510 231 H622" class="line" />
    <rect x="622" y="184" width="200" height="94" class="box client" />
    <text x="722" y="220" text-anchor="middle" class="label">Drill down</text>
    <text x="722" y="244" text-anchor="middle" class="small">sampled detail only</text>
  </svg>
  <figcaption>metric에는 집계 가능한 템플릿을, trace와 log에는 필요한 상세 값을 분리해 넣어야 한다.</figcaption>
</figure>

가장 안전한 기본값은 metric의 `uri` 태그를 실제 호출 URL이 아니라 route template 수준으로 유지하는 것이다. 예를 들어 `/users/123/orders/987`을 그대로 label로 넣으면 사용자와 주문 수만큼 시계열이 늘어난다. 반대로 `/users/{userId}/orders/{orderId}`처럼 템플릿을 유지하면 endpoint별 지연, 오류율, retry 증가를 안정적으로 집계할 수 있다. Spring Framework의 WebClient 관측 문서도 low-cardinality `uri`를 "HTTP request에 사용된 URI template"으로 두고, 전체 request URI는 high-cardinality `http.url`로 분리한다.

Spring Boot Actuator를 함께 쓰는 서비스에서는 observation registry, meter registry, tracing exporter가 한 흐름으로 연결될 수 있다. 따라서 `WebClient`에 단순 filter를 추가해 시간을 재는 것보다, Observation API와 기존 instrumentation을 우선 활용하는 편이 낫다. 필요한 경우에는 custom `ClientRequestObservationConvention`으로 service name, API group, logical route 같은 제한된 tag만 추가한다.

```kotlin
// 개념 예시: metric에는 템플릿 수준의 URI만 태그로 남긴다.
class ExternalApiObservationConvention : ClientRequestObservationConvention {
    override fun getName(): String = "http.client.requests"

    override fun getLowCardinalityKeyValues(context: ClientRequestObservationContext): KeyValues {
        val request = context.carrier ?: return KeyValues.empty()
        val route = request.attribute("routeTemplate")
            .map { it.toString() }
            .orElse("UNKNOWN")

        return KeyValues.of(
            KeyValue.of("client.name", "payment-api"),
            KeyValue.of("method", request.method().name()),
            KeyValue.of("uri", route)
        )
    }
}
```

운영에서는 세 가지를 먼저 정해야 한다. 첫째, metric label로 허용할 값의 목록을 제한한다. HTTP method, client name, route template, status group 정도가 일반적으로 충분하다. 둘째, full URL, query string, tenant id, user id, order id는 metric label이 아니라 trace attribute나 구조화 로그로 보낸다. 셋째, timeout, retry, circuit breaker 지표와 같은 cardinality 규칙을 공유한다. outbound HTTP metric만 잘 설계해도 retry metric이 고유 ID별로 폭증하면 대시보드는 다시 쓰기 어려워진다.

주의할 점도 있다. URI 템플릿이 사라지는 방식으로 `WebClient`를 만들면 관측 라이브러리가 route를 추론하지 못하고 실제 path에 가까운 값을 남길 수 있다. 동적으로 URL 문자열을 조립하는 코드가 많다면 builder 단계에서 route template을 attribute로 넘기거나, API client wrapper에서 endpoint 이름을 명시하는 방식이 필요하다. 또한 query parameter에는 검색어, 이메일, token 같은 민감 값이 섞일 수 있으므로 metric과 log 양쪽에서 기본 비수집 정책을 두는 편이 안전하다.

정리하면 WebClient 관측성의 핵심은 더 많은 값을 붙이는 것이 아니라, 집계 가능한 값과 조사용 상세 값을 분리하는 데 있다. metric은 적은 수의 안정적인 label로 전체 상태를 보여주고, trace와 log는 샘플링된 요청을 따라가며 상세 원인을 확인하게 만든다. 이 경계를 먼저 정해두면 외부 API 장애, retry 폭증, timeout 증가를 분석할 때 관측 시스템 자체가 병목이 되는 상황을 피할 수 있다.

## 참고 링크

- [Spring Framework Reference - WebClient](https://docs.spring.io/spring-framework/reference/web/webflux-webclient.html)
- [Spring Framework Reference - WebClient Observation](https://docs.spring.io/spring-framework/reference/integration/observability.html#observability.http-client)
- [Spring Boot Reference - Observability](https://docs.spring.io/spring-boot/reference/actuator/observability.html)
- [Micrometer Documentation - Observation Concepts](https://docs.micrometer.io/micrometer/reference/observation/introduction.html)
- [OpenTelemetry Semantic Conventions - HTTP](https://opentelemetry.io/docs/specs/semconv/http/)
