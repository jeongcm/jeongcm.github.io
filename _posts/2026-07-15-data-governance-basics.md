---
title: "데이터 거버넌스는 무엇이고 어떻게 활용해야 할까"
date: 2026-07-15 14:43:36 +0900
categories: [data-engineering]
tags: [data-governance, data-catalog, data-lineage, data-quality, data-platform]
---

데이터 거버넌스는 데이터를 잠그는 규칙이 아니라, 조직이 데이터를 믿고 찾고 안전하게 쓸 수 있도록 역할, 기준, 절차를 정하는 운영 체계다. AWS는 데이터 거버넌스를 데이터가 비즈니스 운영과 의사결정을 지원할 수 있는 상태가 되도록 프로세스와 정책을 두는 활동으로 설명한다. 핵심은 "누가 어떤 데이터를 어떤 조건에서 어떤 목적으로 사용할 수 있는가"를 반복 가능하게 만드는 것이다.

데이터 플랫폼이 작을 때는 담당자에게 물어보는 방식으로도 충분해 보인다. 하지만 테이블이 늘고, 배치와 스트리밍 파이프라인이 섞이고, 여러 팀이 같은 지표를 다르게 계산하기 시작하면 문제가 달라진다. 어떤 테이블이 정본인지, 컬럼 의미가 무엇인지, 개인정보가 포함됐는지, 장애가 어떤 리포트와 모델에 영향을 주는지 빠르게 판단하기 어려워진다. 데이터 거버넌스는 이 판단을 개인 기억이 아니라 시스템과 프로세스에 남기는 일이다.

<svg viewBox="0 0 760 260" role="img" aria-label="Data governance operating loop" xmlns="http://www.w3.org/2000/svg">
  <rect x="30" y="34" width="130" height="72" rx="8" fill="#e8f3ff" stroke="#3178c6"/>
  <text x="95" y="64" text-anchor="middle" font-size="15" font-family="sans-serif">Catalog</text>
  <text x="95" y="86" text-anchor="middle" font-size="12" font-family="sans-serif">find assets</text>
  <rect x="214" y="34" width="130" height="72" rx="8" fill="#f0f7e8" stroke="#6a994e"/>
  <text x="279" y="64" text-anchor="middle" font-size="15" font-family="sans-serif">Ownership</text>
  <text x="279" y="86" text-anchor="middle" font-size="12" font-family="sans-serif">who decides</text>
  <rect x="398" y="34" width="130" height="72" rx="8" fill="#fff2d8" stroke="#d08c00"/>
  <text x="463" y="64" text-anchor="middle" font-size="15" font-family="sans-serif">Policy</text>
  <text x="463" y="86" text-anchor="middle" font-size="12" font-family="sans-serif">access rules</text>
  <rect x="582" y="34" width="130" height="72" rx="8" fill="#ffe8e8" stroke="#c0392b"/>
  <text x="647" y="64" text-anchor="middle" font-size="15" font-family="sans-serif">Quality</text>
  <text x="647" y="86" text-anchor="middle" font-size="12" font-family="sans-serif">trust checks</text>
  <rect x="214" y="158" width="130" height="72" rx="8" fill="#f7e8ff" stroke="#8e44ad"/>
  <text x="279" y="188" text-anchor="middle" font-size="15" font-family="sans-serif">Lineage</text>
  <text x="279" y="210" text-anchor="middle" font-size="12" font-family="sans-serif">impact path</text>
  <rect x="398" y="158" width="130" height="72" rx="8" fill="#e8fbf6" stroke="#148f77"/>
  <text x="463" y="188" text-anchor="middle" font-size="15" font-family="sans-serif">Audit</text>
  <text x="463" y="210" text-anchor="middle" font-size="12" font-family="sans-serif">evidence</text>
  <path d="M160 70 H214" stroke="#555" stroke-width="2" marker-end="url(#arrow)"/>
  <path d="M344 70 H398" stroke="#555" stroke-width="2" marker-end="url(#arrow)"/>
  <path d="M528 70 H582" stroke="#555" stroke-width="2" marker-end="url(#arrow)"/>
  <path d="M647 106 C647 148 528 172 528 194" fill="none" stroke="#555" stroke-width="2" marker-end="url(#arrow)"/>
  <path d="M398 194 H344" stroke="#555" stroke-width="2" marker-end="url(#arrow)"/>
  <path d="M214 194 C110 194 95 130 95 106" fill="none" stroke="#555" stroke-width="2" marker-end="url(#arrow)"/>
  <defs>
    <marker id="arrow" markerWidth="10" markerHeight="10" refX="8" refY="3" orient="auto">
      <path d="M0,0 L0,6 L9,3 z" fill="#555"/>
    </marker>
  </defs>
