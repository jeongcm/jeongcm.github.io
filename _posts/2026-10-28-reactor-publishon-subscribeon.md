---
title: Spring WebFlux에서 Reactor의 publishOn / subscribeOn 실행 컨텍스트 전환
date: 2026-10-28 12:20:20 +09:00
categories: [technical-knowledge, backend-knowledge]
tags: [reactor, spring-webflux, reactivestreams, backpressure, multithreading]
---

반응형 스택에서 스레드 전환은 성능의 핵심이다. Spring WebFlux가 Reactor를 활성 연산자 모델로 사용하면서, 실행 컨텍스트를 언제 어떻게 옮길지는 개발자의 명시적 선택이 된다. Reactor에서 실행 컨텍스트를 옮기는 두 축은 publishOn과 subscribeOn이다.

publishOn은 연산자 체인 중간에서 실행 스레드 컨텍스트를 바꾼다. publishOn 이후 오는 연산자들이 지정한 Scheduler 위에서 실행된다. 다만 onNext 호출 자체는 순차적이라는 Reactive Streams 규약을 따르므로, 하나의 스레드에서 순차적으로 처리된다. 체인에서 publishOn 뒤에 오는 연산자만 영향을 받고, 앞에 있는 연산자는 영향을 받지 않는다.

subscribeOn은 구독 시점에 전체 체인의 시작 스레드를 바꾼다. 보통 소스 연산자 직후에 배치하며, 위치 변경이 동작에 거의 영향을 주지 않는다. 여러 개의 subscribeOn을 겹쳐 놓아도 가장 가까운 하나만 실질적으로 작용하므로, 나머지 구독 스레드 변경은 무의미하다.

두 가지 차이는 신호 흐름 방향으로 이해할 수 있다. publishOn은 상위에서 하위로 흐르는 onNext/onError/onComplete 신호를 따라 실행 컨텍스트를 옮기고, subscribeOn은 구독과 요청 신호, doFirst 같은 신호 발생 지점에만 작용한다.

운영 시 주의점이 있다. subscribeOn을 넣지 않고 publishOn만으로는 구독 스레드를 바꿀 수 없고, 기본은 구독한 스레드가 그대로 사용된다. 반대로 publishOn을 넣으려면 subscribeOn이 먼저 필요하지는 않다. 또한 백프레셔를 고려하지 않고 Long.MAX_VALUE의 무제한 요청을 주면 하류가 따라가지 못해 메모리 압력이 커질 수 있다. 병렬 처리에는 Schedulers.parallel(), 블로킹 경계 보호에는 Schedulers.boundedElastic()을 권장한다.

반응형 API에서 스레드 전환은 '어디에' 놓느냐가 동작을 결정한다. 체인 위치와 신호 방향을 함께 고려해야 같은 코드도 다른 실행 트레이스를 만든다.

