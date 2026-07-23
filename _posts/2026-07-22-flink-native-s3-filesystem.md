---
title: "Flink Native S3 FileSystem이 바꾸는 체크포인트 운영 기준"
date: 2026-07-22 22:41:55 +0900
categories: [data-engineering]
tags: [apache-flink, s3, checkpoint, state-backend, data-platform]
---

Apache Flink 2.3.0은 SQL과 Materialized Table 개선뿐 아니라 `flink-s3-fs-native`라는 새로운 S3 파일시스템 플러그인을 실험 기능으로 포함했다. 이 변화는 단순한 커넥터 추가가 아니라, S3 위에서 체크포인트와 세이브포인트를 운영하는 방식의 기본 전제를 바꾼다.

기존 Flink S3 구성은 보통 Hadoop 기반 플러그인이나 Presto 기반 플러그인 중 하나를 고르는 문제였다. Hadoop 플러그인은 `RecoverableWriter`를 지원해 exactly-once 파일 싱크에 유리하지만 의존성이 크고 AWS SDK v1에 묶여 있었다. Presto 플러그인은 체크포인트 읽기 경로에서 자주 선택됐지만 exactly-once 파일 싱크와는 맞지 않는 제약이 있었다. Native S3 FileSystem은 AWS SDK v2 기반으로 Flink 전용 파일시스템을 구현해 이 선택지를 하나로 줄이려는 시도다.

<svg viewBox="0 0 760 270" role="img" aria-label="Flink S3 filesystem plugin comparison" xmlns="http://www.w3.org/2000/svg">
  <rect x="20" y="24" width="720" height="222" rx="12" fill="#f8fafc" stroke="#cbd5e1"/>
  <text x="380" y="58" text-anchor="middle" font-size="22" font-family="sans-serif" fill="#0f172a">Flink에서 S3 플러그인을 고르는 기준</text>
  <rect x="56" y="94" width="190" height="104" rx="8" fill="#e0f2fe" stroke="#0284c7"/>
  <text x="151" y="126" text-anchor="middle" font-size="17" font-family="sans-serif" fill="#075985">Hadoop S3A</text>
  <text x="151" y="156" text-anchor="middle" font-size="13" font-family="sans-serif" fill="#0f172a">exactly-once sink 가능</text>
  <text x="151" y="178" text-anchor="middle" font-size="13" font-family="sans-serif" fill="#0f172a">Hadoop/AWS SDK v1 의존</text>
  <rect x="286" y="94" width="190" height="104" rx="8" fill="#fee2e2" stroke="#dc2626"/>
  <text x="381" y="126" text-anchor="middle" font-size="17" font-family="sans-serif" fill="#991b1b">Presto S3</text>
  <text x="381" y="156" text-anchor="middle" font-size="13" font-family="sans-serif" fill="#0f172a">체크포인트 성능 장점</text>
  <text x="381" y="178" text-anchor="middle" font-size="13" font-family="sans-serif" fill="#0f172a">RecoverableWriter 미지원</text>
  <rect x="516" y="94" width="190" height="104" rx="8" fill="#dcfce7" stroke="#16a34a"/>
  <text x="611" y="126" text-anchor="middle" font-size="17" font-family="sans-serif" fill="#166534">Native S3</text>
  <text x="611" y="156" text-anchor="middle" font-size="13" font-family="sans-serif" fill="#0f172a">AWS SDK v2 / async I/O</text>
  <text x="611" y="178" text-anchor="middle" font-size="13" font-family="sans-serif" fill="#0f172a">체크포인트와 sink 통합</text>
  <text x="380" y="226" text-anchor="middle" font-size="14" font-family="sans-serif" fill="#334155">운영 판단: 기능 호환성, 체크포인트 시간, 의존성/CVE 부담, 롤백 경로를 함께 본다</text>
</svg>

핵심 개념은 세 가지다. 첫째, 새 플러그인은 `s3://`와 `s3a://` 스킴을 등록하면서 `s3.*` 설정 네임스페이스를 사용한다. Hadoop 설정을 Flink 설정으로 우회 매핑하던 복잡도를 줄이는 방향이다. 둘째, AWS SDK v2의 비동기 I/O 경로를 사용한다. 공식 벤치마크는 특정 EKS 환경과 RocksDB 상태 조건에서 Presto 플러그인 대비 평균 처리량과 체크포인트 시간이 개선됐다고 설명한다. 셋째, `FileSystem`과 `RecoverableWriter`를 하나의 플러그인에서 제공해 체크포인트와 exactly-once 파일 싱크를 별도 선택 문제로 분리하지 않는다.

실무적으로 중요한 지점은 성능 수치 자체보다 운영 리스크의 모양이다. 체크포인트 시간이 줄면 장애 후 재처리해야 할 데이터 범위가 작아지고, 체크포인트 주기를 더 공격적으로 잡을 여지가 생긴다. 동시에 Hadoop 의존성을 제거하면 보안 취약점 대응, shading 충돌, 라이브러리 버전 관리의 표면적도 줄어든다. 특히 AWS SDK v1 기반 구성이 수명 종료 이후의 유지보수 리스크를 갖는다는 점은 플랫폼 팀의 업그레이드 판단에 직접 영향을 준다.

다만 Flink 2.3의 Native S3 FileSystem은 아직 experimental opt-in이다. 새 클러스터의 기본값처럼 즉시 간주하기보다, staging에서 기존 Hadoop 또는 Presto 플러그인과 같은 체크포인트/세이브포인트를 읽고 쓰는지 검증해야 한다. 공식 글은 기존 플러그인과 side-by-side 배치 및 우선순위 설정을 통한 단계적 전환을 설명하므로, 운영 전환은 "JAR 교체"가 아니라 성능, 복구, 암호화, IAM, 롤백 절차를 함께 확인하는 배포 변경으로 보는 편이 안전하다.

적용 기준은 다음처럼 잡을 수 있다. S3 체크포인트 시간이 복구 SLA나 처리량에 영향을 주고, Hadoop 의존성 관리 비용이 큰 Flink 플랫폼이라면 우선 검토할 가치가 높다. 반대로 현재 플러그인이 안정적으로 운영되고 있고 체크포인트 병목이 명확하지 않다면, Flink 2.3 업그레이드와 함께 별도 성능 검증 항목으로 두는 것이 적절하다.

## 참고 링크

- [Apache Flink 2.3.0 Release Announcement](https://flink.apache.org/2026/06/25/apache-flink-2.3.0-release-announcement/)
- [Introducing Flink's Native S3 FileSystem: Built for Performance, Designed for Production](https://flink.apache.org/2026/06/26/announcing-native-s3-fs/)
- [Apache Flink Downloads](https://flink.apache.org/downloads/)
