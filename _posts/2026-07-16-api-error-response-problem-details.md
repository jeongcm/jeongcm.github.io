---
title: "API 오류 응답은 왜 Problem Details로 표준화할까"
date: 2026-07-16 21:32:58 +0900
categories: [technical-knowledge, backend-knowledge]
tags: [http-api, error-response, problem-details, spring, rfc9457]
---

API 오류 응답은 단순히 상태 코드와 메시지를 반환하는 문제가 아니다. 클라이언트가 오류를 분류하고, 사용자에게 보여줄 문구를 고르고, 재시도 여부를 판단하며, 운영자가 동일한 장애를 추적할 수 있어야 한다. RFC 9457의 Problem Details는 이런 목적을 위해 HTTP API 오류 본문에 공통 구조를 제공하는 표준이다. 핵심은 `type`, `title`, `status`, `detail`, `instance` 같은 필드를 통해 오류의 종류와 발생 지점을 기계가 읽을 수 있게 만드는 것이다.

HTTP status code는 실패의 큰 범주를 표현한다. 400은 요청이 잘못됐고, 404는 리소스를 찾지 못했으며, 409는 현재 상태와 충돌한다는 식이다. 하지만 같은 409라도 중복 요청, 버전 충돌, 이미 처리된 명령은 대응 방식이 다르다. Problem Details는 status code를 대체하지 않고 보완한다. status code는 HTTP 계층의 의미를 유지하고, `type`은 애플리케이션 오류 종류를 안정적인 식별자로 둔다.

