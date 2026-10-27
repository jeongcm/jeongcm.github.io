---
title: "Flink Async I/O: capacity와 장애 복구 시 외부 재조회 경계"
date: 2026-10-27 15:08:00 +0900
categories: [data-engineering]
tags: [flink, async-io, checkpoint, backpressure, data-enrichment]
---

Flink Async I/O는 외부 조회를 겹쳐 실행해 스트림의 대기 시간을 줄인다. 그러나 동시 요청을 늘리는 것과 장애 후 같은 결과를 재현하는 것은 별개의 문제다. 외부 서비스의 처리 여력과 재조회 시점의 데이터 의미를 함께 설계해야 한다.

## capacity는 서브태스크 단위다

2026년 10월 27일 확인한 Flink stable 문서는 2.3.0을 표시한다. 새 기능 발표가 아니라 현재 공식 문서의 동작을 정리한 글이다. `asyncInvoke`는 요청을 시작한 뒤 빠르게 반환해야 한다. 내부에서 응답을 기다리면 비동기 처리의 이점이 사라진다.

`capacity`는 서브태스크마다 진행할 수 있는 비동기 요청 수를 제한하며, 한도에 도달하면 backpressure가 발생한다. 병렬도를 늘릴 때 외부 시스템에 가해지는 부하도 다시 계산해야 한다. 예를 들어 병렬도 4, capacity 50이면 연산자 전체의 용량 예산은 200이다. 이는 초당 200건이라는 처리율 보장이 아니다. 요청 분기나 클라이언트의 별도 재시도도 점검해야 한다. [공식 Async I/O 문서](https://nightlies.apache.org/flink/flink-docs-stable/docs/dev/datastream/operators/asyncio/)

## 체크포인트가 외부 조회값까지 고정하지는 않는다

Async I/O 연산자는 진행 중인 요청의 입력 레코드를 체크포인트에 저장하고, 복구 시 요청을 다시 실행한다. 아래 그림은 같은 키를 재조회하는 동안 참조 데이터가 바뀌는 개념 예시다.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 760 260" role="img" aria-labelledby="async-title async-desc" style="max-width:100%;height:auto;display:block;background:#f8fafc">
<title id="async-title">체크포인트 복구와 외부 재조회</title>
<desc id="async-desc">체크포인트에 저장된 입력 키 K를 복원해 재조회하면 외부 데이터 변경으로 이전과 다른 버전을 받을 수 있다.</desc>
<defs><marker id="async-arrow" markerWidth="10" markerHeight="10" refX="8" refY="3" orient="auto"><path d="M0,0 L0,6 L9,3 z" fill="#475569"/></marker></defs>
<g font-family="sans-serif" text-anchor="middle" fill="#0f172a">
<rect x="20" y="35" width="200" height="70" rx="10" fill="#dbeafe"/>
<text x="120" y="64" font-size="18">입력 키 K</text><text x="120" y="89" font-size="15">진행 중인 조회</text>
<rect x="540" y="35" width="200" height="70" rx="10" fill="#e2e8f0"/>
<text x="640" y="64" font-size="18">외부 데이터 v1</text><text x="640" y="89" font-size="15">최초 조회 시점</text>
<rect x="20" y="165" width="200" height="70" rx="10" fill="#dbeafe"/>
<text x="120" y="194" font-size="18">입력 키 K 복원</text><text x="120" y="219" font-size="15">체크포인트에서 복구</text>
<rect x="540" y="165" width="200" height="70" rx="10" fill="#fef3c7"/>
<text x="640" y="194" font-size="18">외부 데이터 v2</text><text x="640" y="219" font-size="15">변경 후 재조회 시점</text>
<text x="380" y="55" font-size="16">최초 요청</text><text x="380" y="185" font-size="16">장애 후 요청 재실행</text>
<text x="380" y="137" font-size="15">입력 복원 ≠ 외부 데이터 버전 고정</text>
</g>
<g fill="none" stroke="#475569" stroke-width="2" marker-end="url(#async-arrow)"><path d="M220 72 H530"/><path d="M220 202 H530"/><path d="M120 108 V155"/></g>
</svg>

*직접 작성한 도식. v1·v2는 외부 참조 데이터의 버전이며 체크포인트 번호가 아니다.*

Flink의 exactly-once는 각 이벤트가 관리 상태에 한 번 반영된다는 의미다. 함수나 외부 호출이 물리적으로 한 번만 실행된다는 뜻은 아니다. 종단 간 보장에는 재생 가능한 소스와 트랜잭션 또는 멱등성을 제공하는 싱크도 필요하다. [Fault Tolerance 문서](https://nightlies.apache.org/flink/flink-docs-stable/docs/learn-flink/fault_tolerance/#exactly-once-guarantees)

이 동작에서 도출되는 설계 판단은 다음과 같다. 최신 등급을 붙이는 조회라면 복구 후 값 변경을 허용할 수 있다. 반면 이벤트 발생 당시의 가격이 필요하다면 버전이나 기준 시각으로 조회할 수 있는 데이터 모델이 필요하다. 외부 쓰기를 수행한다면 중복 호출을 받아도 효과가 중복되지 않는 별도 계약을 마련해야 한다.

## 재시도보다 먼저 실패를 분류한다

AWS의 오류 처리 글은 일시적 오류와 반복해도 해결되지 않는 오류를 구분한다. 네트워크의 일시 장애는 제한된 재시도 후보지만, 잘못된 데이터는 격리와 수정 경로가 필요하다. [AWS 원문](https://aws.amazon.com/blogs/big-data/error-handling-in-apache-flink-applications/)

운영에서는 외부 응답 지연, 오류율, 재시도 횟수와 Flink backpressure를 함께 관찰하는 편이 좋다. capacity 증설만으로 느린 외부 서비스를 압박하지 않도록 한다. 특히 복구 시험에서는 처리량뿐 아니라 같은 입력을 재조회했을 때 결과가 달라도 되는지 확인해야 한다. 그 허용 범위가 조회 방식과 복구 정책을 결정한다.

## 참고 자료

- [Apache Flink — Async I/O](https://nightlies.apache.org/flink/flink-docs-stable/docs/dev/datastream/operators/asyncio/)
- [Apache Flink — Fault Tolerance](https://nightlies.apache.org/flink/flink-docs-stable/docs/learn-flink/fault_tolerance/)
- [AWS Big Data Blog — Handle errors in Apache Flink applications on AWS](https://aws.amazon.com/blogs/big-data/error-handling-in-apache-flink-applications/) (2025-02-06)
