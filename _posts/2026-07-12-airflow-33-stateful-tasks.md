---
title: "Airflow 3.3의 Stateful Task가 바꾸는 오케스트레이션 설계 기준"
date: 2026-07-12 18:15:00 +0900
categories: [data-engineering]
tags: [apache-airflow, orchestration, data-platform, stateful-task, task-sdk]
---

Apache Airflow 3.3.0 릴리스에서 눈에 띄는 변화는 단순한 UI 개선보다 **stateful task**와 **multi-language task SDK**다. Airflow가 Python DAG 스케줄러라는 경계를 넘어, 태스크 실행 상태와 런타임 선택을 더 명시적으로 다루기 시작했다는 점이 중요하다.

기존 Airflow 운영에서 태스크는 보통 "실행 후 종료되는 작업"으로 모델링된다. 하지만 실제 데이터 플랫폼의 작업은 그렇게 단순하지 않다. 장시간 대기하는 센서, 외부 시스템의 비동기 완료 이벤트, 재시도 가능한 백필, 중간 상태를 가진 데이터 품질 검사처럼 태스크 내부 상태를 보존하거나 재개해야 하는 경우가 많다. 이런 요구를 모두 DAG 구조나 외부 테이블, XCom, 커스텀 오퍼레이터로 우회하면 운영 규칙이 흩어진다.

Stateful task는 이 문제를 오케스트레이터가 직접 다뤄야 할 영역으로 끌어온다. 핵심은 태스크가 단순 성공/실패 플래그만 갖는 것이 아니라, 실행 중인 작업의 상태와 재개 지점을 더 명확히 표현할 수 있다는 점이다. 데이터 엔지니어링 관점에서는 "작업이 다시 시작되면 어디서부터 안전하게 이어갈 수 있는가"가 설계 질문이 된다.

<figure class="post-diagram">
  <svg viewBox="0 0 820 300" role="img" aria-labelledby="airflow-state-title airflow-state-desc" xmlns="http://www.w3.org/2000/svg">
    <title id="airflow-state-title">Stateful task orchestration flow</title>
    <desc id="airflow-state-desc">A stateful orchestration model records task state, external wait points, resume positions, and operational metrics.</desc>
    <defs>
      <marker id="arrow-airflow-state" markerWidth="11" markerHeight="11" refX="9" refY="5.5" orient="auto">
        <path d="M1,1 L10,5.5 L1,10 Z" fill="#4d6472" />
      </marker>
      <style>
        .box { fill: #f7f4e8; stroke: #4d6472; stroke-width: 2; rx: 8; }
        .state { fill: #e7f0f5; stroke: #4d6472; stroke-width: 2; rx: 8; }
        .metric { fill: #edf7ec; stroke: #5b7d55; stroke-width: 2; rx: 8; }
        .label { font: 700 16px -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; fill: #23323a; }
        .small { font: 13px -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; fill: #57666c; }
        .line { stroke: #4d6472; stroke-width: 3; fill: none; marker-end: url(#arrow-airflow-state); }
      </style>
    </defs>
    <rect x="32" y="96" width="132" height="78" class="box" />
    <text x="98" y="128" text-anchor="middle" class="label">Task start</text>
    <text x="98" y="152" text-anchor="middle" class="small">schedule trigger</text>

    <rect x="218" y="52" width="150" height="78" class="state" />
    <text x="293" y="84" text-anchor="middle" class="label">State store</text>
    <text x="293" y="108" text-anchor="middle" class="small">progress point</text>

    <rect x="218" y="170" width="150" height="78" class="box" />
    <text x="293" y="202" text-anchor="middle" class="label">External wait</text>
    <text x="293" y="226" text-anchor="middle" class="small">event or callback</text>

    <rect x="422" y="96" width="150" height="78" class="state" />
    <text x="497" y="128" text-anchor="middle" class="label">Resume logic</text>
    <text x="497" y="152" text-anchor="middle" class="small">safe restart</text>

    <rect x="626" y="96" width="160" height="78" class="metric" />
    <text x="706" y="128" text-anchor="middle" class="label">Operations</text>
    <text x="706" y="152" text-anchor="middle" class="small">latency, retries, load</text>

    <path d="M164 135 H218" class="line" />
    <path d="M368 91 C398 96 402 116 422 125" class="line" />
    <path d="M368 209 C398 204 402 154 422 144" class="line" />
    <path d="M572 135 H626" class="line" />
  </svg>
  <figcaption>상태 있는 오케스트레이션은 실행 여부보다 재개 지점, 외부 대기, 운영 지표를 함께 모델링하는 문제다.</figcaption>
</figure>

Multi-language task SDK도 같은 맥락에서 봐야 한다. 데이터 플랫폼 팀은 Python만 쓰지 않는다. Spark/Java, JVM 기반 커넥터, Go 기반 내부 도구, Rust로 작성된 검증기처럼 작업 런타임이 섞인다. Airflow가 언어별 태스크 작성 경로를 넓히면, 모든 실행 로직을 Python 래퍼로 감싸는 부담을 줄이고 각 런타임의 배포·관측·테스트 방식을 더 자연스럽게 가져갈 수 있다.

다만 적용 기준은 보수적으로 잡는 편이 좋다. Stateful task가 있다고 해서 모든 중간 상태를 Airflow 안으로 넣어야 하는 것은 아니다. 데이터 자체의 정합성, 멱등성, 체크포인트는 여전히 처리 엔진이나 저장소가 책임지는 편이 낫다. Airflow는 "언제 실행하고, 실패 시 어떻게 재개하며, 외부 완료를 어떻게 추적할지"를 관리하는 계층으로 두는 것이 안정적이다.

운영에서 확인할 지표도 달라진다. 단순 DAG 성공률뿐 아니라 state 전이 지연, 장기 실행 태스크 수, 재개 실패율, 외부 이벤트 대기 시간, scheduler와 executor의 부하를 함께 봐야 한다. 특히 KubernetesExecutor나 CeleryExecutor를 쓰는 환경에서는 stateful task가 worker 점유 시간과 리소스 회수 정책에 어떤 영향을 주는지 확인해야 한다.

Airflow 3.3은 "오케스트레이션은 stateless하게 트리거만 하면 된다"는 관성을 다시 생각하게 만든다. 현대 데이터 플랫폼에서 중요한 것은 DAG 그래프를 예쁘게 그리는 일이 아니라, 실패·대기·재시작·런타임 다양성을 운영 가능한 상태 모델로 표현하는 일이다.

## 참고 링크

- [Apache Airflow 3.3.0 릴리스 블로그](https://airflow.apache.org/blog/airflow-3-3-0/)
- [Apache Airflow 3.3.0 Release Notes](https://airflow.apache.org/docs/apache-airflow/3.3.0/release_notes.html)
