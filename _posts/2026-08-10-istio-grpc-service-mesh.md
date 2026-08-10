---
title: "ALB에서 Istio Ingress Gateway로 들어온 gRPC 요청은 어떻게 Mesh 안에서 흘러갈까"
date: 2026-08-10 09:56:00 +0900
categories: [technical-knowledge, backend-knowledge]
tags: [istio, grpc, service-mesh, envoy, alb]
---

gRPC를 Kubernetes에서 운영할 때 흔한 진입 구조는 `AWS Application Load Balancer -> Istio Ingress Gateway -> 내부 gRPC service`다. 이때 중요한 점은 Istio가 gRPC를 별도 전용 프로토콜처럼 처리하는 것이 아니라, HTTP/2 위에서 동작하는 요청으로 인식하고 Envoy 프록시 체인에서 라우팅·부하 분산·회복성 정책을 적용한다는 것이다.

AWS ALB는 gRPC target group을 지원한다. HTTPS listener로 요청을 받고, target group의 protocol version을 `gRPC` 또는 `HTTP/2`로 설정하면 gRPC 요청을 target으로 전달할 수 있다. ALB는 `/package.Service/Method` 형태의 gRPC method를 보고 rule routing과 health check를 수행할 수 있다. 이 단계의 책임은 외부 클라이언트 연결 수용, TLS 종료 또는 전달 방식, target health 기반의 1차 분산이다.

