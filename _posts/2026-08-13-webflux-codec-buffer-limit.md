---
title: "Spring WebFlux의 maxInMemorySize를 무작정 키우지 않는 기준"
date: 2026-08-13 09:08:00 +0900
categories: [technical-knowledge, backend-knowledge]
tags: [spring-webflux, webclient, databuffer, codec, memory]
---

Spring WebFlux에서 큰 JSON 응답이나 파일 업로드를 다루다 보면 `DataBufferLimitException`을 만날 수 있다. 이때 가장 쉬운 대응은 `maxInMemorySize`를 크게 올리는 것이다. 하지만 이 설정은 단순한 편의 옵션이 아니라, HTTP body를 어디까지 메모리에 모아도 되는지 정하는 운영 한계선이다. 한도를 키우기 전에 "왜 body가 집계되는가"와 "스트리밍으로 바꿀 수 있는가"를 먼저 판단해야 한다.

Spring Framework 문서는 WebFlux codec이 `Decoder`, `HttpMessageReader`, `HttpMessageWriter`를 통해 HTTP 본문을 객체로 변환한다고 설명한다. 이 과정에서 일부 decoder나 reader는 입력 스트림 전체 또는 일부를 메모리에 버퍼링한다. 예를 들어 `@RequestBody byte[]`, `application/x-www-form-urlencoded`, 단일 객체로 읽는 큰 JSON 응답은 집계가 필요할 수 있다. 반대로 NDJSON, SSE, multipart streaming처럼 항목 단위로 처리할 수 있는 흐름에서는 한 요청 전체가 아니라 한 객체 또는 한 파트의 크기가 기준이 된다.

