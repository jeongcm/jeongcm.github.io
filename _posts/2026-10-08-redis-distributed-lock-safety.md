---
title: "Redis 분산 락은 무엇을 보장하는가: SET NX PX부터 fencing token까지"
date: 2026-10-08 07:29:00 +0900
categories: [technical-knowledge, backend-knowledge]
tags: [redis, distributed-lock, concurrency, redlock, fencing-token]
---

Redis 분산 락은 여러 애플리케이션 인스턴스가 같은 작업을 동시에 수행하지 않도록 조정할 때 자주 사용된다. 하지만 `SET NX`로 키를 하나 만들었다고 해서 모든 장애 상황에서 상호 배제가 보장되는 것은 아니다. Redis 락은 유효 시간이 있는 **lease**에 가깝고, 올바른 획득과 해제뿐 아니라 TTL 만료 뒤에도 실행 중인 오래된 작업자를 어떻게 차단할지까지 설계해야 한다.

## 로컬 mutex와 분산 락은 무엇이 다른가

한 프로세스 안의 mutex는 같은 메모리를 공유하는 thread를 조정한다. 반면 분산 환경의 인스턴스들은 서로의 상태를 직접 알지 못한다. 네트워크가 끊기거나 프로세스가 멈추고, 락 저장소의 primary가 교체될 수도 있다. 따라서 분산 락은 최소한 다음 질문에 답해야 한다.

- 같은 자원에 대해 한 시점에 하나의 작업자만 성공하는가
- 락 소유자가 죽어도 다른 작업자가 언젠가 진행할 수 있는가
- 락을 획득하지 않은 작업자가 다른 작업자의 락을 해제할 수 없는가
- 락이 만료된 뒤 재개된 오래된 작업자가 외부 자원을 변경하지 못하게 할 수 있는가

마지막 조건은 Redis 안의 락 키만으로 해결되지 않는다.

## 기본 획득 명령은 `SET NX PX`

Redis 공식 문서가 설명하는 단일 인스턴스 락의 기본 형태는 다음과 같다.

```redis
SET lock:order:42 550e8400-e29b-41d4-a716-446655440000 NX PX 10000
```

- `NX`: 키가 없을 때만 저장한다. 먼저 성공한 작업자만 락을 얻는다.
- `PX 10000`: 10초 뒤 자동 만료시킨다. 소유자가 죽어도 락이 영구히 남는 상황을 막는다.
- UUID 값: 락 요청마다 만든 고유한 소유권 토큰이다.

TTL은 단순한 청소 시간이 아니다. 작업자가 상호 배제를 기대할 수 있는 최대 유효 시간이다. 작업이 TTL 안에 끝나지 않으면 Redis는 키를 만료시키고 다른 작업자의 획득을 허용한다. 따라서 TTL은 정상 처리 시간뿐 아니라 네트워크 지연, JVM GC pause, scheduler 지연을 포함해 정해야 한다.

## `DEL`만 호출하면 다른 작업자의 락을 지울 수 있다

작업자 A가 락을 얻은 뒤 오랫동안 멈췄다고 가정하자. 그사이 TTL이 만료되고 작업자 B가 같은 키로 새 락을 얻을 수 있다. 이때 A가 재개되어 단순히 `DEL lock:order:42`를 실행하면 B의 락을 삭제한다.

따라서 해제는 **현재 저장된 값이 자신이 발급한 토큰과 같은 경우에만** 수행해야 한다. Redis 8.4 이상에서는 조건부 삭제 명령을 사용할 수 있다.

```redis
DELEX lock:order:42 IFEQ 550e8400-e29b-41d4-a716-446655440000
```

Redis 8.4 이전 버전에서는 비교와 삭제를 하나의 원자적 Lua script로 묶는다.

```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
end
return 0
```

`GET`과 `DEL`을 애플리케이션에서 따로 호출하면 두 명령 사이에 락 소유자가 바뀌는 경쟁 조건이 생긴다. Redis script는 실행 전체가 원자적이므로 이 간격을 없앤다.

## 소유권 토큰도 오래된 작업자의 쓰기는 막지 못한다

소유권 토큰은 다른 작업자의 락을 잘못 해제하는 문제를 해결한다. 하지만 락이 만료된 작업자 A가 DB나 외부 API에 쓰는 것까지 막지는 못한다.