<figure class="post-diagram">
  <svg viewBox="0 0 820 300" role="img" aria-labelledby="problem-details-title problem-details-desc" xmlns="http://www.w3.org/2000/svg">
    <title id="problem-details-title">Problem Details error response flow</title>
    <desc id="problem-details-desc">An HTTP API maps domain exceptions to status codes, problem type URIs, safe details, and traceable instance identifiers.</desc>
    <defs>
      <marker id="arrow-problem" markerWidth="11" markerHeight="11" refX="9" refY="5.5" orient="auto">
        <path d="M1,1 L10,5.5 L1,10 Z" fill="#365f73" />
      </marker>
      <style>
        .node { fill: #f7f4e8; stroke: #365f73; stroke-width: 2; rx: 8; }
        .policy { fill: #e7f0f5; stroke: #365f73; stroke-width: 2; rx: 8; }
        .body { fill: #edf7ec; stroke: #4f7d4c; stroke-width: 2; rx: 8; }
        .label { font: 700 16px -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; fill: #23323a; }
        .small { font: 13px -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; fill: #57666c; }
        .line { stroke: #365f73; stroke-width: 3; fill: none; marker-end: url(#arrow-problem); }
      </style>
    </defs>
    <rect x="28" y="88" width="138" height="82" class="node" />
    <text x="97" y="121" text-anchor="middle" class="label">Exception</text>
    <text x="97" y="146" text-anchor="middle" class="small">domain failure</text>

    <rect x="214" y="88" width="144" height="82" class="policy" />
    <text x="286" y="121" text-anchor="middle" class="label">Mapping</text>
    <text x="286" y="146" text-anchor="middle" class="small">status + type</text>

    <rect x="406" y="42" width="150" height="78" class="body" />
    <text x="481" y="74" text-anchor="middle" class="label">Stable fields</text>
    <text x="481" y="98" text-anchor="middle" class="small">type, title, status</text>

    <rect x="406" y="156" width="150" height="78" class="body" />
    <text x="481" y="188" text-anchor="middle" class="label">Safe details</text>
    <text x="481" y="212" text-anchor="middle" class="small">detail, instance</text>

    <rect x="604" y="88" width="178" height="82" class="node" />
    <text x="693" y="121" text-anchor="middle" class="label">Problem JSON</text>
    <text x="693" y="146" text-anchor="middle" class="small">application/problem+json</text>

    <path d="M166 129 H214" class="line" />
    <path d="M358 116 C382 104 390 88 406 82" class="line" />
    <path d="M358 142 C382 160 390 184 406 195" class="line" />
    <path d="M556 82 C588 88 594 112 604 121" class="line" />
    <path d="M556 195 C588 188 594 152 604 139" class="line" />
  </svg>
  <figcaption>오류 응답 표준화는 예외를 숨기는 작업이 아니라, 클라이언트와 운영자가 쓸 수 있는 안전한 계약으로 바꾸는 작업이다.</figcaption>
</figure>

Spring Framework는 `ProblemDetail`, `ErrorResponse`, `ErrorResponseException`, `ResponseEntityExceptionHandler`를 통해 RFC 9457 형식의 오류 응답을 지원한다. `ProblemDetail`은 표준 필드와 확장 필드를 담는 컨테이너이고, `ErrorResponse`는 HTTP status, header, RFC 9457 형식의 body를 함께 노출하는 계약이다. WebFlux와 MVC 모두 이 모델을 제공하므로, 프레임워크별 오류 처리 방식은 달라도 클라이언트에게 보이는 오류 계약은 통일할 수 있다.

```kotlin
// 개념 예시: 도메인 예외를 ProblemDetail 응답으로 매핑
@RestControllerAdvice
class ApiErrorAdvice {
    @ExceptionHandler(VersionConflictException::class)
    fun handleConflict(error: VersionConflictException): ProblemDetail {
        val problem = ProblemDetail.forStatus(HttpStatus.CONFLICT)
        problem.type = URI.create("https://api.example.com/problems/version-conflict")
        problem.title = "Resource version conflict"
        problem.detail = "요청한 리소스 버전이 현재 서버 상태와 다릅니다."
        problem.setProperty("code", "VERSION_CONFLICT")
        return problem
    }
}
```

설계할 때 가장 중요한 필드는 `type`이다. `title`이나 `detail`은 언어, 상황, 제품 문구에 따라 바뀔 수 있지만, `type`은 클라이언트 분기와 문서 연결에 쓰이는 안정적인 식별자여야 한다. URI를 실제 문서 URL로 열 수 있게 만들면 좋지만, 최소한 충돌 없는 이름 공간이어야 한다. 반대로 `detail`에는 내부 클래스명, SQL, 내부 접속 주소, 인증 비밀값, stack trace를 넣지 않는다. 오류 응답은 디버깅 로그가 아니라 외부 계약이다.

확장 필드는 조심해서 추가한다. RFC 9457은 문제 유형별 확장을 허용하지만, 모든 오류에 임의 필드를 붙이면 클라이언트가 다시 사설 포맷에 묶인다. 검증 오류처럼 필드 단위 정보가 필요한 경우에는 `errors` 배열을 둘 수 있다. 결제 잔액 부족처럼 후속 행동이 필요한 경우에는 관련 링크나 부족한 조건을 넣을 수 있다. 다만 확장 필드도 개인정보와 내부 구현 세부사항을 노출하지 않는 선에서 문서화해야 한다.

운영 관점에서는 `instance`와 trace id의 경계를 정해야 한다. `instance`는 특정 오류 발생을 가리키는 식별자로 쓸 수 있지만, 반드시 내부 로그 키를 그대로 노출할 필요는 없다. 외부 응답에는 지원 요청에 사용할 안전한 식별자를 넣고, 내부 로그에는 같은 값을 trace id나 correlation id와 연결해두는 방식이 현실적이다. 이렇게 해야 클라이언트는 표준 포맷으로 오류를 처리하고, 운영자는 같은 오류를 빠르게 추적할 수 있다.

정리하면 좋은 API 오류 응답은 예쁜 JSON이 아니라 안정적인 실패 계약이다. status code는 HTTP 의미를 지키고, `type`은 애플리케이션 오류 종류를 고정하며, `detail`과 확장 필드는 안전하게 제한한다. Spring의 Problem Details 지원을 사용하면 이 계약을 예외 처리 코드에 흩뿌리지 않고 중앙에서 관리할 수 있다.

## 참고 링크

- [RFC 9457: Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457.html)
- [Spring Framework Reference: WebFlux Error Responses](https://docs.spring.io/spring-framework/reference/web/webflux/ann-rest-exceptions.html)
- [Spring Framework Reference: MVC Error Responses](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-ann-rest-exceptions.html)
- [Spring Framework Reference: REST Clients](https://docs.spring.io/spring-framework/reference/integration/rest-clients.html)
