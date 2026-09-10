---
title: "Apache Flink와 Google Cloud Dataflow 비교: Beam까지 알아야 선택이 정확해진다"
date: 2026-09-10 15:18:00 +0900
categories: [technical-knowledge, data-engineering]
tags: [apache-flink, google-cloud-dataflow, apache-beam, stream-processing, data-platform]
---

Apache Flink와 Google Cloud Dataflow는 모두 대규모 배치·스트리밍 데이터 처리에 사용되지만, 정확히 같은 종류의 기술은 아니다. Flink는 상태 기반 분산 처리 엔진이자 프로그래밍 API를 제공하는 오픈소스 프레임워크다. Dataflow는 Apache Beam 파이프라인을 Google Cloud에서 실행하는 관리형 데이터 처리 서비스다. 따라서 두 기술을 비교하려면 중간에 있는 Beam의 역할부터 분리해야 한다.

## 먼저 세 기술의 역할을 구분해야 한다

Apache Beam은 파이프라인을 표현하는 프로그래밍 모델과 SDK다. 개발자는 `Pipeline`, `PCollection`, `PTransform`, window, watermark, trigger, state, timer 같은 추상화로 계산을 정의한다. 실제 실행은 Runner가 담당한다. 같은 Beam 파이프라인도 `FlinkRunner`를 선택하면 Flink job으로 변환되고, `DataflowRunner`를 선택하면 Google Cloud Dataflow job으로 제출된다.

Flink는 이보다 아래의 실행 엔진 역할까지 직접 담당한다. Native Flink DataStream API나 Table API로 작성한 프로그램을 JobManager와 TaskManager가 실행하고, 네트워크 셔플, 상태 저장, checkpoint, 장애 복구, parallelism을 관리한다. Kubernetes, YARN, standalone 환경 등에 배포할 수 있으며, 배포 방식에 따라 플랫폼 운영 책임은 사용자 조직이나 관리형 Flink 제공자에게 남는다.

Dataflow는 Beam 파이프라인을 받아 worker 생성과 종료, 작업 분산, 셔플, 모니터링, 자동 확장 같은 실행 환경을 Google Cloud 서비스 경계 안에서 관리한다. 즉 비교 대상은 단순히 `Flink 대 Beam`이 아니라 다음 세 가지 선택지다.

