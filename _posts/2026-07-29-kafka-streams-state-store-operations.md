---
title: "Kafka Streams 4.3의 StateStore 변경을 운영 관점에서 보기"
date: 2026-07-29 14:45:00 +0900
categories: [data-engineering]
tags: [Kafka, Kafka Streams, StateStore, RocksDB, stream-processing]
---

Kafka Streams 애플리케이션은 장애 복구와 재시작 과정에서 로컬 상태 저장소를 적극적으로 사용한다. 이 상태는 빠른 처리를 가능하게 하지만, 오래 남은 로컬 디렉터리와 changelog offset 관리 방식이 운영 복잡도를 만들기도 한다. Apache Kafka 4.3 계열의 Streams 변경은 새 기능 자체보다 "상태 저장소를 어떤 기준으로 지우고, 어떤 기준으로 복구할 것인가"라는 운영 판단 문제로 읽는 편이 좋다.

Kafka 4.3.0 릴리스 발표에는 Streams 쪽 변경으로 KIP-1035와 KIP-1259가 포함되어 있다. KIP-1035는 StateStore가 changelog offset을 더 직접적으로 관리할 수 있게 해 복구 기준을 명확히 하려는 변경이고, KIP-1259는 애플리케이션 시작 시 오래된 로컬 state directory를 정리할 수 있는 설정을 추가한다. 둘 다 처리 로직의 표현력보다 재시작, 재배포, 디스크 정리, 복구 시간 같은 운영 품질에 더 직접적으로 연결된다.

<svg viewBox="0 0 760 250" role="img" aria-label="Kafka Streams local state recovery flow" xmlns="http://www.w3.org/2000/svg">
  <rect x="20" y="28" width="150" height="64" rx="8" fill="#eef6ff" stroke="#2f6fab"/>
  <text x="95" y="55" text-anchor="middle" font-size="14" fill="#17324d">Input topic</text>
  <text x="95" y="75" text-anchor="middle" font-size="12" fill="#37516b">events</text>
  <path d="M170 60 H255" stroke="#445" stroke-width="2" marker-end="url(#arrow)"/>
  <rect x="255" y="28" width="180" height="64" rx="8" fill="#f0fff5" stroke="#3a7f55"/>
  <text x="345" y="55" text-anchor="middle" font-size="14" fill="#183d27">Kafka Streams task</text>
  <text x="345" y="75" text-anchor="middle" font-size="12" fill="#365a41">process + update store</text>
  <path d="M345 92 V145" stroke="#445" stroke-width="2" marker-end="url(#arrow)"/>
  <rect x="255" y="145" width="180" height="64" rx="8" fill="#fff7e8" stroke="#a36b16"/>
  <text x="345" y="172" text-anchor="middle" font-size="14" fill="#4b3210">Local StateStore</text>
  <text x="345" y="192" text-anchor="middle" font-size="12" fill="#6b4f24">RocksDB / state.dir</text>
  <path d="M435 177 H530" stroke="#445" stroke-width="2" marker-end="url(#arrow)"/>
  <rect x="530" y="145" width="190" height="64" rx="8" fill="#fff0f3" stroke="#ad4056"/>
  <text x="625" y="172" text-anchor="middle" font-size="14" fill="#5a1b29">Changelog topic</text>
  <text x="625" y="192" text-anchor="middle" font-size="12" fill="#743346">restore boundary</text>
  <path d="M625 145 V72 H435" stroke="#445" stroke-width="2" fill="none" marker-end="url(#arrow)"/>
  <text x="548" y="58" font-size="12" fill="#4a4a4a">restart / rebalance restore</text>
  <defs>
    <marker id="arrow" markerWidth="10" markerHeight="10" refX="7" refY="3" orient="auto">
      <path d="M0,0 L0,6 L8,3 z" fill="#445"/>
    </marker>
  </defs>
</svg>

핵심은 로컬 상태를 "캐시처럼 언제든 지워도 되는 것"으로만 보지 않는 것이다. StateStore는 changelog topic으로 복원할 수 있지만, 저장소가 크거나 복구 지점이 불명확하면 재시작 시간이 길어지고 장애 대응이 느려진다. 반대로 오래된 state directory를 무조건 보존하면 디스크 사용량이 누적되고, 재배포가 잦은 환경에서는 남은 디렉터리가 운영 노이즈가 된다. `state.cleanup.dir.max.age.ms` 같은 설정은 이 균형을 명시적으로 조정하기 위한 장치다.

운영 적용 시에는 세 가지를 먼저 확인해야 한다. 첫째, 애플리케이션의 state directory가 컨테이너 재시작 후에도 유지되는지 확인한다. 둘째, changelog topic의 보존 정책과 store 복구 시간이 서비스 SLO 안에 들어오는지 측정한다. 셋째, rolling deployment나 rebalance가 잦은 환경에서 오래된 task 디렉터리를 언제 지워도 안전한지 기준을 정한다. 상태 저장소가 큰 Streams 애플리케이션일수록 이 설정은 단순 청소 옵션이 아니라 복구 전략의 일부가 된다.

Kafka Streams를 운영할 때는 처리 코드의 DSL보다 로컬 상태의 생명주기를 더 먼저 봐야 할 때가 있다. changelog가 복구의 원천이고 local store가 성능의 원천이라면, 두 경계 사이의 offset, retention, cleanup 정책이 장애 복구 시간을 결정한다. Kafka 4.3의 Streams 변경은 이 지점을 더 명시적으로 다루게 한다는 점에서 의미가 있다.

## 참고 링크

- [Apache Kafka 4.3.0 릴리스 발표](https://kafka.apache.org/blog/2026/06/16/apache-kafka-4.3.0-release-announcement/)
- [KIP-1035: StateStore managed changelog offsets](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1035%3A%2BStateStore%2Bmanaged%2Bchangelog%2Boffsets)
- [KIP-1259: Add configuration to wipe Kafka Streams local state on startup](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1259%3A%2BAdd%2Bconfiguration%2Bto%2Bwipe%2BKafka%2BStreams%2Blocal%2Bstate%2Bon%2Bstartup)
- [Kafka Streams memory management documentation](https://kafka.apache.org/43/streams/developer-guide/memory-mgmt/)
