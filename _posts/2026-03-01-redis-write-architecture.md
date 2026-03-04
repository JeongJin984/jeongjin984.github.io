---
title: Redis 캐시 쓰기 관점 아키텍처
description: >-
  Redis 캐시 쓰기 관점 아키텍처 설계
author: jay
date: 2026-03-01 20:55:00 +0800
categories: [Study, Redis]
tags: [redis, study]
pin: false
media_subpath: ''
---

# Redis 쓰기 캐시 안정화 아키텍처

## 1. Write Path의 특성

쓰기(Write) 경로는 읽기와 구조적으로 다르다. 대부분 시스템에서 다음 특성을 가진다.

1. **QPS는 낮지만 영향은 크다**
   - 읽기보다 빈도는 낮지만 데이터 정합성과 사용자 상태에 직접 영향을 준다.
2. **정합성 민감**
   - stale 데이터는 허용될 수 있지만, 잘못된 데이터는 허용되지 않는다.
3. **동시성 충돌 가능성**
   - 동일 키 또는 동일 엔티티에 대해 동시 업데이트가 발생한다.
4. **DB가 Source of Truth**
   - 대부분 시스템에서 Redis는 보조 캐시이며, DB가 최종 데이터 원천이다.

읽기 캐시는 latency 안정화가 목표라면, 쓰기 캐시는 다음이 목표다.

- 정합성 유지
- DB write amplification 방지
- cache ↔ DB 불일치 최소화
- write contention 제어

## 2. Write Path의 비용 구조

쓰기 비용은 단순히 DB UPDATE 한 번으로 끝나지 않는다. 실제로는 아래와 같은 복합 비용이 발생한다.

### 2.1 Write Amplification

한 번의 쓰기가 여러 계층으로 전파된다.

```text
Client
  ↓
App
  ↓
DB write
  ↓
Cache invalidate / update
  ↓
Downstream service
```

쓰기 경로에서 문제는 다음과 같이 나타난다.

1. DB Lock 경합 
2. cache invalidation race 
3. replication lag 
4. eventual consistency

### 2.2 Cache Inconsistency Window

쓰기에서 가장 중요한 문제는 다음이다.

> DB와 Cache 사이의 불일치 시간(inconsistency window)

예시: t1 이전에 read가 들어오면 stale cache가 반환된다.
```text
t0  DB update
t1  cache invalidate
t2  read request
```

### 2.3 Write Contention

핫 엔티티(예: 인기 상품, 계좌, 재고)에 대한 쓰기는 다음을 유발한다.

- DB row lock 경쟁
- cache thrashing
- replication lag
- latency 증가

## 3. 기본 쓰기 캐시 패턴

### 3.1 Write Through

> write → cache → DB

```text
Client
  ↓
Cache write
  ↓
DB write
```

**장점**

- cache와 DB 동기화 유지
- read path 단순

**단점**

- write latency 증가
- cache 장애 시 write 실패 가능

### 3.2 Write Around

> write → DB → cache bypass

```text
Client
  ↓
DB write
  ↓
cache invalidate
```

**장점**

- write latency 낮음
- cache 오염 방지

**단점**

- write 직후 read miss 발생 가능

### 3.3 Write Back (Write Behind)

> write → cache → async DB flush

```text
Client
  ↓
Cache write
  ↓
Async DB flush
```

**장점**

1. write latency 최소화 
2. DB write batching 가능

**단점**

- 데이터 유실 가능 
- crash recovery 복잡
- 실무에서는 대부분 Write Around + Cache Aside 조합을 사용한다.

## 4. 쓰기 캐시의 핵심 문제

쓰기 캐시는 다음 3가지 문제가 핵심이다.

1. Cache Invalidation Race
2. Lost Update
3. Hot Key Write Contention

## 5. Cache Invalidation Race

다음 시나리오는 매우 흔하다.

```text
t0  read request
t1  cache hit (old value)
t2  write update
t3  cache invalidate
t4  read response (old value 반환)
```

즉 update 이후에도 stale read가 발생한다.

이를 Read-Write Race라고 한다.

## 6. Cache Invalidation 전략

캐시 무효화 전략은 크게 3가지가 있다.

| 전략     | 방식             | 특징    |
| ------ | -------------- | ----- |
| Delete | update 후 캐시 삭제 | 가장 안전 |
| Update | cache 값 갱신     | 복잡    |
| TTL 의존 | 만료 대기          | 위험    |