<figure class="post-diagram">
  <svg viewBox="0 0 960 520" role="img" aria-labelledby="flink-dataflow-title flink-dataflow-desc" xmlns="http://www.w3.org/2000/svg">
    <title id="flink-dataflow-title">Native Flink, Beam on Flink, Beam on Dataflow 계층 비교</title>
    <desc id="flink-dataflow-desc">파이프라인 API, Runner, 실행 엔진, 인프라 관리 책임을 세 가지 실행 경로로 나눈 그림</desc>
    <rect x="20" y="20" width="920" height="480" rx="8" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="480" y="54" text-anchor="middle" font-size="20" font-family="sans-serif" fill="#17202a">같은 데이터 처리 요구를 구현하는 세 가지 경로</text>

    <rect x="48" y="82" width="264" height="376" rx="8" fill="#eef6ff" stroke="#2878b5" stroke-width="2"/>
    <text x="180" y="116" text-anchor="middle" font-size="17" font-family="sans-serif" font-weight="bold" fill="#174f78">Native Flink</text>
    <rect x="76" y="142" width="208" height="58" rx="6" fill="#ffffff" stroke="#5c9dcc"/>
    <text x="180" y="166" text-anchor="middle" font-size="13" font-family="sans-serif" fill="#17202a">DataStream / Table API</text>
    <text x="180" y="185" text-anchor="middle" font-size="12" font-family="sans-serif" fill="#52606d">Flink 전용 기능 직접 사용</text>
    <path d="M180 200 V228" stroke="#64748b" stroke-width="2"/>
    <polygon points="180,236 175,226 185,226" fill="#64748b"/>
    <rect x="76" y="238" width="208" height="58" rx="6" fill="#dff0ff" stroke="#2878b5"/>
    <text x="180" y="273" text-anchor="middle" font-size="14" font-family="sans-serif" fill="#17202a">Apache Flink Runtime</text>
    <path d="M180 296 V324" stroke="#64748b" stroke-width="2"/>
    <polygon points="180,332 175,322 185,322" fill="#64748b"/>
    <rect x="76" y="334" width="208" height="86" rx="6" fill="#ffffff" stroke="#5c9dcc"/>
    <text x="180" y="365" text-anchor="middle" font-size="13" font-family="sans-serif" fill="#17202a">Kubernetes / YARN / Standalone</text>
    <text x="180" y="390" text-anchor="middle" font-size="12" font-family="sans-serif" fill="#52606d">운영 경계 직접 설계</text>

    <rect x="348" y="82" width="264" height="376" rx="8" fill="#eef9f1" stroke="#438a58" stroke-width="2"/>
    <text x="480" y="116" text-anchor="middle" font-size="17" font-family="sans-serif" font-weight="bold" fill="#2f6740">Beam on Flink</text>
    <rect x="376" y="142" width="208" height="58" rx="6" fill="#ffffff" stroke="#6fa67e"/>
    <text x="480" y="166" text-anchor="middle" font-size="13" font-family="sans-serif" fill="#17202a">Apache Beam SDK</text>
    <text x="480" y="185" text-anchor="middle" font-size="12" font-family="sans-serif" fill="#52606d">이식 가능한 파이프라인 모델</text>
    <path d="M480 200 V218" stroke="#64748b" stroke-width="2"/>
    <polygon points="480,226 475,216 485,216" fill="#64748b"/>
    <rect x="376" y="228" width="208" height="48" rx="6" fill="#dff3e5" stroke="#438a58"/>
    <text x="480" y="258" text-anchor="middle" font-size="14" font-family="sans-serif" fill="#17202a">Flink Runner</text>
    <path d="M480 276 V294" stroke="#64748b" stroke-width="2"/>
    <polygon points="480,302 475,292 485,292" fill="#64748b"/>
    <rect x="376" y="304" width="208" height="48" rx="6" fill="#dff3e5" stroke="#438a58"/>
    <text x="480" y="334" text-anchor="middle" font-size="14" font-family="sans-serif" fill="#17202a">Apache Flink Runtime</text>
    <path d="M480 352 V370" stroke="#64748b" stroke-width="2"/>
    <polygon points="480,378 475,368 485,368" fill="#64748b"/>
    <rect x="376" y="380" width="208" height="40" rx="6" fill="#ffffff" stroke="#6fa67e"/>
    <text x="480" y="405" text-anchor="middle" font-size="12" font-family="sans-serif" fill="#17202a">Flink 인프라 운영 필요</text>

    <rect x="648" y="82" width="264" height="376" rx="8" fill="#fff6e8" stroke="#c47b16" stroke-width="2"/>
    <text x="780" y="116" text-anchor="middle" font-size="17" font-family="sans-serif" font-weight="bold" fill="#8a5511">Beam on Dataflow</text>
    <rect x="676" y="142" width="208" height="58" rx="6" fill="#ffffff" stroke="#d6a050"/>
    <text x="780" y="166" text-anchor="middle" font-size="13" font-family="sans-serif" fill="#17202a">Apache Beam SDK</text>
    <text x="780" y="185" text-anchor="middle" font-size="12" font-family="sans-serif" fill="#52606d">이식 가능한 파이프라인 모델</text>
    <path d="M780 200 V228" stroke="#64748b" stroke-width="2"/>
    <polygon points="780,236 775,226 785,226" fill="#64748b"/>
    <rect x="676" y="238" width="208" height="58" rx="6" fill="#ffebcc" stroke="#c47b16"/>
    <text x="780" y="273" text-anchor="middle" font-size="14" font-family="sans-serif" fill="#17202a">Dataflow Runner</text>
    <path d="M780 296 V324" stroke="#64748b" stroke-width="2"/>
    <polygon points="780,332 775,322 785,322" fill="#64748b"/>
    <rect x="676" y="334" width="208" height="86" rx="6" fill="#ffffff" stroke="#d6a050"/>
    <text x="780" y="365" text-anchor="middle" font-size="13" font-family="sans-serif" fill="#17202a">Google Cloud Dataflow</text>
    <text x="780" y="390" text-anchor="middle" font-size="12" font-family="sans-serif" fill="#52606d">관리형 worker·shuffle·scaling</text>

    <text x="180" y="482" text-anchor="middle" font-size="12" font-family="sans-serif" fill="#52606d">제어 범위 큼</text>
    <text x="480" y="482" text-anchor="middle" font-size="12" font-family="sans-serif" fill="#52606d">Beam 추상화 + Flink 운영</text>
    <text x="780" y="482" text-anchor="middle" font-size="12" font-family="sans-serif" fill="#52606d">관리 책임 위임 큼</text>
  </svg>
  <figcaption>직접 작성한 다이어그램: API, Runner, runtime, 인프라의 책임 경계</figcaption>
