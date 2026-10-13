---
title: "Debezium 증분 스냅샷: CDC를 유지하며 백필하는 원리"
date: 2026-10-09 09:00:00 +0900
categories: [data-engineering]
tags: [Debezium, CDC, PostgreSQL, backfill, snapshot]
---

Debezium 증분 스냅샷은 테이블을 작은 청크로 읽어 실시간 CDC와 백필을 함께 진행한다.
스냅샷 윈도 안에서 같은 키의 변경이 발견되면 버퍼의 READ를 버리고 변경 이벤트를 전달한다.
소비자는 READ가 항상 먼저 온다고 가정하지 말고, 키 기반 반영과 재처리 안전성을 설계해야 한다.

## 백필과 변경 수집이 만나는 지점

새 테이블을 수집 대상에 추가하거나 누락된 데이터를 다시 채울 때, 테이블 조회와 변경 로그를 단순히 합치면 오래된 조회 결과가 최신 값을 덮을 수 있다. Debezium의 증분 스냅샷은 이 충돌을 조정한다. 이름의 ‘증분’은 변경된 행만 조회한다는 뜻이 아니라, 대상 테이블을 나누어 스냅샷한다는 뜻이다.

2026년 10월 9일 확인한 공식 stable 문서는 3.7로 표시된다. 아래 내용은 신규 릴리스 소개가 아니라 PostgreSQL 커넥터의 기존 기능 해설이다. 캡처 대상과 signaling 설정을 준비하고 `execute-snapshot` 신호로 실행한다. 전체 스냅샷이 끝날 때까지 스트리밍을 미루지 않아도 된다. [공식 문서](https://debezium.io/documentation/reference/stable/connectors/postgresql.html#postgresql-incremental-snapshots)

## 스냅샷 윈도는 충돌을 판별하는 구간이다

기본 키로 정렬한 청크를 읽어 READ 이벤트를 메모리 버퍼에 담는다. 윈도가 열린 동안 로그에서 같은 키의 변경을 만나면 해당 READ를 제거하고 변경 이벤트를 내보낸다. 윈도가 닫히면 남은 READ만 전달한다. 여기서 워터마크는 로그와 청크를 조정하는 경계이며, 이벤트 시간 윈도의 종료 기준과는 용도가 다르다. [설계 문서 DDD-3](https://github.com/debezium/debezium-design-documents/blob/main/DDD-3.md)

<figure>
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 760 280" role="img" aria-labelledby="inc-title inc-desc" style="width:100%;height:auto">
<title id="inc-title">증분 스냅샷의 동일 키 충돌 처리</title>
<desc id="inc-desc">청크 버퍼의 A와 B 중 A에 대한 변경이 윈도 안에서 도착하면 A의 READ를 제거한다. 변경 A는 전달하고 윈도가 닫히면 남은 READ B를 전달한다.</desc>
<rect width="760" height="280" rx="16" fill="#f8fafc"/>
<g font-family="sans-serif" fill="#0f172a" text-anchor="middle">
<text x="380" y="32" font-size="18" font-weight="bold">한 청크의 snapshot window</text>
<rect x="20" y="60" width="200" height="125" rx="10" fill="#dbeafe"/>
<text x="120" y="92" font-size="17">청크 조회 → 버퍼</text>
<text x="120" y="125" font-size="16">READ A</text>
<text x="120" y="155" font-size="16">READ B</text>
<rect x="280" y="60" width="200" height="125" rx="10" fill="#fef3c7"/>
<text x="380" y="92" font-size="17">로그에서 UPDATE A</text>
<text x="380" y="125" font-size="16">같은 키 A 확인</text>
<text x="380" y="155" font-size="16">버퍼의 READ A 제거</text>
<rect x="540" y="60" width="200" height="125" rx="10" fill="#d1fae5"/>
<text x="640" y="92" font-size="17">출력 이벤트</text>
<text x="640" y="125" font-size="16">UPDATE A 전달</text>
<text x="640" y="155" font-size="16">닫힌 뒤 READ B 전달</text>
<text x="380" y="232" font-size="16">윈도 내부 충돌 처리 ≠ 파이프라인 전체 중복 제거</text>
</g>
<g stroke="#475569" stroke-width="3" fill="none"><path d="M225 122 H266 M256 113 L266 122 L256 131"/><path d="M485 122 H526 M516 113 L526 122 L516 131"/></g>
</svg>
<figcaption>직접 작성한 개념도. A와 B는 설명용 키이며, 그림은 윈도 안에서 A가 갱신된 경우를 나타낸다.</figcaption>
</figure>

## 소비자가 지켜야 할 계약

READ는 행의 최초 상태를 뜻하지 않는다. UPDATE만 먼저 도착하거나, 한 번도 보지 못한 키의 DELETE가 도착할 수 있다. 또한 청크별로 조회하므로 전체 테이블을 동일 시점에 고정한 결과로 해석하면 안 된다. 공식 설명은 재시작에도 at-least-once 전달 의미가 적용된다고 명시한다. [동작 원리와 제한](https://debezium.io/blog/2021/10/07/incremental-snapshots/)

이 특성에서 도출되는 설계 기준은 키 기반 upsert, 없는 키에 대한 삭제 허용, 재전달 대응이다. 금액 누적처럼 이벤트마다 값을 더하는 소비자는 특히 별도 중복 처리 전략이 필요하다. downstream에서 같은 키를 병렬 처리해 순서를 뒤집는 문제까지 스냅샷 버퍼가 해결해 주지는 않는다.

## 적용 전 확인할 것

부분 재수집과 운영 중 백필에는 적합하지만, 단일 시점의 정산 결과가 필요한 작업은 별도 일관성 전략이 필요하다. `incremental.snapshot.chunk.size`를 늘리면 조회 횟수를 줄일 수 있지만 버퍼 메모리가 늘어난다. DB 조회 부하, 스트리밍 지연, sink 처리량을 함께 측정한다.

PostgreSQL 커넥터 문서는 증분 스냅샷 실행 중 스키마 변경을 지원하지 않는다고 명시한다. 실행 구간의 DDL을 조정하고, 재시작·동일 키 갱신·삭제를 함께 시험해야 한다. 소스 조회 완료와 목적지 반영 완료도 구분해 확인한다.

## 참고 링크

- [Debezium PostgreSQL 커넥터 — 증분 스냅샷과 설정](https://debezium.io/documentation/reference/stable/connectors/postgresql.html)
- [Incremental Snapshots in Debezium — 2021년 공식 기술 글](https://debezium.io/blog/2021/10/07/incremental-snapshots/)
- [DDD-3 — Incremental snapshotting 설계 문서](https://github.com/debezium/debezium-design-documents/blob/main/DDD-3.md)
