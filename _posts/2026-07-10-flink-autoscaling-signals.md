---
title: "Flink 자동 스케일링은 CPU 사용률만 보면 왜 부족할까"
date: 2026-07-10 09:10:00 +0900
categories: [data-engineering]
tags: [apache-flink, autoscaling, streaming, kubernetes, backpressure]
---

실시간 스트리밍 잡의 자동 스케일링은 일반적인 웹 서버 스케일링보다 어렵다. 웹 서버는 요청 수, CPU 사용률, 응답 시간 같은 지표를 기준으로 인스턴스 수를 늘리는 방식이 비교적 직관적이다. 하지만 Apache Flink 같은 stateful 스트림 처리 시스템에서는 단순히 CPU 사용률만 보고 parallelism을 늘리면 충분하지 않다. 병목은 특정 operator에만 있을 수 있고, 상태 이동과 재처리 비용 때문에 스케일링 자체가 지연과 backpressure를 만들 수 있기 때문이다.

Flink Kubernetes Operator Autoscaler 문서는 이 문제를 job 전체 parallelism이 아니라 **job vertex 단위**로 바라본다. job vertex는 chained operator group에 가까운 실행 단위다. 복잡한 파이프라인에서는 source, map/process, window, sink의 부하 특성이 모두 다르다. 어떤 단계는 Kafka backlog 때문에 밀리고, 어떤 단계는 state access 때문에 바쁘고, 어떤 단계는 외부 sink 속도 때문에 backpressure를 만든다. 따라서 전체 pod 수나 CPU 평균만 보면 실제 병목 위치를 놓치기 쉽다.

Autoscaler가 보는 핵심 지표도 컨테이너 CPU/메모리 자체가 아니다. 공식 문서에 따르면 source backlog, source input rate, 각 job vertex의 processing rate, busy time, backpressured time 같은 Flink 내부 지표를 사용한다. source에서 들어오는 데이터율을 기준으로 downstream operator가 처리해야 할 target data rate를 계산하고, 사용자가 정한 target utilization에 맞춰 각 vertex의 parallelism을 조정하는 방식이다. 즉 "클러스터가 바쁜가"가 아니라 "각 operator가 현재 입력률을 안정적으로 따라잡을 수 있는가"를 묻는다.

여기서 중요한 개념은 여유율이다. target utilization을 100%에 가깝게 잡으면 자원을 아낄 수 있어 보이지만, 실제 운영에서는 작은 변동에도 지연이 급격히 커질 수 있다. 문서에서는 target utilization과 함께 min/max utilization 경계를 두어, 일정 범위 안에서는 스케일링을 하지 않고 안정성을 유지하도록 설명한다. 이는 자동 스케일링이 너무 자주 발생해 오히려 불안정해지는 상황을 막기 위한 장치다.

stateful 잡에서는 max parallelism도 미리 설계해야 한다. Flink의 keyed state는 key group 단위로 나뉘고, parallelism 변경 시 이 상태 배치가 영향을 받는다. Autoscaler 문서는 max parallelism을 divisor가 많은 값으로 선택하라고 권장한다. 예를 들어 120, 180, 240, 360, 720 같은 값은 여러 parallelism으로 나누기 쉬워 향후 스케일링 선택지가 넓어진다. 반대로 기본값이나 애매한 값을 쓰면 나중에 조정 가능한 parallelism이 제한될 수 있다.

최근 연구들도 같은 방향을 가리킨다. stateful 스트림 처리의 동적 스케일링에서는 단순히 새 task를 늘리는 것보다, 어떤 상태를 언제 어떻게 옮길지, migration 중 record scheduling을 어떻게 할지, scaling 중 지연을 어떻게 줄일지가 핵심이다. DRRS 같은 연구는 state migration을 더 작은 단위로 나누고 re-routing과 scheduling을 결합해 scaling 중 latency spike를 줄이는 방향을 제안한다. 이는 운영 관점에서도 중요한 시사점이다. autoscaling은 리소스 최적화 기능이 아니라, 상태 이동과 지연 전파를 함께 다루는 제어 루프다.

실무적으로는 세 가지를 먼저 점검해야 한다. 첫째, source connector가 backlog와 busy time 같은 표준 지표를 제대로 노출하는가. 둘째, metrics window와 stabilization interval이 workload 변동성에 맞게 설정되어 있는가. 셋째, scale-out 이후 backlog를 몇 분 안에 따라잡을지, 재시작 또는 in-place rescaling 시간이 어느 정도인지 SLO 기준으로 정의되어 있는가. 이 값들이 없으면 autoscaler는 동작하더라도 운영자가 기대한 방식으로 움직이지 않을 수 있다.

Flink 자동 스케일링의 핵심은 "CPU가 높으면 늘린다"가 아니다. 입력률, 처리율, backlog, busy time, backpressure, max parallelism, 상태 이동 비용을 함께 보고, 지연과 비용 사이의 균형점을 찾는 일이다. 스트리밍 파이프라인이 길어질수록 병목은 평균값 속에 숨어든다. 좋은 autoscaling 설정은 평균 CPU를 낮추는 설정이 아니라, 어느 operator가 어느 입력률에서 병목이 되는지 드러내고, 그 병목을 안전하게 완화하는 설정에 가깝다.

## 참고 링크

- [Apache Flink Kubernetes Operator Autoscaler](https://nightlies.apache.org/flink/flink-kubernetes-operator-docs-main/docs/custom-resource/autoscaler/)
- [Towards Fine-Grained Scalability for Stateful Stream Processing Systems](https://arxiv.org/abs/2503.11320)
- [Justin: Hybrid CPU/Memory Elastic Scaling for Distributed Stream Processing](https://arxiv.org/abs/2505.19739)