</figure>

## 프로그래밍 모델: 엔진의 기능인가, Runner 간 이식성인가

Native Flink API는 Flink runtime과 가장 가까운 선택이다. keyed state, broadcast state, process function, event-time timer, async I/O 등 Flink의 기능과 튜닝 지점을 직접 사용할 수 있다. 복잡한 상태 기반 스트림 처리, 세밀한 operator 제어, 낮은 지연시간 최적화가 핵심이라면 이 직접성이 장점이다. 반면 코드가 Flink API와 실행 의미에 더 강하게 결합된다.

Beam은 실행 엔진보다 파이프라인의 논리 모델을 앞세운다. bounded와 unbounded data를 `PCollection`으로 표현하고, window와 trigger로 언제 결과를 낼지 정의한다. Runner가 논리 그래프를 대상 엔진의 실행 그래프로 번역한다. 하나의 프로그래밍 모델로 batch와 streaming을 다루고 Runner 선택지를 남길 수 있지만, 모든 Runner가 모든 Beam 기능을 같은 수준으로 지원하는 것은 아니다. 실제 이전 가능성은 Beam capability matrix, 사용 중인 connector, 언어 SDK, custom transform, 서비스 전용 pipeline option까지 확인해야 한다.

따라서 `Beam으로 작성했으니 언제든 Runner만 바꾸면 된다`는 표현은 목표에 가깝지 보장에 가깝지는 않다. portability를 중요하게 본다면 개발 초기부터 복수 Runner 테스트, 지원 기능의 교집합, 외부 시스템에 대한 부수 효과를 함께 관리해야 한다.

## 상태와 시간: 공통점은 많지만 제어 위치가 다르다

스트리밍 시스템은 처리 중간 결과를 state로 보존하고, event time과 watermark로 늦게 도착한 데이터를 판단해야 한다. Flink와 Beam/Dataflow 모두 이 문제를 다루지만 추상화와 운영 제어의 위치가 다르다.

Flink의 keyed state는 같은 key의 event가 같은 logical state에 접근하도록 partition된다. checkpoint는 operator state와 처리 위치를 일관된 snapshot으로 남기고, 장애 시 source가 재생 가능한 지점과 state를 함께 복구한다. savepoint는 계획된 버전 교체나 마이그레이션의 기준점으로 활용할 수 있다. checkpoint 주기, timeout, state backend, checkpoint storage, restart strategy, sink 보장 수준은 운영자가 명시적으로 설계해야 한다.

Beam에서는 window, watermark, trigger, state, timer로 처리 의도를 표현하고 Runner가 이를 구현한다. Dataflow streaming job의 기본 처리 모드는 exactly-once지만, 이 의미를 외부 시스템의 부수 효과까지 자동으로 한 번만 실행된다는 뜻으로 넓히면 안 된다. Google Cloud 공식 문서도 user code가 외부 시스템에 보내는 요청은 worker retry 때문에 두 번 이상 실행될 수 있다고 설명한다. HTTP 호출, 외부 DB 갱신, 알림 발송 같은 동작에는 idempotency key, upsert, deduplication 같은 별도 대책이 필요하다.

Flink도 마찬가지로 checkpoint만 켠다고 모든 sink가 exactly-once가 되지는 않는다. source의 replay 가능성, sink connector의 transaction 또는 idempotency 지원, checkpoint 완료와 외부 commit의 연계가 함께 맞아야 한다. 두 제품 모두 `정확히 한 번`을 제품 이름의 특성보다 데이터 소스부터 외부 sink까지 이어지는 end-to-end 계약으로 봐야 한다.

## 자동 확장과 장애 복구: 기본 제공 범위가 다르다

Dataflow는 worker CPU 사용률과 pipeline parallelism 같은 신호를 바탕으로 worker 수를 조정하는 Horizontal Autoscaling을 제공한다. batch pipeline에는 기본 활성화되고, Streaming Engine을 사용하는 streaming job에서도 기본 활성화된다. 최대·최소 worker 수를 제한할 수 있고, Dataflow 서비스가 worker 생성과 제거를 담당한다. 실행 그래프 최적화, work rebalancing, 모니터링 UI도 관리형 서비스 경계 안에 있다.

