---
title: "HTTP 캐시에서 ETag와 조건부 요청을 설계하는 기준"
date: 2026-07-30 09:52:00 +0900
categories: [technical-knowledge, backend-knowledge]
tags: [http, cache, etag, spring-webflux, api-design]
---

HTTP 캐시는 단순히 응답을 오래 저장하는 기능이 아니다. API 서버, 브라우저, CDN, 프록시가 같은 표현(representation)을 다시 내려받아야 하는지 합의하는 프로토콜이다. 그 중심에 `Cache-Control`, `ETag`, `If-None-Match`, `304 Not Modified`가 있다.

Spring WebFlux 문서는 HTTP 캐싱을 `Cache-Control` 응답 헤더와 `Last-Modified`, `ETag` 같은 조건부 요청 헤더의 조합으로 설명한다. `Cache-Control`은 캐시가 응답을 언제까지 재사용할 수 있는지 알려주고, `ETag`는 클라이언트가 보유한 표현이 아직 같은지 검증하는 식별자 역할을 한다. 내용이 바뀌지 않았다면 서버는 본문 없이 `304 Not Modified`를 돌려줄 수 있다.

<svg viewBox="0 0 760 230" role="img" aria-label="ETag conditional request flow" xmlns="http://www.w3.org/2000/svg">
  <rect x="20" y="24" width="170" height="58" rx="8" fill="#eef6ff" stroke="#4f8cc9"/>
  <text x="105" y="58" text-anchor="middle" font-size="15" fill="#1f3f5f">Client Cache</text>
  <rect x="295" y="24" width="170" height="58" rx="8" fill="#f6f7f9" stroke="#89909a"/>
  <text x="380" y="58" text-anchor="middle" font-size="15" fill="#2d333b">API Server</text>
  <rect x="570" y="24" width="170" height="58" rx="8" fill="#f1fbf4" stroke="#5da86b"/>
  <text x="655" y="58" text-anchor="middle" font-size="15" fill="#244b2a">Resource State</text>
  <path d="M190 118 H292" stroke="#4f8cc9" stroke-width="2" marker-end="url(#arrow)"/>
  <text x="241" y="108" text-anchor="middle" font-size="13" fill="#334155">If-None-Match: v7</text>
  <path d="M465 118 H568" stroke="#89909a" stroke-width="2" marker-end="url(#arrow)"/>
  <text x="516" y="108" text-anchor="middle" font-size="13" fill="#334155">compare current tag</text>
  <path d="M570 166 H466" stroke="#5da86b" stroke-width="2" marker-end="url(#arrow)"/>
  <text x="518" y="156" text-anchor="middle" font-size="13" fill="#334155">v7 unchanged</text>
  <path d="M295 166 H192" stroke="#4f8cc9" stroke-width="2" marker-end="url(#arrow)"/>
  <text x="244" y="156" text-anchor="middle" font-size="13" fill="#334155">304, no body</text>
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#64748b"/>
    </marker>
  </defs>
</svg>

핵심 판단은 `max-age`와 검증자를 분리해서 보는 것이다. `max-age`가 길면 캐시는 원 서버에 묻지 않고 응답을 재사용할 수 있다. 반대로 `no-cache`는 저장 자체를 금지한다는 뜻이 아니라, 재사용 전에 원 서버 검증을 요구한다는 뜻이다. 저장 자체를 막아야 하는 민감 응답에는 `no-store`가 필요하다. RFC 9111도 캐시가 저장 응답을 재사용하려면 신선하거나, stale 사용이 허용되거나, 성공적으로 검증되어야 한다고 정의한다.

ETag는 변경 여부를 빠르게 판단할 수 있을 때 특히 유용하다. 예를 들어 리소스에 버전 컬럼, 갱신 시각, 콘텐츠 해시가 있다면 이를 기반으로 안정적인 태그를 만들 수 있다. 클라이언트가 다음 요청에 `If-None-Match`를 보내면 서버는 현재 태그와 비교해 `GET`/`HEAD`에는 `304 Not Modified`, 그 외 조건부 변경 요청에는 상황에 따라 `412 Precondition Failed`를 사용할 수 있다. RFC 9110은 `If-None-Match`를 `If-Modified-Since`보다 더 정확한 검증 조건으로 다룬다.

Spring에서는 컨트롤러가 `ResponseEntity`에 `CacheControl`과 `eTag`를 함께 지정할 수 있다. 다음 코드는 설명을 위한 개념 예시다.

```kotlin
@GetMapping("/articles/{id}")
fun article(@PathVariable id: Long): ResponseEntity<ArticleView> {
    val article = articleReader.findView(id)
    val tag = "\"article-${article.version}\""

    return ResponseEntity.ok()
        .cacheControl(CacheControl.noCache())
        .eTag(tag)
        .body(article)
}
```

`ShallowEtagHeaderFilter`처럼 응답 본문으로 얕은 ETag를 계산하는 방식도 있다. 다만 이 방식은 이미 응답을 렌더링한 뒤 태그를 계산하므로 네트워크 대역폭은 줄여도 서버 CPU나 데이터 조회 비용은 크게 줄이지 못한다. 조회 비용이 큰 API라면 필터에 맡기기보다 리소스의 버전 정보를 먼저 읽고 조건부 요청을 조기에 판정하는 편이 낫다.

운영 관점에서는 세 가지를 구분해야 한다. 첫째, 정적 자산처럼 파일명이 해시를 포함한 리소스는 긴 `max-age`와 `immutable`이 적합하다. 둘째, 사용자별 응답이나 권한에 따라 달라지는 API는 `private`, 짧은 신선도, `Vary` 헤더를 검토해야 한다. 셋째, 결제·주문·개인정보처럼 저장되면 안 되는 응답에는 `no-store`를 사용해야 하며, 이것만으로 보안이 완성된다고 보면 안 된다.

ETag 설계는 캐시 성능 최적화이면서 동시에 API 의미론 설계다. 무조건 캐시 시간을 늘리는 대신, 어떤 응답을 저장해도 되는지, 언제 원 서버 검증이 필요한지, 변경 충돌을 어떤 상태 코드로 표현할지를 함께 정해야 한다.

## 참고 링크

- [Spring Framework WebFlux HTTP Caching](https://docs.spring.io/spring-framework/reference/web/webflux/caching.html)
- [Spring Framework `CacheControl` API](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/http/CacheControl.html)
- [Spring Framework `ShallowEtagHeaderFilter` API](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/filter/ShallowEtagHeaderFilter.html)
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)
- [RFC 9111: HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111.html)