실무에서 가장 안전한 방식은 다음이다.

> Write → DB commit → cache delete

## 7. Double Delete 패턴

단순 delete만으로는 race를 완전히 제거할 수 없다.

그래서 사용하는 것이 Double Delete이다.

동작

```text
1. cache delete
2. DB update
3. sleep (수 ms)
4. cache delete again
```

목적 : write 직전에 발생한 read가 cache를 다시 채우는 것을 제거

동작 흐름 : 두 번째 delete가 stale cache를 제거한다.

```text
t0  read request → DB → cache set
t1  write request
t2  cache delete
t3  DB update
t4  read request → DB → cache set
t5  second delete
```

## 8. Write Contention 제어

핫 엔티티 쓰기는 다음 문제를 유발한다.

- DB row lock 경쟁 
- write latency 증가 
- replication lag

이를 해결하는 전략이 필요하다.

## 9. Write Coalescing

동일 키에 대한 여러 쓰기를 합쳐서 처리하는 전략이다.

예:

```text
counter += 1
counter += 1
counter += 1

→ counter += 3
```

적용 사례:

- view count 
- like count 
- metrics 
- inventory update

## 10. Redis Atomic Write

Redis는 원자적 연산을 제공한다.

예:

```text
INCR
HINCRBY
ZINCRBY
```

장점:

- DB write 감소
- write contention 감소

하지만 반드시 DB와 동기화 전략이 필요하다.

## 11. Write Buffer / Write Back Queue

핫 쓰기 트래픽을 DB에서 직접 처리하면 다음이 발생한다.

- DB write QPS 폭증 
- transaction lock 증가

이를 해결하는 방법:

> Redis → write buffer → async DB flush

구조:

```text
Client
↓
Redis write
↓
Kafka / Queue
↓
Batch DB update
```

## 12.Idempotent Write 설계

분산 환경에서 동일 write가 여러 번 실행될 수 있다.

원인:

- retry 
- network timeout 
- duplicate message

따라서 write는 반드시 idempotent해야 한다.

방법:

- request id 
- version column 
- upsert

예:

```text
UPDATE account
SET balance = balance + 100
WHERE id = 1
AND version = 10
```

## 13. Write Ordering 문제

멀티 인스턴스 환경에서는 write 순서가 뒤집힐 수 있다.

예:

```text
write A
write B
```

하지만 실제 실행은

```text
write B
write A
```

이 문제는 versioning으로 해결한다.

## 14. Versioned Cache

cache에 version을 저장한다.

예:

```text
{
"data": {...},
"version": 42
}
```

write 시:

```text
update DB version
invalidate cache
```

read 시:

> cache version < DB version → invalidate

## 15. Hot Key Write 보호

핫키에 대한 write는 Redis 자체가 병목이 될 수 있다.

대응 전략:

- key sharding
- write queue
- batching

예:

```text
counter:{id}:1
counter:{id}:2
counter:{id}:3
```

주기적으로 합산.

## 16. 운영 메트릭

쓰기 캐시는 다음 지표 없이는 운영이 불가능하다.

**Cache**

- write QPS 
- invalidate rate 
- stale read rate

**Redis**

- command latency 
- hot key 
- eviction

**DB**

- write QPS 
- lock wait 
- replication lag

**Correlation**

- write QPS ↔ lock wait 
- invalidate rate ↔ cache miss 
- replication lag ↔ stale read

## 17. 마지막 방어선: Write Rate Limit

쓰기 폭주 상황에서는 DB가 먼저 죽는다.

따라서 다음이 필요하다.

- endpoint rate limit
- entity-level rate limit 
- write queue backlog limit

## 결론

쓰기 캐시 설계의 핵심 목표는 다음이다.

- cache ↔ DB 불일치 최소화 
- write contention 제어 
- DB write amplification 감소 
- retry / duplicate write 안전성 확보

운영에서 가장 안정적인 구조는 다음이다.

> Write Around + Cache Invalidation(Delete) + Versioning + Write Coalescing + Async Flush

정리하면 쓰기 캐시는 단순히 write → cache update 문제가 아니라

- invalidation race 
- write ordering 
- contention 
- duplicate write

까지 고려한 분산 시스템 문제로 설계해야 한다.