Flink도 Adaptive Scheduler, Reactive Mode, Kubernetes Operator autoscaler 같은 확장 수단을 제공한다. 다만 `필요한 TaskManager를 누가 늘리고 줄이는가`, rescale 때 어떤 checkpoint에서 복구하는가, state 크기에 따른 재시작 시간과 재처리량을 어떻게 제한하는가는 배포 구조와 운영 설정에 달려 있다. 특히 rescale은 실행 중인 state를 재배치하는 작업이므로 단순히 Pod 수만 늘리는 것과 같지 않다. checkpoint 건강도, max parallelism, key 분포, state 크기, restart 시간까지 함께 관찰해야 한다.

이 차이는 기능 유무보다 기본 책임 경계의 차이다. Dataflow는 플랫폼이 확장 정책과 worker lifecycle의 상당 부분을 맡는다. Flink는 세부 정책과 런타임에 대한 제어 폭이 넓은 대신, 오픈소스 Flink를 직접 운영하면 Kubernetes, checkpoint storage, HA, 배포, 버전 호환성, 관측 체계를 데이터 플랫폼 팀이 구성해야 한다. 관리형 Flink 서비스를 선택하면 이 운영 부담의 일부를 공급자에게 넘길 수 있으므로 `Flink는 항상 self-managed`라고 단정해서는 안 된다.

## 비용은 worker 단가보다 운영 경계로 비교해야 한다

Dataflow 비용에는 job이 사용한 worker vCPU와 memory, Dataflow Shuffle 또는 Streaming Engine, persistent disk와 snapshot 등이 포함될 수 있다. 여기에 Pub/Sub, BigQuery, Cloud Storage, Logging, cross-region network 같은 인접 Google Cloud 서비스 비용도 더해진다. 자동 확장은 유휴 자원을 줄일 수 있지만, max worker와 backlog가 통제되지 않으면 비용도 함께 빠르게 증가할 수 있다.

Flink를 직접 운영할 때는 compute, disk, object storage, network, Kubernetes 같은 인프라 비용이 보인다. 여기에 업그레이드, on-call, checkpoint 장애 대응, state migration, autoscaling 정책, 보안 패치에 드는 엔지니어링 비용을 포함해야 공정하다. 이미 성숙한 Kubernetes와 데이터 플랫폼 운영 역량이 있다면 자원 밀도와 세밀한 튜닝으로 비용을 통제할 여지가 크다. 반대로 작은 팀이 소수의 pipeline을 빠르게 운영해야 한다면 관리형 Dataflow가 총소유비용을 낮출 수 있다.

## 선택 기준을 표로 정리하면

| 판단 기준 | Native Flink | Beam on Flink | Beam on Dataflow |
|---|---|---|---|
| 핵심 가치 | Flink 기능과 runtime 제어 | Beam 모델의 이식성과 Flink 실행 | Google Cloud 관리형 운영 |
| API 결합 | Flink DataStream/Table API | Apache Beam SDK | Apache Beam SDK |
| 인프라 책임 | 직접 운영 또는 관리형 Flink 선택 | Flink cluster 운영 필요 | Google Cloud가 worker lifecycle 관리 |
| 상태 처리 제어 | state backend, checkpoint, timer를 세밀하게 제어 | Beam 추상화와 Runner 지원 범위 안에서 제어 | Beam 의미를 Dataflow가 관리형으로 실행 |
| 자동 확장 | Operator·scheduler·인프라 정책 조합 | Flink 측 확장 체계 필요 | Horizontal Autoscaling 등 서비스 기능 사용 |
| 생태계 결합 | Kafka, Iceberg, Kubernetes 등 조합 자유도 큼 | Beam connector와 Flink 생태계의 교집합 | Pub/Sub, BigQuery, Cloud Storage와 결합이 자연스러움 |
| 이식성 | Flink runtime 중심 | Runner 교체 가능성을 확보 | Beam 수준의 이식성, GCP 전용 옵션은 별도 관리 |
| 적합한 상황 | 복잡한 stateful stream, 낮은 지연, 세부 튜닝 | Beam 표준화와 자체 Flink 기반을 함께 유지 | 운영 인력이 제한적이고 GCP 중심으로 빠르게 배포 |