<figure class="post-diagram">
  <svg viewBox="0 0 960 500" role="img" aria-labelledby="redis-lock-title redis-lock-desc" xmlns="http://www.w3.org/2000/svg">
    <title id="redis-lock-title">Redis 락 만료 후 오래된 작업자가 다시 쓰기를 수행하는 과정</title>
    <desc id="redis-lock-desc">작업자 A의 일시 정지 중 lease가 만료되고 작업자 B가 새 락으로 쓰기를 완료한 뒤 A가 재개되어 오래된 쓰기를 수행하는 시간 순서도</desc>
    <rect x="20" y="20" width="920" height="450" rx="8" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="480" y="55" text-anchor="middle" font-family="sans-serif" font-size="20" fill="#17202a">락 키가 없어졌다고 이전 소유자의 실행까지 멈추지는 않는다</text>

    <text x="80" y="110" font-family="sans-serif" font-size="15" font-weight="bold" fill="#1f5f8b">Worker A</text>
    <text x="80" y="210" font-family="sans-serif" font-size="15" font-weight="bold" fill="#9a5b12">Redis</text>
    <text x="80" y="310" font-family="sans-serif" font-size="15" font-weight="bold" fill="#2f6f44">Worker B</text>
    <text x="80" y="410" font-family="sans-serif" font-size="15" font-weight="bold" fill="#7a3e65">Database</text>

    <line x1="170" y1="105" x2="900" y2="105" stroke="#8ca6b8" stroke-width="2"/>
    <line x1="170" y1="205" x2="900" y2="205" stroke="#c5a16d" stroke-width="2"/>
    <line x1="170" y1="305" x2="900" y2="305" stroke="#7ca88a" stroke-width="2"/>
    <line x1="170" y1="405" x2="900" y2="405" stroke="#ac7b9b" stroke-width="2"/>

    <circle cx="220" cy="105" r="7" fill="#2878b5"/>
    <path d="M220 112 V195" stroke="#2878b5" stroke-width="2" marker-end="url(#blue-arrow)"/>
    <text x="230" y="145" font-family="sans-serif" font-size="12" fill="#17202a">SET NX PX, token=A</text>
    <rect x="315" y="82" width="150" height="46" rx="5" fill="#e8f3ff" stroke="#2878b5"/>
    <text x="390" y="101" text-anchor="middle" font-family="sans-serif" font-size="12" fill="#17202a">GC pause / 지연</text>
    <text x="390" y="118" text-anchor="middle" font-family="sans-serif" font-size="11" fill="#52606d">TTL보다 오래 멈춤</text>

    <circle cx="485" cy="205" r="7" fill="#c47b16"/>
    <text x="485" y="235" text-anchor="middle" font-family="sans-serif" font-size="12" fill="#8a5511">token=A 만료</text>

    <circle cx="560" cy="305" r="7" fill="#438a58"/>
    <path d="M560 298 V215" stroke="#438a58" stroke-width="2" marker-end="url(#green-arrow)"/>
    <text x="570" y="265" font-family="sans-serif" font-size="12" fill="#17202a">SET NX PX, token=B</text>
    <path d="M650 312 V395" stroke="#438a58" stroke-width="2" marker-end="url(#green-arrow)"/>
    <text x="660" y="355" font-family="sans-serif" font-size="12" fill="#17202a">새 값 쓰기</text>

    <circle cx="790" cy="105" r="7" fill="#b74040"/>
    <path d="M790 112 V395" stroke="#b74040" stroke-width="3" stroke-dasharray="7 5" marker-end="url(#red-arrow)"/>
    <text x="800" y="170" font-family="sans-serif" font-size="12" fill="#9c2f2f">A 재개</text>
    <text x="800" y="190" font-family="sans-serif" font-size="12" fill="#9c2f2f">오래된 값 쓰기</text>
    <rect x="742" y="420" width="150" height="32" rx="5" fill="#fdecec" stroke="#b74040"/>
    <text x="817" y="441" text-anchor="middle" font-family="sans-serif" font-size="12" fill="#9c2f2f">정합성 훼손 가능</text>

    <defs>
      <marker id="blue-arrow" markerWidth="10" markerHeight="10" refX="8" refY="3" orient="auto"><path d="M0,0 L0,6 L9,3 z" fill="#2878b5"/></marker>
      <marker id="green-arrow" markerWidth="10" markerHeight="10" refX="8" refY="3" orient="auto"><path d="M0,0 L0,6 L9,3 z" fill="#438a58"/></marker>
      <marker id="red-arrow" markerWidth="10" markerHeight="10" refX="8" refY="3" orient="auto"><path d="M0,0 L0,6 L9,3 z" fill="#b74040"/></marker>
    </defs>
  </svg>
  <figcaption>직접 작성한 다이어그램: lease 만료 뒤 두 작업자가 외부 자원에 접근하는 상황</figcaption>
