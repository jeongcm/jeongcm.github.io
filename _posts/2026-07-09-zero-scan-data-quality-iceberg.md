---
title: "데이터 품질 검사를 꼭 테이블 스캔으로 해야 할까"
date: 2026-07-09 09:18:00 +0900
categories: [technical-knowledge, data-engineering]
tags: [apache-iceberg, data-quality, lakehouse, metadata, observability]
---

오늘 본 글은 LinkedIn의 Iceberg 기반 데이터 품질 모니터링 논문 **Zero-Scan Data Quality: Leveraging Table Format Metadata for Continuous Observability at Scale**다. 핵심 아이디어는 단순하다. 데이터 품질 검사를 매번 테이블을 읽어서 하지 말고, Iceberg 같은 테이블 포맷이 이미 쓰기 시점에 남기는 메타데이터를 먼저 활용하자는 것이다.

보통 데이터 품질 검사는 SQL 테스트나 profiling job으로 수행된다. row count가 일정 범위 안에 있는지, 특정 컬럼의 null 비율이 급증하지 않았는지, 값의 min/max가 말이 되는지 확인하려면 실제 데이터를 읽는다. 문제는 규모가 커질수록 이 방식이 비싸진다는 점이다. 테이블이 수천 개만 되어도 모든 테이블을 주기적으로 스캔하는 비용은 빠르게 커지고, 검사가 늦게 돌면 나쁜 데이터는 이미 downstream으로 흘러간다.

Iceberg는 이 문제를 다르게 볼 수 있게 만든다. Iceberg manifest에는 data file별 record count, null count, NaN count, lower/upper bound 같은 통계가 저장된다. 원래는 query planning과 data skipping을 위해 쓰이는 정보지만, row count 이상치나 null 비율 변화, 값 범위 위반 같은 품질 신호에도 그대로 쓸 수 있다. 논문은 LinkedIn 규모의 Iceberg 테이블에서 이런 메타데이터만으로 상당수의 품질 규칙을 처리할 수 있다고 설명한다.

여기서 중요한 포인트는 "zero-scan"이 모든 품질 검사를 대체한다는 뜻이 아니라는 점이다. 참조 무결성, 복잡한 SQL 조건, regex 기반 검증, 실제 값 분포를 정밀하게 봐야 하는 검사는 여전히 스캔이 필요하다. 대신 기본적인 freshness, row count, null rate, min/max, snapshot 간 변화 감지는 이미 존재하는 메타데이터로 먼저 처리할 수 있다. 품질 검사를 비싼 full scan부터 시작하지 않고, 싸고 빠른 metadata layer부터 시작하는 구조다.

이 관점은 Lakehouse 운영에서 꽤 실용적이다. Iceberg를 쓰다 보면 데이터 파일뿐 아니라 manifest, snapshot, delete file, compaction 같은 메타데이터 운영이 중요해진다. 특히 CDC나 스트리밍 파이프라인에서는 작은 파일과 delete file이 쌓이고, compaction 시점에 통계가 다시 정리된다. 품질 모니터링도 이 lifecycle을 이해해야 한다. 예를 들어 merge-on-read 상태의 통계는 pending delete를 완전히 반영하지 못할 수 있고, compaction 후에는 더 정확한 통계가 된다.

내가 이 글에서 가져가고 싶은 실무 질문은 세 가지다.

1. 우리 데이터 품질 검사는 무조건 테이블을 읽는 방식으로 시작하고 있나?
2. Iceberg metadata table이나 manifest 통계를 품질 신호로 쓰고 있나?
3. compaction, rewrite, snapshot expire 같은 maintenance 작업이 품질 알림에 noise를 만들고 있지는 않나?

데이터 품질은 보통 Great Expectations, dbt test, custom SQL 같은 도구 이름으로 이야기되지만, 그 아래에는 "무엇을 읽어서 판단할 것인가"라는 비용 모델이 있다. Lakehouse에서는 이 질문에 대한 답이 조금 달라진다. 모든 검사를 row scan으로 밀어붙이기보다, 테이블 포맷이 이미 갖고 있는 commit-time metadata를 1차 관측 계층으로 쓰는 것이 더 자연스럽다.

결국 중요한 것은 품질 검사를 더 많이 하는 것이 아니라, 더 빨리 싸게 알아차리는 것이다. Iceberg metadata는 이미 쓰기 경로에서 만들어지는 정보다. 이 정보를 품질 모니터링의 첫 번째 신호로 삼으면, 데이터 플랫폼은 더 적은 비용으로 더 넓은 테이블을 감시할 수 있다. "데이터를 읽지 않고도 알 수 있는 것"을 먼저 챙기는 설계가 Lakehouse 운영에서는 점점 더 중요해질 것 같다.

## 참고 링크

- [Zero-Scan Data Quality: Leveraging Table Format Metadata for Continuous Observability at Scale](https://arxiv.org/abs/2605.30308)
- [Apache Iceberg Table Specification](https://iceberg.apache.org/spec/)
- [Apache Iceberg Puffin Spec](https://iceberg.apache.org/docs/latest/puffin-spec/)
