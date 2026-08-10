---
title: "Flink 2.3의 Checkpointing During Recovery를 운영 관점에서 보기"
date: 2026-08-10 09:13:00 +0900
categories: [data-engineering]
tags: [apache-flink, checkpoint, recovery, unaligned-checkpoint, streaming]
---

Apache Flink 2.3.0의 여러 변경 중 운영 관점에서 눈여겨볼 기능은 `Checkpointing during recovery`다. 이 기능은 unaligned checkpoint에서 복구 중인 job도 새로운 checkpoint를 만들 수 있게 한다. 장애나 rescale 이후 복구가 오래 걸리는 대규모 스트리밍 job에서는 "복구가 끝날 때까지 다음 안전 지점이 없다"는 구간이 길어질 수 있는데, Flink 2.3은 이 구간을 줄이는 선택지를 제공한다.

기존 동작을 먼저 보면 문제의 배경이 분명하다. Unaligned checkpoint는 backpressure 상황에서 checkpoint barrier가 데이터 흐름에 오래 묶이지 않도록 in-flight buffer까지 checkpoint state로 저장한다. 이 방식은 checkpoint 완료 시간을 줄이는 데 유리하지만, 복구 시에는 저장된 channel state를 다시 소비해야 한다. 복구해야 할 in-flight state가 크거나, stateful operator와 join이 무겁거나, rescale이 반복되면 복구 단계가 길어진다. 이 동안 추가 장애가 발생하면 이미 처리한 복구 진행분을 보존하지 못하고 이전 checkpoint에서 다시 시작해야 한다.

