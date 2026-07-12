---
title: "Iceberg Puffin은 벡터 인덱스를 어디에 붙일 수 있을까"
date: 2026-07-11 18:05:00 +0900
categories: [technical-knowledge, data-engineering]
tags: [apache-iceberg, lakehouse, vector-search, puffin, metadata]
---

Lakehouse에서 벡터 검색을 붙일 때 먼저 부딪히는 문제는 검색 알고리즘보다 운영 경계다. 데이터는 Iceberg 테이블과 object storage에 있는데, ANN(Approximate Nearest Neighbor) 인덱스만 별도 벡터 DB나 저장소에 두면 snapshot, time travel, compaction, orphan file 정리, 접근 권한을 다시 맞춰야 한다. 데이터와 인덱스의 수명주기가 갈라지는 순간 정합성 관리는 별도 플랫폼 문제가 된다.

2026년 6월 arXiv에 공개된 **Puffin-Backed Vector Indexes** 논문은 이 경계를 Iceberg 내부 메타데이터 모델로 끌어오는 방식을 제안한다. 핵심 아이디어는 벡터 인덱스를 독립 서비스가 아니라 Iceberg의 Puffin sidecar file에 담고 snapshot summary로 연결하는 것이다. 논문은 Puffin을 Vamana graph 같은 ANN 인덱스 조각의 컨테이너로 사용하고, Iceberg REST catalog의 optimistic commit 경로에 index build와 incremental refresh를 결합하는 설계를 설명한다.

이 접근이 흥미로운 이유는 Iceberg의 기본 장점과 같은 방향을 보기 때문이다. Iceberg snapshot은 특정 시점의 테이블 상태를 나타내고, manifest는 그 snapshot이 참조하는 data/delete file을 추적한다. 공식 spec도 snapshot summary, statistics, partition statistics 같은 필드를 통해 테이블 상태와 운영 메타데이터를 기록할 수 있게 한다. Puffin은 이런 메타데이터 확장 지점으로 쓰일 수 있다.

실무적으로 보면 이 설계는 "벡터 검색을 Lakehouse에 넣는다"보다 "인덱스도 테이블 수명주기 안에서 관리한다"에 가깝다. 인덱스가 snapshot에 묶이면 time travel 조회에서 어떤 인덱스를 써야 하는지 명확해지고, compaction이나 rewrite 이후 어떤 인덱스를 폐기해야 하는지도 Iceberg maintenance 흐름과 연결할 수 있다. stateless executor가 object storage를 읽는 compute-disaggregated 구조에서는 운영 표면도 작아질 수 있다.

다만 바로 표준 운영 패턴으로 받아들이기는 이르다. 논문은 설계 패턴과 구현 방향을 제시하지만, 엔진별 지원, 표준화 수준, 인덱스 build 비용, recall/latency trade-off, compaction과 refresh 타이밍은 별도 검증이 필요하다. 특히 벡터 검색은 query path에 직접 들어가기 때문에 p95/p99 latency, cold read, SSD cache hit, object storage read amplification 같은 지표를 함께 봐야 한다.

데이터 플랫폼 관점에서 볼 질문은 세 가지다. 첫째, 벡터 인덱스가 원본 테이블 snapshot과 같은 정합성 경계를 공유하는가. 둘째, 인덱스 파일의 생성·갱신·만료가 Iceberg maintenance와 충돌하지 않는가. 셋째, 별도 벡터 저장소의 빠른 기능성과 Iceberg 메타데이터에 붙일 때의 단순한 수명주기 관리 중 어느 쪽이 운영 비용을 줄이는가.

결론적으로 Puffin-backed vector index는 "Iceberg가 벡터 DB를 대체한다"는 주장이라기보다, Lakehouse 테이블 포맷이 데이터뿐 아니라 파생 인덱스의 일관성 경계까지 품을 수 있는지 묻는 시도다. LLM/RAG 파이프라인이 분석 테이블과 가까워질수록, 인덱스를 어디에 저장할지보다 어떤 snapshot과 함께 유효한지 정의하는 일이 더 중요해진다.

## 참고 링크

- [Puffin-Backed Vector Indexes: Attaching Approximate Nearest Neighbor Indexes to Apache Iceberg Snapshots for Compute-Disaggregated Query Engines](https://arxiv.org/abs/2606.04196)
- [Apache Iceberg Specification](https://iceberg.apache.org/spec/)
- [Apache Iceberg Puffin Spec](https://iceberg.apache.org/docs/latest/puffin-spec/)
