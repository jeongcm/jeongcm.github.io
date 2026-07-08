---
title: "Apache Flink는 어떻게 일을 나눠서 실행할까: JobManager와 TaskManager"
date: 2026-07-08 16:00:00 +0900
categories: [technical-knowledge, data-engineering]
tags: [apache-flink, streaming, jobmanager, taskmanager, distributed-systems]
---

Flink를 처음 보면 API보다 먼저 헷갈리는 부분이 있다. 내가 작성한 `map`, `filter`, `keyBy`, `window`, `aggregate` 같은 코드가 실제 클러스터에서는 어디에서 실행되는가 하는 점이다. 이 질문에 답하려면 Flink 런타임을 크게 두 종류의 프로세스로 나눠서 보면 된다. 조율하는 쪽은 **JobManager**, 실제 데이터를 처리하는 쪽은 **TaskManager**다.

내가 했던 주문 스트리밍 작업도 이 관점으로 보면 훨씬 선명해진다. 이 잡은 Kafka 주문 변경 이벤트를 읽고, 입금 전/미결제 취소 상태를 걸러낸 뒤, 중복 제거, 삭제 이벤트 병합, 주문 금액 보정을 거쳐 Iceberg에 적재한다. 코드 한 줄로는 `keyBy(...).process(...)`가 이어지는 것처럼 보이지만, 실제로는 key별 상태를 가진 여러 subtask가 TaskManager slot 위에 분산되고, JobManager가 이 전체 실행 그래프와 checkpoint를 조율한다.

JobManager는 잡의 두뇌에 가깝다. 클라이언트가 제출한 프로그램을 실행 가능한 그래프로 만들고, 어떤 태스크를 언제 스케줄링할지 결정하고, 체크포인트를 조율하고, 실패가 발생했을 때 복구를 지휘한다. JobManager 안에는 리소스 할당을 담당하는 ResourceManager, 잡 제출과 Web UI를 담당하는 Dispatcher, 개별 잡 실행을 관리하는 JobMaster 같은 역할이 있다. 운영 관점에서는 "Flink 잡이 왜 재시작됐는가", "체크포인트가 어디서 막혔는가", "슬롯이 부족해서 대기 중인가" 같은 질문이 JobManager 쪽 신호와 연결된다.

TaskManager는 워커다. 실제 데이터 레코드를 읽고, 연산자를 실행하고, 네트워크로 다음 연산자에게 데이터를 넘긴다. TaskManager는 JVM 프로세스이고, 내부에 여러 개의 task slot을 가진다. slot은 Flink가 병렬 작업을 배치하는 기본 단위다. 병렬도가 8인 잡을 실행한다면, Flink는 그 병렬 subtask들을 사용 가능한 slot 위에 배치한다. 그래서 운영할 때는 CPU 코어 수만 보는 것이 아니라 TaskManager 수, slot 수, parallelism, operator chaining이 같이 맞물린다.

여기서 중요한 개념이 **operator chain**이다. Flink는 모든 연산자를 무조건 별도 스레드로 쪼개지 않는다. 가능한 경우 여러 연산자를 하나의 task로 묶어서 실행한다. 예를 들어 `map -> filter`처럼 shuffle이 필요 없는 연산은 한 task 안에서 이어 붙일 수 있다. 이렇게 하면 스레드 간 전달, 버퍼링, 네트워크 비용이 줄고 지연시간도 낮아진다. 반대로 `keyBy`처럼 데이터를 key 기준으로 재분배해야 하는 지점에서는 네트워크 shuffle이 생기고, 그 뒤의 연산자는 새로운 병렬 subtask 묶음으로 실행된다.

내 작업에서는 이 shuffle 지점이 설계의 경계가 된다. 주문 아이템 중복 제거는 특정 키 기준으로 key를 잡고, 삭제 이벤트 병합은 삭제키 기준으로 다시 key를 잡는다. 주문 금액 보정 역시 특정 키 단위의 MapState를 사용한다. 즉 같은 주문 파이프라인 안에서도 "어떤 비즈니스 충돌을 해결하느냐"에 따라 key의 단위가 달라지고, 그때마다 Flink는 데이터를 다시 나눠 TaskManager의 subtask로 보낸다.

