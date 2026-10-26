---
title: "Spark 다중 워터마크: min과 max가 바꾸는 지연 데이터 처리"
date: 2026-10-20 09:02:00 +0900
categories: [data-engineering]
tags: [spark, structured-streaming, watermark, stateful-streaming]
---

Spark Structured Streaming에서 여러 입력을 합칠 때 워터마크 정책은 결과 확정 속도와 지연 데이터 처리 범위를 함께 바꾼다. 기본값 `min`을 `max`로 바꾸는 것은 단순한 성능 조정이 아니다. 느린 입력의 데이터를 잃을 수 있는 설계 선택이다.

## 워터마크는 벽시계 타이머가 아니다

`withWatermark`는 관측한 최대 이벤트 시각에서 허용 지연을 뺀 값을 바탕으로 워터마크를 계산한다. 처리 시각이 10분 지났다고 워터마크도 10분 전진하는 것은 아니다. 집계에서는 이 경계를 결과 확정과 상태 정리에 활용한다. 실제 반영에는 조정 지연이 있으므로, 허용 지연보다 오래된 데이터가 항상 즉시 버려진다고 해석해서도 안 된다. [withWatermark API](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/api/pyspark.sql.DataFrame.withWatermark.html)

## 두 입력 중 누구의 속도를 따를까

입력별 워터마크가 A는 12:10, B는 11:50이라고 가정하자. 전역 워터마크는 기본 정책에서 11:50이 된다. B의 진행이 멈추면 A가 계속 들어와도 전역 경계가 정체될 수 있다. 윈도 집계 결과 확정과 상태 정리가 늦어지는 이유다.

`spark.sql.streaming.multipleWatermarkPolicy=max`는 같은 상황에서 12:10을 선택한다. 경계는 빨리 전진하지만 B의 늦은 데이터가 제거될 위험이 커진다. 입력별 허용 지연 설정만 보고 B의 처리 범위가 보장된다고 판단하면 안 된다. [다중 워터마크 정책](https://spark.apache.org/docs/latest/streaming/apis-on-dataframes-and-datasets.html#policy-for-handling-multiple-watermarks)

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 760 270" role="img" aria-labelledby="wm-title wm-desc" style="width:100%;height:auto;background:#f8fafc;border-radius:12px">
  <title id="wm-title">입력별 워터마크와 min, max 정책 비교</title>
  <desc id="wm-desc">A의 워터마크는 12시 10분, B는 11시 50분이다. min은 11시 50분, max는 12시 10분을 전역 경계로 선택한다.</desc>
  <g font-family="sans-serif" text-anchor="middle">
    <rect x="30" y="30" width="240" height="70" rx="10" fill="#dbeafe"/>
    <text x="150" y="58" fill="#172554" font-size="18">Input A watermark</text>
    <text x="150" y="84" fill="#172554" font-size="22">12:10</text>
    <rect x="30" y="160" width="240" height="70" rx="10" fill="#e0e7ff"/>
    <text x="150" y="188" fill="#172554" font-size="18">Input B watermark</text>
    <text x="150" y="214" fill="#172554" font-size="22">11:50</text>
    <path d="M270 65 L350 130 M270 195 L350 130 M350 130 L430 65 M350 130 L430 195" fill="none" stroke="#475569" stroke-width="3"/>
    <rect x="430" y="25" width="300" height="85" rx="10" fill="#dcfce7"/>
    <text x="580" y="58" fill="#14532d" font-size="22">min → 11:50</text>
    <text x="580" y="88" fill="#14532d" font-size="17">Follow the slower watermark</text>
    <rect x="430" y="155" width="300" height="85" rx="10" fill="#ffedd5"/>
    <text x="580" y="188" fill="#7c2d12" font-size="22">max → 12:10</text>
    <text x="580" y="218" fill="#7c2d12" font-size="17">Risk: drop slower input data</text>
  </g>
</svg>

*직접 작성한 개념 도식. 시각은 관측된 입력별 워터마크의 가상 예시이며, 개별 행의 처리 결과를 나타내지 않는다.*

## 적용 전에 정할 운영 기준

완전성이 중요한 집계에서는 우선 `min`을 유지하고 입력 지연의 원인을 해결하는 편이 합리적이다. 일부 지연 데이터를 제외해도 되는 실시간 지표라면 `max`를 검토하되, 누락 허용량과 사후 보정 경로를 먼저 정해야 한다. 이는 정책의 동작에서 도출한 설계 판단이다.

상태 크기 증가만으로 정책을 바꾸지 말고 입력별 이벤트 시각 분포, 워터마크 정체 시간, 처리 지연을 함께 비교한다. 검증 데이터에는 한 입력의 중단·재개와 오래된 이벤트의 재유입을 포함한다. 화면이 빨리 갱신되는지만 확인하면 늦게 도착한 데이터의 누락을 놓칠 수 있다.

## 체크포인트와 정책을 함께 관리한다

공식 문서는 기존 쿼리의 체크포인트를 이어 사용할 때 다중 워터마크 정책을 유지하도록 요구한다. 정책 변경은 워터마크 일관성을 깨뜨릴 수 있기 때문이다. 변경이 필요하면 별도 체크포인트로 시작할 새 쿼리, 재처리 범위, 출력 중복 방지 방법을 함께 설계한다. 운영 체크포인트를 바로 삭제하는 방식은 복구 선택지를 줄인다. [체크포인트 관련 설정 제약](https://spark.apache.org/docs/latest/streaming/additional-information.html#miscellaneous-notes)

## 참고 자료

2026-10-20에 공식 최신 문서가 Spark 4.2.0으로 표시되는 것을 확인했다. 신규 릴리스 소개가 아닌 기존 동작의 운영 해설이다.

- [Apache Spark — 다중 워터마크 정책](https://spark.apache.org/docs/latest/streaming/apis-on-dataframes-and-datasets.html#policy-for-handling-multiple-watermarks)
- [PySpark — withWatermark API](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/api/pyspark.sql.DataFrame.withWatermark.html)
- [Apache Spark — 체크포인트 관련 설정 제약](https://spark.apache.org/docs/latest/streaming/additional-information.html#miscellaneous-notes)