</svg>

기본 구성 요소는 다섯 가지로 볼 수 있다. 첫째, 데이터 카탈로그는 테이블, 토픽, 대시보드, 모델 피처 같은 데이터 자산을 찾고 설명하는 공간이다. 둘째, 오너십은 데이터 정의와 접근 승인, 품질 기준을 누가 책임지는지 정한다. 셋째, 데이터 품질은 신선도, 완전성, 중복, null 비율, 값 범위 같은 신뢰 조건을 측정한다. 넷째, lineage는 데이터가 어디서 와서 어떤 파이프라인과 산출물로 흘러가는지 보여준다. 다섯째, 접근 제어와 감사 로그는 민감 데이터 사용을 추적하고 설명할 수 있게 한다.

활용은 큰 프로그램을 먼저 만드는 방식보다, 중요한 데이터 도메인 하나를 골라 작은 루프로 시작하는 편이 낫다. 예를 들어 주문, 결제, 고객, 상품처럼 여러 팀이 반복해서 쓰는 도메인을 선택한다. 그다음 핵심 테이블과 토픽을 카탈로그에 등록하고, 정본 여부와 설명을 붙이며, 데이터 오너와 스튜어드를 지정한다. 이어서 freshness, row count, null ratio 같은 최소 품질 규칙을 붙이고, upstream/downstream lineage를 연결한다. 마지막으로 접근 요청, 승인, 사용 로그를 남겨 문제가 생겼을 때 근거를 추적할 수 있게 한다.

중요한 점은 거버넌스를 문서 프로젝트로 끝내지 않는 것이다. 문서만 있고 파이프라인 배포, 스키마 변경, 대시보드 생성, 권한 신청 흐름과 연결되지 않으면 금방 낡는다. 실무에서는 메타데이터 수집을 자동화하고, CI나 오케스트레이터에서 품질 검사를 실행하며, lineage 이벤트를 파이프라인 실행 시점에 같이 남기는 방식이 더 지속 가능하다. OpenLineage 같은 표준은 job, run, dataset, facet 개념으로 실행 메타데이터를 남겨 lineage를 시스템 간에 교환할 수 있게 한다.

데이터 거버넌스가 과해지는 신호도 있다. 모든 테이블에 동일한 승인 절차를 강제하거나, 실험용 데이터까지 정본 데이터와 같은 수준으로 통제하면 속도가 떨어진다. 반대로 매출, 정산, 개인정보, ML 학습 데이터처럼 영향 범위가 큰 자산은 더 엄격해야 한다. 따라서 데이터 자산을 중요도와 민감도 기준으로 등급화하고, 등급별로 카탈로그 필수 항목, 품질 규칙, 접근 승인, 보존 기간을 다르게 두는 것이 현실적이다.

정리하면 데이터 거버넌스는 "데이터를 못 쓰게 하는 규칙"이 아니라 "데이터를 책임 있게 더 잘 쓰게 하는 운영 장치"다. 좋은 거버넌스는 사용자가 필요한 데이터를 빨리 찾게 하고, 엔지니어가 변경 영향을 예측하게 하며, 보안/컴플라이언스 팀이 사용 근거를 확인하게 한다. 데이터 플랫폼이 커질수록 거버넌스는 별도 행정 업무가 아니라 카탈로그, 파이프라인, 품질 검사, 권한 시스템에 녹아 있어야 한다.

## 참고 링크

- [AWS: What is Data Governance?](https://aws.amazon.com/what-is/data-governance/)
- [OpenLineage specification](https://openlineage.io/docs/spec/)
- [DataHub documentation](https://docs.datahub.com/docs/)
- [Google Cloud: Data governance architecture](https://cloud.google.com/architecture/data-governance)