</figure>

이 문제에는 fencing token이 필요하다. 락을 획득할 때마다 증가하는 번호를 받고, DB나 storage 같은 최종 자원이 자신이 처리한 가장 큰 번호를 기억하도록 한다.

```text
Worker A: fencing token 41
Worker B: fencing token 42

Database: 42를 먼저 처리했다면 이후 도착한 41의 쓰기를 거부
```

fencing token은 단순 UUID와 역할이 다르다. UUID는 락의 소유자를 식별하지만 순서를 표현하지 않는다. fencing token은 반드시 단조 증가해야 하고, 최종 자원이 토큰을 검사할 수 있어야 한다. Redis primary failover까지 고려한 강한 정확성이 필요하다면 단순 `INCR`만으로 충분하다고 가정하지 말고, 토큰 발급의 지속성과 일관성까지 검증해야 한다.

## Kotlin에서 구성할 때의 최소 경계

아래는 명령의 책임을 보여주기 위한 **개념 예시(pseudocode)**다. 실제 애플리케이션에서는 검증된 Redis client나 lock library를 사용하고 timeout, retry, cancellation 동작을 함께 검증해야 한다.

```kotlin
suspend fun <T> withRedisLease(
    key: String,
    ttlMillis: Long,
    work: suspend () -> T,
): T? {
    val ownerToken = UUID.randomUUID().toString()
    val acquired = redis.setNxPx(key, ownerToken, ttlMillis)

    if (!acquired) return null

    return try {
        work()
    } finally {
        // Redis 8.4+: DELEX key IFEQ ownerToken
        // 이전 버전: token 비교와 DEL을 Lua로 원자 실행
        redis.deleteOnlyIfValueEquals(key, ownerToken)
    }
}
```

이 wrapper가 보장하는 것은 제한된 시간 동안의 획득 경쟁과 안전한 해제다. `work()`가 TTL을 넘기지 않는다는 보장이나, 만료 뒤 외부 쓰기를 막는 보장은 별도다. 실행 시간이 길다면 제한된 횟수의 lease 연장과 fencing token을 함께 검토해야 한다. 무제한 연장은 다른 작업자가 영원히 진행하지 못하는 liveness 문제를 만들 수 있다.

## 복제와 failover가 락을 자동으로 안전하게 만들지는 않는다

Redis replication은 기본적으로 비동기다. 작업자 A가 primary에 락 키를 저장한 직후, replica에 복제되기 전에 primary가 죽으면 새 primary에는 락 키가 없을 수 있다. 그러면 작업자 B도 같은 락을 얻어 상호 배제가 깨진다.

`WAIT`는 쓰기가 지정한 수의 replica에 전달될 때까지 기다려 데이터 유실 가능성을 줄인다. 그러나 Redis 공식 문서는 `WAIT`가 Redis를 강한 일관성을 가진 저장소로 바꾸지는 않으며, failover 과정에서 확인된 쓰기도 유실될 수 있다고 명시한다. 따라서 고가용성 구성을 붙였다는 이유만으로 락의 safety가 강해졌다고 단정하면 안 된다.

## Redlock은 어떤 문제를 풀려고 하는가

Redlock은 서로 독립적인 Redis master N개에서 과반수의 락을 제한 시간 안에 얻는 방식이다. 공식 설명의 N=5 예에서는 최소 3개에서 성공해야 하고, 전체 획득 시간이 TTL보다 짧아야 한다. 실제 유효 시간은 대략 다음과 같이 줄어든다.

```text
남은 유효 시간 = TTL - 획득에 사용한 시간 - clock drift 여유
```

Redlock은 단일 primary와 비동기 replica failover 문제를 그대로 사용하는 것보다 더 강한 모델을 목표로 한다. 다만 clock이 비슷한 속도로 흐르고, 작업이 유효 시간 안에 끝난다는 전제가 있다. Redis 공식 문서도 correctness가 중요하면 fencing token을 구현하고, Redis TTL이 monotonic clock을 사용하지 않아 wall-clock 변경이 상호 배제를 훼손할 수 있음을 고려하라고 안내한다.