Flink 잡을 그림으로 상상하면 더 쉽다. 사용자가 작성한 코드는 논리적인 dataflow가 되고, Flink는 이를 병렬 실행 그래프로 바꾼다. JobManager는 이 그래프를 보고 "어떤 subtask를 어느 TaskManager slot에 올릴지"를 정한다. TaskManager들은 각자 맡은 subtask를 실행하며 데이터 스트림을 주고받는다. checkpoint barrier가 흐르면 JobManager가 전체 체크포인트 완료 여부를 조율하고, TaskManager들은 각자의 상태를 state backend에 스냅샷으로 남긴다.

이 구조를 이해하면 장애 분석도 덜 추상적이 된다. JobManager 장애는 잡 조율자의 장애다. HA 구성이 되어 있다면 standby JobManager가 leader가 되어 이어받을 수 있다. TaskManager 장애는 실제 연산을 수행하던 워커 일부가 사라진 것이다. 이 경우 해당 TaskManager에서 실행 중이던 subtask들이 실패하고, Flink는 마지막으로 성공한 체크포인트를 기준으로 필요한 태스크를 다시 배치해 복구한다.

실무에서 자주 보는 문제는 "잡이 느리다"라는 한 문장으로 들어오지만, 원인은 서로 다르다. slot이 부족하면 스케줄링이 밀린다. 특정 TaskManager의 CPU나 GC가 튀면 그 워커의 subtask가 병목이 된다. downstream sink가 느려지면 backpressure가 upstream으로 전파된다. key skew가 심하면 일부 subtask만 바쁘고 나머지는 놀게 된다. 이때 JobManager/TaskManager의 역할을 알고 있으면 병목을 어디서 볼지 정리된다.

특히 Iceberg sink까지 붙으면 JobManager/TaskManager를 보는 눈이 더 중요해진다. 내가 한 작업에서는 order item upsert 테이블과 append-only ingest log 테이블을 동시에 썼다. upsert 테이블은 최신 주문 상태를 조회하기 위한 것이고, append log는 같은 상태가 반복 수집되는 과다 호출을 탐지하기 위한 것이다. 이처럼 sink가 여러 개가 되면 checkpoint 완료, Iceberg commit, small file compaction, backpressure가 서로 영향을 준다. 그래서 Flink 운영은 코드보다 실행 그래프를 보는 일이 먼저다.

나는 Flink를 볼 때 다음 순서로 생각하는 편이다.

1. JobManager는 잡의 실행 계획, 스케줄링, 체크포인트, 복구를 조율한다.
2. TaskManager는 실제 연산을 수행하고, slot 단위로 병렬 subtask를 실행한다.
3. shuffle이 없는 연산은 chain으로 묶여 효율적으로 실행될 수 있다.
4. `keyBy`, join, window aggregation처럼 재분배가 필요한 지점은 네트워크와 상태 관리 비용이 커진다.
5. 장애가 나면 "조율자의 문제인가, 워커의 문제인가, 외부 시스템의 문제인가"를 분리해서 본다.

Flink를 잘 운영하려면 API 사용법만으로는 부족하다. 어떤 연산이 JobManager의 조율을 필요로 하고, 어떤 연산이 TaskManager의 CPU와 메모리를 쓰며, 어떤 지점에서 네트워크 shuffle과 state backend 비용이 생기는지 알아야 한다. 결국 Flink의 강점은 코드를 클러스터 전체의 지속 실행 그래프로 바꾸는 데 있고, 그 중심에 JobManager와 TaskManager의 역할 분리가 있다.

## 참고 링크

- [Apache Flink Architecture](https://nightlies.apache.org/flink/flink-docs-release-2.1/docs/concepts/flink-architecture/)
- [Apache Flink Checkpointing](https://nightlies.apache.org/flink/flink-docs-release-2.1/docs/dev/datastream/fault-tolerance/checkpointing/)
