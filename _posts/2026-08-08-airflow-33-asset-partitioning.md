---
title: "Airflow 3.3 Asset Partitioning을 파티션 단위 오케스트레이션으로 보기"
date: 2026-08-08 16:43:00 +0900
categories: [data-engineering]
tags: [apache-airflow, asset-partitioning, orchestration, data-platform, scheduling]
---

Apache Airflow 3.3.0에서 눈에 띄는 핵심 변화는 Asset Partitioning 확장이다. Airflow 3.2에서 도입된 파티션 단위 asset scheduling 위에 `RollupMapper`, `FanOutMapper`, `FixedKeyMapper`, `SegmentWindow`, `wait_policy`가 추가되면서, "데이터가 갱신되면 DAG를 실행한다"는 모델을 "어떤 파티션이 준비됐을 때 어떤 downstream 파티션을 실행할지"로 더 세밀하게 표현할 수 있게 됐다.

기존 시간 기반 DAG는 보통 스케줄러의 시간표가 실행 단위였다. 예를 들어 매시간 원천 데이터를 적재하고, 매일 집계를 만들려면 daily DAG가 별도 시간에 실행된 뒤 누락된 hourly partition을 직접 검사하는 방식이 흔하다. Asset Partitioning은 이 판단 일부를 스케줄링 모델로 끌어올린다. upstream task가 `partition_key`를 가진 asset event를 내보내고, downstream DAG는 mapper를 통해 그 key를 자신이 처리할 partition으로 변환한다.

<svg viewBox="0 0 760 260" role="img" aria-label="Airflow asset partitioning flow" xmlns="http://www.w3.org/2000/svg">
  <rect x="24" y="34" width="200" height="58" rx="8" fill="#e8f2ff" stroke="#2f6fb0"/>
  <text x="124" y="58" text-anchor="middle" font-size="15" font-family="Arial, sans-serif" fill="#143b63">hourly asset events</text>
  <text x="124" y="78" text-anchor="middle" font-size="12" font-family="Arial, sans-serif" fill="#143b63">2026-03-10T00..23</text>
  <path d="M224 63 H324" stroke="#333" stroke-width="2" marker-end="url(#arrow)"/>
  <rect x="324" y="34" width="170" height="58" rx="8" fill="#fff6df" stroke="#a36a00"/>
  <text x="409" y="58" text-anchor="middle" font-size="15" font-family="Arial, sans-serif" fill="#664000">RollupMapper</text>
  <text x="409" y="78" text-anchor="middle" font-size="12" font-family="Arial, sans-serif" fill="#664000">DayWindow + policy</text>
  <path d="M494 63 H594" stroke="#333" stroke-width="2" marker-end="url(#arrow)"/>
  <rect x="594" y="34" width="142" height="58" rx="8" fill="#e9f7ef" stroke="#2f7d4f"/>
  <text x="665" y="58" text-anchor="middle" font-size="15" font-family="Arial, sans-serif" fill="#1f5135">daily DAG run</text>
  <text x="665" y="78" text-anchor="middle" font-size="12" font-family="Arial, sans-serif" fill="#1f5135">2026-03-10</text>
  <rect x="72" y="150" width="616" height="58" rx="8" fill="#f7f7f7" stroke="#777"/>
  <text x="380" y="174" text-anchor="middle" font-size="14" font-family="Arial, sans-serif" fill="#333">scheduler decision: all partitions arrived, or MinimumCount(n) threshold met</text>
  <text x="380" y="194" text-anchor="middle" font-size="12" font-family="Arial, sans-serif" fill="#555">execution becomes data-readiness driven, not only cron-time driven</text>
  <defs>
    <marker id="arrow" markerWidth="10" markerHeight="10" refX="8" refY="3" orient="auto">
      <path d="M0,0 L0,6 L9,3 z" fill="#333"/>
    </marker>
  </defs>
</svg>

핵심은 mapper가 partition key의 의미를 명확히 한다는 점이다. `IdentityMapper`는 key를 그대로 유지하고, `StartOfDayMapper` 같은 temporal mapper는 timestamp형 key를 일 단위로 정규화한다. `ProductMapper`는 `region|timestamp`처럼 composite key를 세그먼트별로 변환한다. Airflow 문서는 변환된 upstream key가 downstream이 기다리는 key와 맞지 않으면 downstream DAG가 실행되지 않는다고 설명한다. 즉 파티션 key 형식은 단순 문자열이 아니라 운영 계약에 가깝다.

`RollupMapper`는 더 거친 downstream 단위를 만들 때 중요하다. hourly partition 24개가 모두 들어온 뒤 daily summary를 실행하거나, 일별 입력을 주별 리포트로 묶는 식이다. 기본 정책인 `WaitForAll`은 모든 partition이 도착할 때까지 기다린다. 반대로 `MinimumCount(n)`은 일부 partition이 늦거나 누락되어도 정해진 개수 이상이 도착하면 실행을 허용한다. 이 선택은 정합성과 적시성 사이의 설계 판단이다.

```python
# 개념 예시: 시간별 입력을 일별 집계 DAG로 연결하는 형태
from airflow.sdk import Asset, DayWindow, PartitionedAssetTimetable
from airflow.sdk import RollupMapper, StartOfHourMapper

hourly_sales = Asset(uri="s3://warehouse/sales/hourly", name="hourly_sales")

daily_schedule = PartitionedAssetTimetable(
    assets=hourly_sales,
    default_partition_mapper=RollupMapper(
        upstream_mapper=StartOfHourMapper(),
        window=DayWindow(),
    ),
)
```

운영 관점에서는 세 가지를 먼저 정해야 한다. 첫째, partition key의 표준 포맷이다. 시간대가 섞이면 DST나 로컬 타임 경계에서 daily rollup이 영원히 대기할 수 있으므로 UTC 기준을 기본값으로 두는 편이 안전하다. 둘째, 누락 partition 처리 정책이다. 재무·정산처럼 완전성이 중요한 배치는 `WaitForAll`이 맞고, 대시보드 최신성처럼 지연 허용보다 노출 시점이 중요한 경우에는 `MinimumCount(n)`을 검토할 수 있다. 셋째, fan-out 상한이다. Airflow 3.3은 upstream event 하나가 너무 많은 downstream key를 만들지 않도록 `partition_mapper_max_downstream_keys` 계열의 제한을 둔다. 파티션 모델은 스케줄링을 정교하게 만들지만, 잘못 설계하면 scheduler 부하와 pending run 증가로 이어진다.

따라서 Asset Partitioning은 단순한 Airflow 기능 추가라기보다 데이터 플랫폼의 dependency contract를 명시화하는 도구로 보는 편이 적절하다. 크론 시간표만으로는 "데이터가 준비됐는가"를 표현하기 어렵다. 반면 partition-aware scheduling은 준비된 데이터 범위, 기다릴 조건, downstream 실행 단위를 DAG 정의에 드러낸다. 데이터 레이크·CDC·집계 파이프라인처럼 파티션 단위 재처리와 부분 지연이 자주 발생하는 환경일수록 이 차이가 운영 복잡도를 줄이는 기준이 된다.

## 참고 링크

- [Apache Airflow 3.3.0 Release Notes](https://airflow.apache.org/docs/apache-airflow/stable/release_notes.html)
- [Airflow Asset Definitions](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/assets.html)
- [Airflow Task and Asset State Store Overview](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/task-and-asset-state-store.html)