<figure class="post-diagram">
  <svg viewBox="0 0 860 330" role="img" aria-labelledby="flink-recovery-title flink-recovery-desc" xmlns="http://www.w3.org/2000/svg">
    <title id="flink-recovery-title">Flink checkpointing during recovery flow</title>
    <desc id="flink-recovery-desc">A Flink job recovering from an unaligned checkpoint can create a new checkpoint before all restored channel state is consumed.</desc>
    <defs>
      <marker id="arrow-recovery" markerWidth="10" markerHeight="10" refX="9" refY="5" orient="auto">
        <path d="M1,1 L9,5 L1,9 Z" fill="#475569" />
      </marker>
      <style>
        .box { rx: 8; stroke-width: 2; }
        .old { fill: #eef2ff; stroke: #4f46e5; }
        .run { fill: #ecfdf5; stroke: #059669; }
        .risk { fill: #fef2f2; stroke: #dc2626; }
        .new { fill: #fff7ed; stroke: #ea580c; }
        .label { font: 700 15px -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; fill: #0f172a; }
        .small { font: 13px -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; fill: #475569; }
        .line { stroke: #475569; stroke-width: 3; fill: none; marker-end: url(#arrow-recovery); }
      </style>
    </defs>
    <rect x="36" y="58" width="168" height="86" class="box old" />
    <text x="120" y="92" text-anchor="middle" class="label">Checkpoint N</text>
    <text x="120" y="116" text-anchor="middle" class="small">state + channel state</text>
    <path d="M204 101 H288" class="line" />
    <rect x="288" y="58" width="196" height="86" class="box run" />
    <text x="386" y="92" text-anchor="middle" class="label">Recovery phase</text>
    <text x="386" y="116" text-anchor="middle" class="small">consume restored buffers</text>
    <path d="M484 101 H568" class="line" />
    <rect x="568" y="58" width="238" height="86" class="box risk" />
    <text x="687" y="92" text-anchor="middle" class="label">Old boundary</text>
    <text x="687" y="116" text-anchor="middle" class="small">wait until recovery completes</text>
    <path d="M386 144 V210 H458" class="line" />
    <rect x="458" y="186" width="214" height="86" class="box new" />
    <text x="565" y="220" text-anchor="middle" class="label">Checkpoint N+1</text>
    <text x="565" y="244" text-anchor="middle" class="small">record recovery progress</text>
    <path d="M672 229 H760" class="line" />
    <rect x="760" y="186" width="64" height="86" class="box run" />
    <text x="792" y="220" text-anchor="middle" class="label">Next</text>
    <text x="792" y="244" text-anchor="middle" class="small">resume</text>
  </svg>
  <figcaption>복구 도중 checkpoint를 만들면 장애·rescale 반복 시 다시 돌아가야 하는 지점을 앞당길 수 있다.</figcaption>
</figure>

Flink 2.3 릴리스 노트는 이 기능을 `FLINK-35761`과 `FLIP-547`로 설명한다. 핵심은 복구 중인 channel state를 안전하게 다루면서도 exactly-once semantics를 유지하는 것이다. FLIP-547은 단순히 기존 unaligned checkpoint 로직을 복구 단계에 그대로 적용하면 rescale 과정에서 buffer가 중복 처리될 수 있다고 설명한다. 따라서 이 기능은 "checkpoint를 더 자주 찍는다"가 아니라, 복구 중인 in-flight data를 어떤 task 경계에서 다시 조직하고 보존할지에 대한 런타임 변경에 가깝다.

설정은 기본적으로 꺼져 있다. Flink 2.3 릴리스 노트 기준으로는 아래 두 옵션을 함께 켜야 한다.

```yaml
# 개념 예시: 대규모 unaligned checkpoint 복구 시간을 줄이기 위한 검토 설정
execution.checkpointing.unaligned.recover-output-on-downstream.enabled: true
execution.checkpointing.unaligned.during-recovery.enabled: true
```

적용 대상은 명확하다. 첫째, backpressure 때문에 unaligned checkpoint를 사용하고 있고 channel state가 크게 쌓이는 job이다. 둘째, Kubernetes, YARN, adaptive scheduler 등에서 rescale이나 restart가 비교적 자주 발생하는 환경이다. 셋째, 복구가 완료되기 전 장애가 다시 나면 동일한 대량 복구를 반복하는 job이다. 이런 경우 복구 중 checkpoint가 "전체 복구 완료"보다 앞선 안전 지점을 만들어 운영 리스크를 줄일 수 있다.

반대로 모든 Flink job에 무조건 켤 기능은 아니다. Aligned checkpoint만으로도 checkpoint duration과 recovery time이 안정적인 job이라면 이 설정이 주는 이득은 작다. 또한 unaligned checkpoint 자체가 in-flight buffer를 state로 저장하므로 checkpoint storage 사용량, network buffer 크기, checkpoint timeout, externalized checkpoint 보존 정책을 함께 봐야 한다. 복구 중 checkpoint가 가능해져도 느린 UDF, 과도한 join state, 느린 checkpoint storage 같은 근본 원인을 대신 해결하지는 않는다.

운영 도입 시에는 세 가지 지표를 먼저 비교하는 편이 좋다. 첫째, 장애 또는 rescale 이후 첫 checkpoint가 완료되기까지 걸리는 시간이다. 둘째, 복구 중 재시작이 발생했을 때 재처리되는 시간과 backlog 증가량이다. 셋째, checkpoint state size와 channel state 비중이다. 이 값들이 크고 변동성이 높다면 Flink 2.3의 변경은 단순 릴리스 기능이 아니라 복구 전략의 일부로 검토할 만하다.

정리하면 `Checkpointing during recovery`는 장애를 없애는 기능이 아니라, 장애 후 복구가 길어질 때 진행 상황을 더 빨리 durable boundary로 남기는 기능이다. 대규모 상태와 backpressure를 가진 스트리밍 파이프라인에서는 checkpoint 완료 시간뿐 아니라 "복구 중 다시 실패했을 때 어디서 재시작하는가"가 운영 안정성을 좌우한다. Flink 2.3의 이 변화는 그 경계를 더 세밀하게 설계할 수 있게 만든다.

## 참고 링크

- [Apache Flink 2.3.0 Release Announcement](https://flink.apache.org/2026/06/25/apache-flink-2.3.0-release-announcement/)
- [Apache Flink 2.3 Release Notes](https://nightlies.apache.org/flink/flink-docs-stable/release-notes/flink-2.3/)
- [FLIP-547: Support checkpoint during recovery](https://cwiki.apache.org/confluence/display/FLINK/FLIP-547%3A%2BSupport%2Bcheckpoint%2Bduring%2Brecovery)
- [Apache Flink Checkpointing Documentation](https://nightlies.apache.org/flink/flink-docs-master/docs/dev/datastream/fault-tolerance/checkpointing/)