## 어떤 경우에 무엇을 선택할까

다음 조건이 중요하면 Native Flink가 유리하다.

- 복잡한 keyed state와 timer를 사용하는 장기 실행 streaming application
- operator별 parallelism, state backend, checkpoint와 복구 시간을 세밀하게 제어해야 하는 경우
- on-premise, multi-cloud, Kubernetes 기반처럼 실행 환경 선택권이 중요한 경우
- 데이터 플랫폼 팀이 Flink cluster와 stateful workload를 운영할 역량을 갖춘 경우

다음 조건이라면 Beam on Flink를 검토할 수 있다.

- 조직의 pipeline API를 Beam으로 표준화하면서 실행 환경은 자체 Flink를 유지하려는 경우
- Dataflow를 포함한 다른 Runner로 이동할 선택지를 코드 구조에 남기려는 경우
- 사용할 transform과 connector가 Flink Runner에서 충분히 지원되는지 검증할 수 있는 경우

다음 조건이 중요하면 Beam on Dataflow가 유리하다.

- GCP가 주 실행 환경이고 Pub/Sub, BigQuery, Cloud Storage 연계가 중심인 경우
- cluster provisioning, worker lifecycle, scaling, distributed shuffle 운영을 서비스에 위임하려는 경우
- traffic 변화가 크고 빠른 배포와 관리형 모니터링이 세부 runtime 제어보다 중요한 경우
- 서비스별 과금과 GCP 종속성을 받아들일 수 있는 경우

## 결론: 엔진 비교보다 책임 경계 비교가 먼저다

Apache Beam과 Flink는 완전한 경쟁 관계가 아니다. Beam은 Flink 위에서 실행될 수 있고, Dataflow 역시 Beam pipeline을 실행하는 Runner다. 그래서 기술 선택은 `Beam이냐 Flink냐` 한 줄로 끝나지 않는다. 파이프라인 코드를 어느 API에 결합할지, 실행 엔진을 누가 운영할지, 상태와 장애 복구를 어느 수준까지 제어할지, cloud service에 어떤 책임을 위임할지를 순서대로 결정해야 한다.

복잡한 stateful stream 처리와 runtime 제어가 우선이면 Native Flink가 강하다. Beam의 공통 모델을 유지하면서 자체 실행 환경을 쓰려면 Beam on Flink가 중간 선택지가 된다. GCP 중심의 운영 단순화와 빠른 확장이 우선이면 Beam on Dataflow가 자연스럽다. 최종 판단에서는 기능 목록보다 운영 인력, recovery 목표, connector 지원, 예상 traffic, 외부 sink의 정확성 계약을 실제 workload로 검증하는 것이 중요하다.

## 참고 링크

- [Apache Beam: Basics of the Beam model](https://beam.apache.org/documentation/basics/)
- [Apache Beam: Flink Runner](https://beam.apache.org/documentation/runners/flink/)
- [Apache Beam: Dataflow Runner](https://beam.apache.org/documentation/runners/dataflow/)
- [Apache Beam: Runner capability matrix](https://beam.apache.org/documentation/runners/capability-matrix/)
- [Apache Flink: Stateful Stream Processing](https://nightlies.apache.org/flink/flink-docs-release-2.1/docs/concepts/stateful-stream-processing/)
- [Apache Flink: Checkpoints](https://nightlies.apache.org/flink/flink-docs-stable/docs/ops/state/checkpoints/)
- [Apache Flink: Deployment overview](https://nightlies.apache.org/flink/flink-docs-stable/docs/deployment/overview/)
- [Apache Flink: Elastic Scaling](https://nightlies.apache.org/flink/flink-docs-stable/docs/deployment/elastic_scaling/)
- [Google Cloud Dataflow: Deploy pipelines](https://cloud.google.com/dataflow/docs/guides/deploying-a-pipeline)
- [Google Cloud Dataflow: Horizontal Autoscaling](https://cloud.google.com/dataflow/docs/horizontal-autoscaling)
- [Google Cloud Dataflow: Streaming modes](https://cloud.google.com/dataflow/docs/guides/streaming-modes)
- [Google Cloud Dataflow: Exactly-once processing](https://cloud.google.com/dataflow/docs/concepts/exactly-once)
- [Google Cloud Dataflow pricing](https://cloud.google.com/dataflow/pricing)
