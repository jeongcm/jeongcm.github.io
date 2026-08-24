---
title: "Snowflake와 Apache Iceberg는 무엇이 다를까: 웨어하우스와 오픈 테이블 포맷 비교"
date: 2026-08-24 11:08:21 +0900
categories: [data-engineering]
tags: [snowflake, apache-iceberg, lakehouse, data-warehouse, table-format]
---

Snowflake와 Apache Iceberg는 둘 다 대규모 분석 데이터를 다룰 때 자주 등장하지만, 같은 층의 기술이 아니다. Snowflake는 저장, 컴퓨트, 쿼리 최적화, 보안, 거버넌스, 운영 기능을 함께 제공하는 관리형 데이터 플랫폼이다. 반면 Iceberg는 object storage나 분산 파일 시스템 위의 데이터 파일을 하나의 테이블처럼 다루기 위한 오픈 테이블 포맷이다. 비교의 출발점은 "어느 쪽이 더 좋은가"가 아니라, 무엇을 소유하고 무엇을 위임할 것인가다.

Snowflake의 기본 모델은 데이터 웨어하우스에 가깝다. 사용자는 SQL로 테이블을 만들고 조회하며, Snowflake는 자체적으로 데이터를 컬럼 기반 micro-partition으로 나누고 각 partition의 통계를 관리한다. 공식 문서는 micro-partition이 자동으로 만들어지고, 값 범위나 distinct count 같은 메타데이터를 통해 pruning과 최적화에 활용된다고 설명한다. 사용자는 파일 배치, manifest, compaction 같은 저수준 운영을 직접 다루기보다 Snowflake 플랫폼이 제공하는 성능과 관리 기능을 사용한다.

Iceberg는 다른 문제에서 출발한다. 데이터 파일은 Parquet, Avro, ORC 같은 형식으로 object storage에 있고, 여러 엔진이 같은 테이블을 읽고 쓰고 싶다. 단순히 디렉터리 경로를 테이블처럼 쓰면 schema evolution, partition evolution, snapshot, delete file, concurrent write를 안정적으로 관리하기 어렵다. Iceberg는 metadata file, manifest list, manifest, snapshot을 통해 "현재 테이블 상태가 어떤 파일 집합인가"를 명시적으로 기록한다. 모든 변경은 새 metadata file을 만들고 현재 metadata pointer를 atomic하게 교체하는 방식으로 반영된다.

<svg viewBox="0 0 860 360" role="img" aria-label="Snowflake and Iceberg architecture comparison" xmlns="http://www.w3.org/2000/svg">
  <rect x="38" y="42" width="330" height="250" rx="10" fill="#e8f3ff" stroke="#3178c6"/>
  <text x="203" y="76" text-anchor="middle" font-size="18" font-family="sans-serif">Snowflake</text>
  <rect x="78" y="108" width="250" height="48" rx="7" fill="#ffffff" stroke="#6b9fd6"/>
  <text x="203" y="138" text-anchor="middle" font-size="13" font-family="sans-serif">Managed SQL platform</text>
  <rect x="78" y="174" width="250" height="48" rx="7" fill="#ffffff" stroke="#6b9fd6"/>
  <text x="203" y="204" text-anchor="middle" font-size="13" font-family="sans-serif">Micro-partitions and pruning</text>
  <rect x="78" y="240" width="250" height="48" rx="7" fill="#ffffff" stroke="#6b9fd6"/>
  <text x="203" y="270" text-anchor="middle" font-size="13" font-family="sans-serif">Platform governance and operations</text>

  <rect x="492" y="42" width="330" height="250" rx="10" fill="#f2f7e8" stroke="#6a994e"/>
  <text x="657" y="76" text-anchor="middle" font-size="18" font-family="sans-serif">Apache Iceberg</text>
  <rect x="532" y="108" width="250" height="48" rx="7" fill="#ffffff" stroke="#8ab36f"/>
  <text x="657" y="138" text-anchor="middle" font-size="13" font-family="sans-serif">Open table format</text>
  <rect x="532" y="174" width="250" height="48" rx="7" fill="#ffffff" stroke="#8ab36f"/>
  <text x="657" y="204" text-anchor="middle" font-size="13" font-family="sans-serif">Snapshots, manifests, metadata</text>
  <rect x="532" y="240" width="250" height="48" rx="7" fill="#ffffff" stroke="#8ab36f"/>
  <text x="657" y="270" text-anchor="middle" font-size="13" font-family="sans-serif">Multi-engine access on object storage</text>

  <path d="M368 168 H492" stroke="#555" stroke-width="2" marker-end="url(#arrow)"/>
  <text x="430" y="152" text-anchor="middle" font-size="12" font-family="sans-serif">can meet via</text>
  <text x="430" y="188" text-anchor="middle" font-size="12" font-family="sans-serif">Snowflake Iceberg tables</text>
  <defs>
    <marker id="arrow" markerWidth="10" markerHeight="10" refX="8" refY="3" orient="auto">
      <path d="M0,0 L0,6 L9,3 z" fill="#555"/>
    </marker>
  </defs>