<svg xmlns="http://www.w3.org/2000/svg" width="760" height="200" viewBox="0 0 760 200" role="img" aria-labelledby="reactor-title reactor-desc">
  <title id="reactor-title">Reactor 구독 체인: publishOn / subscribeOn</title>
  <desc id="reactor-desc">publishOn은 하위(onNext) 신호 방향으로 실행 스레드를 전환하고, subscribeOn은 구독/요청 신호 지점에만 작용한다.</desc>
  <rect x="20" y="40" width="110" height="60" rx="8" fill="#eef7ff" stroke="#3b82f6" stroke-width="2"/>
  <text x="75" y="62" text-anchor="middle" font-family="sans-serif" font-size="13" fill="#1e40af">Source: Range(3)</text>
  <rect x="150" y="40" width="130" height="60" rx="8" fill="#e6f7e6" stroke="#10b981" stroke-width="2"/>
  <text x="215" y="62" text-anchor="middle" font-family="sans-serif" font-size="13" fill="#065f46">subscribeOn(schedulerA)</text>
  <rect x="300" y="40" width="120" height="60" rx="8" fill="#fef3c7" stroke="#f59e0b" stroke-width="2"/>
  <text x="360" y="62" text-anchor="middle" font-family="sans-serif" font-size="13" fill="#92400e">publishOn(schedulerB)</text>
  <rect x="440" y="40" width="110" height="60" rx="8" fill="#eef7ff" stroke="#3b82f6" stroke-width="2"/>
  <text x="495" y="62" text-anchor="middle" font-family="sans-serif" font-size="13" fill="#1e40af">Operator 1</text>
  <rect x="570" y="40" width="120" height="60" rx="8" fill="#e6f7e6" stroke="#10b981" stroke-width="2"/>
  <text x="630" y="62" text-anchor="middle" font-family="sans-serif" font-size="13" fill="#065f46">publishOn(schedulerC)</text>
  <rect x="710" y="40" width="30" height="60" rx="8" fill="#f3f4f6" stroke="#6b7280" stroke-width="2"/>
  <text x="725" y="62" text-anchor="middle" font-family="sans-serif" font-size="12" fill="#374151">Subscriber</text>
  <defs>
    <marker id="arrow" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto"><polygon points="0 0, 6 3, 0 6"/></marker>
  </defs>
  <line x1="130" y1="70" x2="150" y2="70" stroke="#9ca3af" stroke-width="2" marker-end="url(#arrow)"/>
  <line x1="280" y1="70" x2="300" y2="70" stroke="#9ca3af" stroke-width="2" marker-end="url(#arrow)"/>
  <line x1="420" y1="70" x2="440" y2="70" stroke="#9ca3af" stroke-width="2" marker-end="url(#arrow)"/>
  <line x1="550" y1="70" x2="570" y2="70" stroke="#9ca3af" stroke-width="2" marker-end="url(#arrow)"/>
  <text x="380" y="185" text-anchor="middle" font-family="sans-serif" font-size="11" fill="#6b7280">subscribeOn: 구독/요청 신호 방향에 영향 없음</text>
  <text x="380" y="198" text-anchor="middle" font-family="sans-serif" font-size="11" fill="#6b7280">publishOn: onNext 하위 신호 방향에 실행 스레드 전환</text>
</svg>
실행 컨텍스트 전환의 핵심을 정리하면 다음과 같다.
- `publishOn`: 체인 중간에서 실행 스레드를 바꾸며, 뒤(하위 신호 방향)의 연산자에만 영향을 준다. `onNext`는 순차 처리된다.
- `subscribeOn`: 구독 시점에 전체 체인의 시작 스레드를 바꾸며, 소스 직후 배치가 일반적이다. 여러 개 배치하면 가장 가까운 것만 유효하다.

---

[참고 자료]
- Spring WebFlux: https://docs.spring.io/spring-framework/reference/web/webflux.html
- Reactor Core Threading and Schedulers: https://projectreactor.io/docs/core/release/reference/coreFeatures/schedulers.html
- Reactor Core Getting Started: https://projectreactor.io/docs/core/release/reference/gettingStarted.html

---

[개념 예시]
아래 코드는 본문을 설명하기 위한 도식용 예시이며 실제 실행 대상이 아니다.
```java
Flux<Integer> flux = Flux.range(1, 3);

flux
    .subscribeOn(Schedulers.boundedElastic())  // 구독 시점 스레드 전환
    .publishOn(Schedulers.parallel())          // 하위 연산자 스레드 전환
    .map(i -> i * 2)                            // schedulerB 위에서 실행
    .subscribeOn(Schedulers.boundedElastic()); // 효과 없음(가장 가까운 것만 유효)
```
위 코드에서 `subscribeOn`을 두 번 넣었지만 실질적으로는 가장 가까운 `Schedulers.boundedElastic()` 하나만 동작한다.
- Spring WebFlux: https://docs.spring.io/spring-framework/reference/web/webflux.html
- Reactor Core Threading and Schedulers: https://projectreactor.io/docs/core/release/reference/coreFeatures/schedulers.html
- Reactor Core Getting Started: https://projectreactor.io/docs/core/release/reference/gettingStarted.html