<figure class="post-diagram">
  <svg viewBox="0 0 920 360" role="img" aria-labelledby="istio-grpc-title istio-grpc-desc" xmlns="http://www.w3.org/2000/svg">
    <title id="istio-grpc-title">gRPC request through ALB and Istio service mesh</title>
    <desc id="istio-grpc-desc">A gRPC request enters through ALB, reaches Istio ingress gateway, is routed by VirtualService, then forwarded through Envoy sidecars to a workload.</desc>
    <defs>
      <marker id="arrow-grpc" markerWidth="10" markerHeight="10" refX="9" refY="5" orient="auto">
        <path d="M1,1 L9,5 L1,9 Z" fill="#475569" />
      </marker>
      <style>
        .box { rx: 8; stroke-width: 2; }
        .edge { fill: #eef6ff; stroke: #2563eb; }
        .mesh { fill: #ecfdf5; stroke: #059669; }
        .policy { fill: #fff7ed; stroke: #ea580c; }
        .svc { fill: #f8fafc; stroke: #64748b; }
        .label { font: 700 15px -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; fill: #0f172a; }
        .small { font: 13px -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; fill: #475569; }
        .line { stroke: #475569; stroke-width: 3; fill: none; marker-end: url(#arrow-grpc); }
      </style>
    </defs>
    <rect x="30" y="74" width="150" height="88" class="box edge" />
    <text x="105" y="108" text-anchor="middle" class="label">Client</text>
    <text x="105" y="132" text-anchor="middle" class="small">gRPC over HTTP/2</text>
    <path d="M180 118 H250" class="line" />
    <rect x="250" y="74" width="150" height="88" class="box edge" />
    <text x="325" y="108" text-anchor="middle" class="label">ALB</text>
    <text x="325" y="132" text-anchor="middle" class="small">target health / TLS</text>
    <path d="M400 118 H470" class="line" />
    <rect x="470" y="74" width="178" height="88" class="box mesh" />
    <text x="559" y="108" text-anchor="middle" class="label">Istio Gateway</text>
    <text x="559" y="132" text-anchor="middle" class="small">Envoy edge proxy</text>
    <path d="M648 118 H718" class="line" />
    <rect x="718" y="74" width="172" height="88" class="box svc" />
    <text x="804" y="108" text-anchor="middle" class="label">gRPC service</text>
    <text x="804" y="132" text-anchor="middle" class="small">sidecar + workload</text>
    <rect x="196" y="230" width="528" height="76" class="box policy" />
    <text x="460" y="260" text-anchor="middle" class="label">Mesh policies</text>
    <text x="460" y="284" text-anchor="middle" class="small">VirtualService routing, DestinationRule load balancing, retry, timeout, circuit breaking, outlier detection</text>
  </svg>
  <figcaption>ALB는 외부 진입과 target 상태를, Istio는 mesh 내부 라우팅과 서비스 간 정책을 담당한다.</figcaption>
</figure>

Istio Ingress Gateway에 도착한 뒤에는 Envoy가 요청을 받는다. `Gateway`는 어떤 host와 port/protocol을 열지 정의하고, `VirtualService`는 들어온 요청을 어떤 Kubernetes service로 보낼지 결정한다. gRPC는 HTTP/2 path가 method 이름을 담고 있으므로, 필요하면 URI prefix나 exact path로 method 단위 라우팅도 가능하다. 다만 gateway가 backend로 HTTP/2를 유지하려면 서비스 포트 이름을 `grpc` 또는 `http2`로 두거나 `appProtocol: grpc`처럼 명시하는 것이 안전하다. Istio 문서도 gateway가 명시적 프로토콜 선택 없이 backend로 전달할 때 HTTP/1.1을 사용할 수 있다고 설명한다.

Mesh 내부에서 "service mesh를 한다"는 말은 요청이 목적지 pod로 바로 가는 것이 아니라, Envoy가 목적지 service의 endpoint 목록과 정책을 xDS로 받아 upstream cluster를 구성한다는 뜻이다. 요청이 들어오면 Envoy는 cluster의 endpoint 중 하나를 고른다. Istio 기본 부하 분산은 least requests 계열이며, `DestinationRule`로 round robin, random, consistent hash 같은 정책을 바꿀 수 있다. 선택된 endpoint로는 HTTP/2 connection pool을 통해 gRPC stream이 전달된다.

부하가 발생했을 때 조절 지점은 여러 층으로 나뉜다. ALB는 unhealthy target을 제외하고, target group 단위로 외부 유입을 분산한다. Istio Ingress Gateway는 edge Envoy이므로 gateway pod 자체의 CPU, memory, connection, HTTP/2 stream 한계가 병목이 될 수 있다. 내부 service 호출에서는 DestinationRule의 connection pool, HTTP/2 request 한도, outlier detection, retry/timeout이 작동한다. 과부하 상황에서 무조건 retry를 늘리면 gRPC stream이 더 오래 붙잡혀 tail latency와 pending request를 키울 수 있으므로, retry는 idempotent한 unary 요청 위주로 제한하는 편이 안전하다.

```yaml
# 개념 예시: gRPC 서비스에 대한 Istio 라우팅과 부하 보호 설정
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: order-grpc
spec:
  host: order-grpc.default.svc.cluster.local
  trafficPolicy:
    loadBalancer:
      simple: LEAST_REQUEST
    connectionPool:
      http:
        http2MaxRequests: 1000
        maxRequestsPerConnection: 10000
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 10s
      baseEjectionTime: 30s
---
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: order-grpc
spec:
  hosts:
  - api.example.com
  gateways:
  - grpc-gateway
  http:
  - match:
    - uri:
        prefix: /order.v1.OrderService/
    route:
    - destination:
        host: order-grpc.default.svc.cluster.local
        port:
          number: 9090
    timeout: 3s
```

주의할 점은 gRPC의 long-lived stream이다. unary RPC는 HTTP 요청처럼 timeout, retry, circuit breaker를 비교적 단순하게 적용할 수 있다. 반면 server streaming이나 bidirectional streaming은 하나의 HTTP/2 stream이 오래 유지되므로 `maxRequestsPerConnection`, idle timeout, keepalive, retry 정책이 실제 연결 수명과 충돌할 수 있다. 장애 처리도 HTTP status만 보지 말고 gRPC status code, deadline exceeded, unavailable 같은 의미를 함께 봐야 한다.

정리하면 ALB와 Istio의 역할을 섞어 이해하지 않는 것이 중요하다. ALB는 외부 gRPC 트래픽을 mesh edge까지 안정적으로 가져오는 L7 load balancer다. Istio는 ingress gateway와 sidecar Envoy를 통해 HTTP/2/gRPC 요청을 service 단위로 라우팅하고, endpoint 선택, 회로 차단, 이상 endpoint 제거, timeout/retry를 정책으로 적용한다. 부하 제어의 핵심은 한 곳에서 모든 문제를 해결하려는 것이 아니라, ALB target health, gateway capacity, DestinationRule의 per-service 한도, 애플리케이션의 gRPC deadline을 같은 운영 계약으로 맞추는 데 있다.

## 참고 링크

- [AWS ELB Documentation - Target groups for Application Load Balancers](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html)
- [AWS Announcement - Application Load Balancers enables gRPC workloads](https://aws.amazon.com/about-aws/whats-new/2020/10/application-load-balancers-enable-grpc-workloads-end-to-end-http-2-support/)
- [Istio Ingress Gateways](https://istio.io/latest/docs/tasks/traffic-management/ingress/ingress-control/)
- [Istio Protocol Selection](https://istio.io/latest/docs/ops/configuration/traffic-management/protocol-selection/)
- [Istio Traffic Management](https://istio.io/latest/docs/concepts/traffic-management/)
- [Istio DestinationRule Reference](https://istio.io/latest/docs/reference/config/networking/destination-rule/)
- [Envoy Circuit Breaking](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/circuit_breaking.html)