</svg>

핵심 차이를 표로 정리하면 아래와 같다.

| 관점 | Snowflake | Apache Iceberg |
|---|---|---|
| 기술 층 | 관리형 데이터 플랫폼/웨어하우스 | 오픈 테이블 포맷 |
| 저장 모델 | Snowflake 관리 저장소 중심, Iceberg table 옵션도 제공 | 사용자가 관리하는 object storage 또는 파일 시스템 위 metadata layer |
| 성능 책임 | 플랫폼이 micro-partition, pruning, 최적화 상당 부분을 담당 | 엔진, catalog, 파일 크기, compaction, metadata 관리 조합에 좌우 |
| 상호운용성 | Snowflake 생태계 안에서 강함, Iceberg 지원으로 외부 lake와 연결 | Spark, Flink, Trino, Snowflake 등 여러 엔진이 같은 테이블을 공유 가능 |
| 운영 복잡도 | 낮은 편. 대신 플랫폼 종속성과 비용 모델을 받아들임 | 높은 편. 대신 저장소/엔진 선택권과 개방성이 큼 |

Snowflake를 선택하기 좋은 경우는 분석 사용자에게 안정적인 SQL 환경, 권한 관리, 성능 최적화, 운영 자동화를 빠르게 제공해야 할 때다. 데이터 팀이 파일 레이아웃, catalog 운영, compaction, engine compatibility를 직접 관리하고 싶지 않다면 Snowflake의 관리형 경계가 장점이다. 특히 BI, ad hoc query, governance, sharing, warehouse 운영이 중심이면 플랫폼 완성도가 생산성을 크게 좌우한다.

Iceberg를 선택하기 좋은 경우는 여러 처리 엔진이 같은 lakehouse 테이블을 읽고 써야 할 때다. 예를 들어 Flink가 실시간으로 쓰고, Spark가 배치 보정을 하고, Trino가 ad hoc query를 수행하며, Snowflake가 일부 분석 워크로드를 읽는 구조라면 데이터의 정본을 특정 엔진 전용 저장소에만 두기 어렵다. Iceberg는 테이블 상태를 열린 metadata로 관리하므로 engine interoperability와 storage ownership을 더 강하게 가져갈 수 있다.

흥미로운 지점은 두 기술이 이제 대립만 하지 않는다는 점이다. Snowflake는 Apache Iceberg table을 지원하고, Snowflake를 Iceberg catalog로 쓰거나 외부 REST catalog를 통해 Iceberg table을 조회하는 경로를 제공한다. Snowflake Open Catalog도 Iceberg REST protocol 기반의 관리형 catalog로 설명된다. 즉 "Snowflake냐 Iceberg냐"가 아니라, Snowflake를 query/governance platform으로 쓰면서 Iceberg를 lakehouse table boundary로 쓰는 혼합 구조도 가능하다.

다만 혼합 구조는 공짜가 아니다. Iceberg table을 Snowflake에서 쓴다고 해서 모든 엔진의 쓰기 정책, file size, compaction, snapshot expiration, delete file 처리, 권한 모델이 자동으로 통일되지는 않는다. Snowflake-managed Iceberg table인지, external catalog Iceberg table인지, storage를 누가 관리하는지, 다른 엔진도 write하는지에 따라 책임 경계가 달라진다. 이 경계를 흐리면 "열린 포맷"과 "관리형 플랫폼"의 장점을 모두 얻는 대신, 양쪽 운영 문제를 같이 떠안을 수 있다.

결론적으로 Snowflake는 데이터를 사용하는 경험을 플랫폼으로 단순화하는 선택이고, Iceberg는 데이터를 저장하고 공유하는 테이블 경계를 개방적으로 정의하는 선택이다. 조직이 원하는 것이 빠른 SQL 분석 환경과 관리형 운영이면 Snowflake 중심이 자연스럽다. 반대로 여러 엔진, 장기 보관, object storage 소유권, vendor-neutral table state가 중요하면 Iceberg 중심 lakehouse가 더 잘 맞는다. 실무 설계에서는 둘을 경쟁 제품으로만 보지 말고, query platform, catalog, storage, writer engine의 책임을 나눠서 판단해야 한다.

## 참고 링크

- [Snowflake Documentation: Apache Iceberg tables](https://docs.snowflake.com/en/user-guide/tables-iceberg)
- [Snowflake Documentation: Micro-partitions and Data Clustering](https://docs.snowflake.com/en/user-guide/tables-clustering-micropartitions)
- [Snowflake Documentation: Snowflake Open Catalog overview](https://docs.snowflake.com/en/user-guide/opencatalog/overview)
- [Apache Iceberg Table Spec](https://iceberg.apache.org/spec/)
- [Snowflake SQL Reference: CREATE ICEBERG TABLE with Snowflake catalog](https://docs.snowflake.com/en/sql-reference/sql/create-iceberg-table-snowflake)
- [Snowflake SQL Reference: CREATE ICEBERG TABLE with REST catalog](https://docs.snowflake.com/en/sql-reference/sql/create-iceberg-table-rest)