<figure class="post-diagram">
  <svg viewBox="0 0 860 340" role="img" aria-labelledby="buffer-title buffer-desc" xmlns="http://www.w3.org/2000/svg">
    <title id="buffer-title">WebFlux codec memory boundary</title>
    <desc id="buffer-desc">HTTP body can be aggregated into one object or streamed item by item. maxInMemorySize protects the aggregation boundary.</desc>
    <defs>
      <marker id="arrow-buffer" markerWidth="10" markerHeight="10" refX="9" refY="5" orient="auto">
        <path d="M1,1 L9,5 L1,9 Z" fill="#475569" />
      </marker>
      <style>
        .box { rx: 8; stroke-width: 2; }
        .entry { fill: #eef6ff; stroke: #2563eb; }
        .codec { fill: #ecfdf5; stroke: #059669; }
        .limit { fill: #fff7ed; stroke: #ea580c; }
        .warn { fill: #fef2f2; stroke: #dc2626; }
        .label { font: 700 15px -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; fill: #0f172a; }
        .small { font: 13px -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; fill: #475569; }
        .line { stroke: #475569; stroke-width: 3; fill: none; marker-end: url(#arrow-buffer); }
      </style>
    </defs>
    <rect x="34" y="45" width="150" height="76" class="box entry" />
    <text x="109" y="78" text-anchor="middle" class="label">HTTP body</text>
    <text x="109" y="101" text-anchor="middle" class="small">DataBuffer stream</text>
    <path d="M184 83 H270" class="line" />
    <rect x="270" y="28" width="210" height="110" class="box codec" />
    <text x="375" y="66" text-anchor="middle" class="label">Codec / Reader</text>
    <text x="375" y="90" text-anchor="middle" class="small">decode to object</text>
    <text x="375" y="112" text-anchor="middle" class="small">may aggregate bytes</text>
    <path d="M480 83 H590" class="line" />
    <rect x="590" y="45" width="215" height="76" class="box limit" />
    <text x="697" y="78" text-anchor="middle" class="label">maxInMemorySize</text>
    <text x="697" y="101" text-anchor="middle" class="small">aggregation guardrail</text>
    <path d="M375 138 V212 H270" class="line" />
    <rect x="270" y="212" width="210" height="86" class="box codec" />
    <text x="375" y="247" text-anchor="middle" class="label">Streaming path</text>
    <text x="375" y="270" text-anchor="middle" class="small">Flux&lt;Item&gt; / PartEvent</text>
    <path d="M480 255 H590" class="line" />
    <rect x="590" y="212" width="215" height="86" class="box warn" />
    <text x="697" y="247" text-anchor="middle" class="label">Design choice</text>
    <text x="697" y="270" text-anchor="middle" class="small">limit per item, not whole API</text>
  </svg>
  <figcaption>`maxInMemorySize`는 WebFlux가 body를 객체로 만들기 위해 모으는 메모리 경계를 제한한다.</figcaption>
</figure>

`DataBufferLimitException`의 Javadoc은 누적 소비 byte 수가 미리 설정된 한도를 넘을 때 발생한다고 설명한다. `DataBufferUtils.join`처럼 buffer를 모으는 경우뿐 아니라, Jackson async parsing이나 SSE 파싱처럼 buffer 자체는 해제됐지만 파싱된 표현을 내부적으로 모으는 경우에도 발생할 수 있다. 즉 이 예외는 "메모리가 부족하다"는 신호만이 아니라, 현재 endpoint나 client 호출이 body를 모으는 방식으로 설계됐다는 신호이기도 하다.

설정 지점은 서버와 클라이언트가 다르다. Spring WebFlux 서버에서는 `ServerCodecConfigurer`를 통해 기본 codec의 `maxInMemorySize`를 조정할 수 있다. Spring Boot 자동 설정을 쓰는 경우에는 `spring.http.codecs.max-in-memory-size`가 auto-configured WebFlux server와 WebClient 인스턴스에 적용된다. multipart에는 별도 속성이 있다. `spring.webflux.multipart.max-in-memory-size`는 파일이 아닌 part의 메모리 한도이며, 파일 part에서는 디스크로 쓰기 시작하는 임계값으로 동작한다. 따라서 "전체 업로드 최대 크기"와 "메모리에 둘 수 있는 part 크기"를 같은 설정으로 보면 안 된다.

다음은 설명을 위한 Kotlin 개념 예시다. 작은 JSON API는 한도를 명시적으로 두고, 큰 데이터는 `Flux`로 항목 단위 처리하도록 경계를 나누는 방식이다.

```kotlin
// 개념 예시: 실제 값은 payload 분포와 동시 요청 수를 기준으로 정한다.
@Configuration
class WebFluxCodecConfig : WebFluxConfigurer {
    override fun configureHttpMessageCodecs(configurer: ServerCodecConfigurer) {
        configurer.defaultCodecs().maxInMemorySize(512 * 1024)
    }
}

fun largeClient(): WebClient =
    WebClient.builder()
        .codecs { codecs ->
            codecs.defaultCodecs().maxInMemorySize(2 * 1024 * 1024)
        }
        .build()

fun readEvents(client: WebClient): Flux<Event> =
    client.get()
        .uri("/events.ndjson")
        .retrieve()
        .bodyToFlux(Event::class.java)
```

운영에서 먼저 볼 것은 payload 크기의 분포다. 99퍼센타일이 300KB인데 한도가 256KB라면 512KB로 올리는 선택이 합리적일 수 있다. 그러나 일부 요청이 20MB까지 커진다면 전역 한도를 20MB로 키우는 순간 동시 요청 수만큼 heap 압박이 커진다. 이 경우에는 페이지네이션, 압축된 파일 업로드, object storage 직접 업로드, NDJSON/SSE 스트리밍, multipart streaming 같은 API 형태 변경을 검토해야 한다.

두 번째 기준은 실패를 숨기지 않는 것이다. `maxInMemorySize` 초과를 500으로 흘려보내면 호출자는 서버 장애로 이해한다. 요청 body가 정책보다 크다면 413 Payload Too Large처럼 명확한 응답으로 바꾸고, downstream 응답이 너무 커서 client codec에서 실패했다면 호출 대상 API의 응답 크기 계약을 점검해야 한다. WebClient 쪽에서만 한도를 키우면 서버 간 계약 문제를 클라이언트 메모리로 덮는 구조가 될 수 있다.

마지막으로, `DataBuffer`를 직접 다루는 코드는 별도 주의가 필요하다. Spring 문서는 Netty 같은 서버의 byte buffer가 pooled, reference-counted일 수 있고, 직접 소비한 buffer는 release가 필요하다고 설명한다. 일반 controller와 WebClient 사용자는 codec에 맡기는 편이 안전하지만, custom codec이나 raw `DataBuffer` 처리로 최적화하려면 cancel, error, discard 경로에서 누수가 없는지 테스트해야 한다.

정리하면 `maxInMemorySize`는 에러를 없애기 위해 올리는 숫자가 아니라 API body 처리 전략의 일부다. 작은 객체는 명확한 한도 안에서 집계하고, 큰 데이터는 스트리밍이나 외부 저장소 경유로 설계를 바꾼다. 한도를 올릴 때도 endpoint별 payload 분포, 동시 요청 수, multipart 정책, 실패 응답, 관측 지표를 함께 정해야 WebFlux의 non-blocking 장점이 메모리 병목으로 상쇄되지 않는다.

## 참고 링크

- [Spring Framework Reference - WebFlux Reactive Core, Codecs and Limits](https://docs.spring.io/spring-framework/reference/web/webflux/reactive-spring.html#webflux-codecs)
- [Spring Framework Reference - WebFlux Config, HTTP Message Codecs](https://docs.spring.io/spring/reference/6.2/web/webflux/config.html#webflux-config-message-codecs)
- [Spring Framework Javadoc - DataBufferLimitException](https://docs.spring.io/spring-framework/docs/7.0.5/javadoc-api/org/springframework/core/io/buffer/DataBufferLimitException.html)
- [Spring Boot Common Application Properties - HTTP codecs and WebFlux multipart](https://docs.enterprise.spring.io/spring-boot/appendix/application-properties/index.html)