따라서 Redlock을 선택했다는 사실만으로 외부 시스템의 오래된 쓰기가 차단되지는 않는다. 구현 library의 과반수 획득, timeout, 부분 획득 해제, random jitter retry, lease 연장 제한을 검증하고 최종 자원의 fencing 지원까지 확인해야 한다.

## Redis 분산 락이 적합한 경우

다음처럼 중복 실행을 줄이는 것이 주목적이고, 작업 자체가 멱등적이거나 중복을 복구할 수 있다면 Redis 락은 실용적인 선택이다.

- 여러 인스턴스 중 하나만 수행하면 되는 cache refresh
- 중복되어도 결과를 다시 계산하거나 덮어쓸 수 있는 scheduled job
- 동일 메시지의 동시 처리를 줄이되 별도의 idempotency key가 있는 작업
- 짧고 실행 시간의 상한을 예측할 수 있는 임계 구역

반대로 다음 상황에서는 Redis 락 하나만 정확성의 마지막 방어선으로 두기 어렵다.

- 결제, 재고 차감처럼 중복 쓰기가 금전적·비가역적 결과를 만드는 경우
- 실행 시간이 길거나 외부 API 때문에 상한을 정하기 어려운 작업
- strict leader election이나 단 한 번의 실행이 반드시 필요한 제어 작업
- 락 만료 뒤 도착한 요청을 최종 저장소가 구분할 수 없는 경우

보호할 데이터가 이미 관계형 DB에 있다면 unique constraint, transaction, row lock, PostgreSQL advisory lock 같은 DB 내부 수단이 더 단순할 수 있다. 강한 coordination이 핵심이면 consensus 기반의 etcd나 ZooKeeper 계열을 검토할 수 있다. 구조적으로 가능하다면 Kafka partition처럼 key별 single writer를 만들거나, 명령에 idempotency key를 부여하는 방식이 락 자체를 없애기도 한다.

## 운영 전 확인할 체크리스트

1. 락 키의 단위가 실제 경쟁 자원과 정확히 일치하는가
2. 모든 획득 요청에 재사용하지 않는 owner token을 발급하는가
3. 획득과 TTL 설정이 하나의 `SET NX PX`로 원자 실행되는가
4. 해제가 `DELEX IFEQ` 또는 원자적 Lua 비교·삭제로 수행되는가
5. TTL이 p99 작업 시간, GC pause, 네트워크 지연보다 충분히 큰가
6. 획득 timeout과 random jitter retry가 무한 대기를 막는가
7. lease 연장 횟수와 전체 실행 deadline에 상한이 있는가
8. failover 때 중복 소유를 허용할 수 있는 업무인가
9. 오래된 작업자의 쓰기를 fencing token이나 DB 제약조건이 거부하는가
10. 획득 실패율, 대기 시간, TTL 만료, 연장 실패, 실행 시간 초과를 관측하는가

Redis 분산 락의 핵심은 명령 하나가 아니라 보장 범위를 정확히 정하는 데 있다. `SET NX PX`와 소유자 검증 해제는 기본이며, lease 만료와 failover 뒤에도 업무 정합성을 지켜야 한다면 fencing token, 멱등성, 최종 저장소의 제약조건을 함께 설계해야 한다.

## 참고 링크

- [Redis Documentation: Distributed Locks with Redis](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/)
- [Redis Command: SET](https://redis.io/docs/latest/commands/set/)
- [Redis Command: DELEX](https://redis.io/docs/latest/commands/delex/)
- [Redis Documentation: Scripting with Lua](https://redis.io/docs/latest/develop/programmability/eval-intro/)
- [Redis Command: WAIT](https://redis.io/docs/latest/commands/wait/)
- [Redis Documentation: Replication](https://redis.io/docs/latest/operate/oss_and_stack/management/replication/)
- [Martin Kleppmann: How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
- [etcd API Reference: Concurrency](https://etcd.io/docs/v3.6/dev-guide/api_concurrency_reference_v3/)
- [PostgreSQL Documentation: Advisory Locks](https://www.postgresql.org/docs/current/explicit-locking.html#ADVISORY-LOCKS)
