---
title: Redis 캐시 읽기 관점 아키텍처
description: >-
  Redis 캐시 읽기 관점 아키텍처 설계
author: jay
date: 2026-02-02 20:55:00 +0800
categories: [Study, Redis]
tags: [redis, study]
pin: false
media_subpath: ''
---

# Redis 읽기 캐시 안정화 아키텍처

## 1. Read Path의 특성

읽기(Read) 경로는 대개 다음 특성을 동시에 가진다.

- **팬아웃(fan-out)**: 한 요청이 여러 키/여러 서비스로 확장된다.
- **QPS가 높다**: 쓰기보다 훨씬 자주 호출된다.
- **지연 민감**: p50이 아니라 p99/p999가 사용자 경험을 결정한다.
- **동시성 증폭**: 동일한 데이터(같은 키)가 동시에 필요해지는 순간이 있다.


### 1.1 Cache Miss 비용 구조

> 여기서 “캐시 미스 비용”은 DB를 한 번 더 때리는 정도가 아니다. 실무에선 4중 복합 비용으로 나타난다.

1. **네트워크 지연 증가** 
   - Client → App → Redis(미스) → DB → App → Client로 경로가 늘어남. 평균 지연보다 tail latency가 더 크게 상승한다.
2. **DB 리소스 소모**
   - 캐시 미스는 DB QPS를 올린다. 그리고 DB는 CPU보다 락/IO/커넥션풀이 먼저 터진다.
3. **동시성 증폭과 실패 전파**
   - 미스가 동시에 터지면 DB에서 스로틀링/타임아웃이 발생하고, 그것이 앱 서버 스레드/이벤트 루프를 잡아먹어 연쇄 장애로 번진다.
4. **p99 비선형 상승**
   - 미스는 “평균 지연을 조금 올리는” 게 아니라 p99를 비선형으로 끌어올린다. 캐시 설계의 목적은 hit ratio가 아니라 p99 안정화다.

## 2. Cache stampede

> Cache Stampede는 특정 캐시 키가 TTL 만료된 순간, 다수의 클라이언트 요청이 동시에 캐시 미스를 일으켜 백엔드(DB, 서비스 계층)에 집중적인 부하를 발생시키는 현상

발생 조건은 보통 아래 중 하나다.
- 동일 TTL로 대량 생성 → 같은 시각에 대량 만료
- 배치 invalidate / deploy 이후 캐시 cold start
- 특정 이벤트(뉴스, 장 시작, 프로모션)로 트래픽이 한 시점에 급증

### 2.1 Hot Key 스탬피드

> 핫키는 “키 집단이 동시에 만료”가 아니라 키 1개가 QPS를 빨아먹는 구조다.

- 특정 키(예: 메인 페이지 구성, 인기상품 TopN, 환율/지표)가 QPS의 상당 부분 차지
- TTL 만료 시점에 DB QPS가 순간적으로 피크 + p99 지연 급등
- **원인**
  - 핫키는 “만료 순간”에 동시 미스가 대량 발생
  - 동일 키에 대한 동시 갱신(concurrent refresh) 폭발

## 3. 기본 읽기 패턴

### 3.1 Cache Aside (Lazy Loading)

> 요청 → Redis GET → miss면 DB 조회 → Redis SET → 응답

- 장점 : 단순, 운영 편함, RDBMS가 소스 오브 트루스
- 단점: miss 시 DB 부하 급증(스탬피드), 만료 순간 폭발, “Hot Key”에 취약

### 3.2 Read-Through

동작 자체는 지연 로딩과 같지만 요청 → 캐시 계층이 miss면 내부적으로 DB에서 채움

- 장점: 애플리케이션 로직 단순화
- 단점: 구현/운영 복잡(라이브러리/프록시 필요), 장애 격리 어려움

## 4. “읽기 캐시”는 상태 기계로 설계해야 한다

> 실무에서 캐시 읽기 경로를 안정화하려면 “TTL 하나”로 생각하면 안 된다. 키를 **상태(state)**로 정의해야 한다.

### 4.1 키의 4가지 상태

| 상태         | 조건                                | 읽기 응답               | 갱신                    |
| ---------- | --------------------------------- | ------------------- | --------------------- |
| Fresh      | now < softExpireAt                | 캐시 값 반환             | 없음                    |
| Stale      | softExpireAt ≤ now < hardExpireAt | **stale 즉시 반환**     | **일부 요청만 refresh 시도** |
| Expired    | now ≥ hardExpireAt                | miss 취급             | **반드시 refresh 시도**    |
| Refreshing | refresh_lock 존재                   | stale 반환(또는 짧게 재시도) | 중복 갱신 금지              |


이제 “언제 stale을 반환해도 되는가”, “언제 DB로 가야 하는가”, “언제 락이 필수인가”가 정의로 결정

## 5. Cache Stampede 성숙도 모델

아래는 Cache Stampede에 대해 동시성을 제어하는 성숙도의 단계이다.

| 단계      | 전략                           | 목적          |
|---------|------------------------------| ----------- |
| Level 0 | TTL Only                     | 위험한 기본형    |
| Level 1 | TTL Jitter                   | 동시 만료 완화    |
| Level 2 | Single-flight / Refresh Lock | 중복 갱신 차단    |
| Level 3 | Soft TTL + SWR               | 응답 안정화      |
| Level 4 | PER                          | 만료 집중 분산 |
| Level 5 | Rate limit / Circuit breaker | Redis 장애, DB 장애 시 최종 생존 장치 |

최소 실무선은 L3(L2만으로도 stampede는 줄지만, p99가 흔들린다.)

## 6. TTL Only

> 고정적인 TTL을 사용하는 구조로 고트래픽 환경에서 장애 유발이 쉽다.

- 구조
  - 단순 Cache Aside 
  - 고정 TTL 
  - 만료 시 즉시 miss → DB 조회
- 특징
  - 소규모 트래픽에서는 문제 없음
  - 구현이 쉬움
- 한계
  - TTL 동시 만료 
  - Hot key 만료 시 동시 miss 폭발 
  - DB QPS 순간 급증 
  - p99 급등

## 7. TTL Jitter

> TTL(Random Jitter)는 캐시 키의 만료 시간을 고정값이 아니라 무작위 오프셋을 포함한 값으로 설정하는 기법
> - TTL Jitter는 “완화” 전략일 뿐, 해결책은 아니다.
> - Jitter는 “동시 만료 타이밍 분산”일 뿐, “갱신 동시성 제어”가 아니다.

- Base TTL의 10~30% 정도가 적당
- TTL이 짧은데 jitter를 크게 주면 캐시 정책이 깨지고 예측 불가해진다
- 권한/재고/금융 데이터에 과도한 jitter를 주면 stale 리스크가 커진다

**핫키는 jitter로 해결 안 된다.**

핫키는 “만료 순간” 자체가 문제이기 때문.

## 8. Single-flight / Refresh Lock

> Single-flight / Refresh Lock는 평균 hit ratio를 올리는 기술이 아니다. latency를 줄이는 기술도 아니다
> - 동일 키에 대한 동시 갱신을 1회로 수렴시키는 동시성 제어 장치다

### 8.1 single-flight

특정 하나의 키(Hot Key)에 대해 동시에 miss가 폭발하는 것은 Jitter로는 해결이 불가하다.

Cache Stampede의 본질은 “miss”가 아니라
동일 키에 대한 동시 갱신 폭발이다.
Mutex / Single-flight는 이를 직접 통제하는 1차 방어선이다.

> 목적 : 동일 키에 대한 동시 miss를 1회 갱신으로 수렴시키는 것으로 동일키에 대한 동시 갱신 폭발을 방지하는 것이 목적이다.

**방식**

- 로컬 single-flight(프로세스 내부) : 동일한 키(key)에 대한 중복 요청을 하나의 실행으로 합치고, 그 결과를 동일 프로세스 내의 모든 대기 요청이 공유하도록 만드는 동시성 제어 패턴
  - 로컬 single-flight의 경우 모놀리식 기반의 단일 인스턴스라면 적용 가능하다.
- Redis SETNX 기반 refresh lock
- 짧은 TTL의 refresh-lock 키


**동작**

```markdown
1. 요청이 miss 감지 
2. 해당 key에 대해 "이미 갱신 중인지" 확인
3. 첫 요청만 DB 조회
4. 나머지는 대기(wait) 또는 stale 반환
   - stale 데이터를 즉시 반환
   - 동시에 background(또는 inline)로 single-flight 리프레시 트리거
5. 갱신 완료 후 캐시 업데이트
6. 대기 요청에 동일 결과 반환
```

이러한 동작은 아래의 효과가 있다.
- 트래픽 피크에서도 latency를 안정적으로 유지
- 갱신 폭주를 single-flight로 차단
- “stale 요청이 갱신하면 안 된다” 요구를 정확히 만족

**로컬 single-flight의 한계**

> 인스턴스 수만큼 갱신이 발생한다.

- 멀티 인스턴스 환경
- 오토스케일
- 쿠버네티스

예시로:
- 10 pods
- stale QPS = 10,000
- pod별로 독립 시행 결과로 PER 통과 평균 100(p = 10%, 각 파드당 1000개의 stale 그중 100이라서)
- 각 pod에서 100 → 최대 1000 refresh 시도 가능

**이에 대해 TTL의 refresh-lock 키(SETNX)를 활용할 수 있다.(분산락은 과도하게 복잡)**
- 갱신은 “정확성”보다 “중복 억제”가 목적이기 때문입니다.

### 8.2 Redis SETNX 기반 refresh lock

> Redis의 SETNX를 활용한 refresh lock.
> - SET refresh_lock:{key} 1 NX EX 10

- NX → 이미 존재하면 실패
- EX → 자동 만료(데드락 방지)

**동작**

```markdown
1. miss 감지
2. lock 시도
3. 성공한 1개만 DB 조회
4. 실패한 요청은:
   - 잠시 대기 후 재조회
   - 또는 stale 반환
5. 갱신 완료 후 lock 자연 만료
```

### 8.3 대기 전략 비교

**Blocking 방식**

> lock 실패 → sleep(락 대기) → retry

- 일관성 높음
- latency 증가 가능
- Thread 점유, event loop 지연, connection pool 고갈 장애 발생 가능

**Stale 반환 + Background Refresh**

> lock 실패 → stale 반환

- latency 안정
- 약간의 stale 허용

### 8.4 Lock TTL 설계 기준

> Lock TTL ≈ (평균 DB 응답시간 × 2~3)
> - 대개 5~30초 범위.

- **너무 짧으면**
  - DB 조회 끝나기 전에 만료 
  - 다른 요청이 또 lock 획득 
  - 중복 갱신 발생
- **너무 길면**
  - 갱신 실패 시 장시간 갱신 불가 
  - stale 지속

### 8.5 과도한 분산락 설계의 문제

> Cache refresh 락의 목적은 정확성(consistency) 보장이 아니라 **중복 갱신 억제(deduplication)**다.
> 
> 이 목적을 벗어나 “강한 분산 합의 수준”으로 설계하면, 비용만 증가하고 실질 이득은 거의 없다.

1. Refresh Lock의 실제 목표 :
   1. 동일 키에 대해 DB를 여러 번 치지 않도록 제한
   2. 최악의 경우 2~3회 중복은 허용 가능
   3. 시스템 보호가 1순위
   4. 그런데 Redlock, 다중 노드 quorum, fencing token 등을 적용하면 락의 성격이 데이터 정합성 락으로 바뀐다.(목적 대비 과도한 보장)
2. 성능 비용 증가
   1. 네트워크 RTT 증가 : Redlock (3~5노드) → 3~5 RTT (병목 발생)
3. Fail-Over 복잡성 증가: clock drift, lock 만료 타이밍 불일치...
4. 결과적으로 Redis에 부하가 집중되며 Redis가 병목이 될 수 있다.

**단순한 설계가 더 안전한 이유**

> SET refresh_lock:{key} 1 NX EX 10

- 중복 발생에도 치명적 장애로 이어지지 않음
- 짧은 TTL
- 실패 시 무시

## 9. Soft TTL + SWR

### 9.1 왜 Soft TTL이 필요한가?

- Hard TTL : 300초 이후 → 무조건 miss
  - 동시 요청 많으면 → DB 폭발
- Soft TTL : 값과 함께 “만료 시각”을 저장,
  - 본질 :갱신을 지연시키는 게 아니라, 응답을 안정화
  - softExpireAt : 이 시점 이후는 “stale”
  - hardExpireAt : 이 시점 이후는 “완전 만료”
  - 갱신이 실패하면 stale이 무한히 유지될 수 있다.
- Soft TTL을 적용할 때에는 아래의 전략을 통해 stale이 무한히 유지되는 것을 방지해야한다.
  - Hard TTL 반드시 존재
  - refresh 실패 카운트 모니터링
  - stale age 메트릭 수집

Mutex만 쓰면 다른 문제가 생긴다.
- stale 구간 QPS가 크면
- 전부가 lock 경쟁을 하며
- Redis가 병목이 된다

Soft TTL은 이걸 해결한다.
- stale이면 그냥 반환한다(응답 안정)
- 일부만 갱신을 시도한다(부하 상한)

Soft TTL 예시:

```json
{
  "data": {...},
  "softExpireAt": 1700000000,
  "hardExpireAt": 1700000600
}
```

### 9.2 Soft TTL의 필수 조건: Hard TTL

> Soft TTL만 있으면 갱신 실패 시 stale이 무한 유지될 수 있다.

따라서 반드시:
- hardTTL 존재
- stale age 관측
- refresh 실패율 관측

### 9.3 Stale-While-Revalidate (SWR)

> 데이터가 soft TTL을 지나 stale 상태가 되더라도 즉시 응답을 제공하고, 동시에 **백그라운드에서 재검증/재갱신(revalidate)**을 수행 

단 json으로 저장하면 메모리 사용량 증가하므로 대규모 키에서는 그 크기를 고려해야한다.

## 10. Probabilistic Early Refresh(PER) 

> TTL이 끝나는 시점에 한 번에 갱신하지 말고, 만료 직전 구간에서 “일부 요청만” 확률적으로 먼저 갱신
> - PER는 “TTL Jitter의 고급형”이라고 보면 정확하다.

60초 시점에 트래픽이 몰려 있으면 동일 키에 대해 갱신이 동시에 폭발할수 있기 때문에 PER은 이 갱신을 시간적으로 분산시킨다.

하지만, 확률에 기반하기 때문에 stale이 업데이트가 되지 않을 수도 있는데 이를 확률 함수를 설계하여 해결한다.

- 장점
  - TTL 만료 시점 집중 부하 완화
  - 트래픽이 많을수록 자연스럽게 조기 갱신 확률 상승
- 단점
  - 이론적으로는 “예상보다 늦게 갱신”될 수 있음
  - 트래픽이 매우 낮으면 갱신이 지연될 수 있음(Hard TTL은 반드시 별도로 둬야 함)

### 10.1 확률 함수 설계 방식

**1. 선형 증가**
```text
p = 1 - (remaining / window)
```
- 만료에 가까울수록 확률 증가

**2. 지수 기반(트래픽 고려)**
```markdown
refresh if now > expiry - beta * TTL * ln(U)
```
- U = (0,1] 균등 난수
  - Poisson arrival 가정
  - exponential distribution 기반
  - TTL 만료 시점에 집중되지 않도록 분산
- beta = 조정 파라미터
- 트래픽이 많을수록 자연스럽게 조기 갱신 발생

### 10.2 그래도 “락/싱글플라이트”는 필수

> PER은 트리거 시도를 줄이는 장치지, 중복 갱신을 막는 장치가 아니다. 
> - 권장 조합 : TTL Jitter + Soft TTL(SWR) + PER + Refresh Lock(SETNX)

1. 요청이 stale(또는 만료 임박)인지 판단
2. PER 확률을 통과한 일부 요청만 refresh 시도 
3. refresh 시도는 SETNX refresh_lock_key EX 5~30s 같은 짧은 락으로 1개만 DB를 치게 함
   - 복잡한 분산락 보단 짧은 TTL의 refresh-lock 키(SETNX)면 충분
   - 단일 인스턴스(매우 드묾)라면 로컬 single-flight만으로도 충분
4. 갱신 성공 시 캐시 값 + softExpireAt 갱신

예를 들어 
- stale 구간 QPS = 20,000/s
- PER p = 0.01
- 기대 갱신 트리거 = 200/s
  - 초당 200개가 동시에 DB 갱신을 시도할 수 있다.

즉, PER만 쓰면 스탬피드가 “덜” 일어날 뿐, 여전히 폭발할 수 있다.

## 11. 운영 메트릭 없으면 설계는 “없는 것”이다

반드시 뽑아야 하는 지표:

- Cache 
  - hit ratio 
  - stale ratio 
  - refresh trigger rate (PER 통과율)
  - refresh lock contention rate (락 경쟁률)
  - refresh latency p95/p99 
  - stale age 분포(softTTL 이후 얼마나 오래된 값이 반환되는지)
- DB 
  - read QPS 
  - slow query 
  - connection pool 사용률 
  - timeout rate 
- Correlation
  - expiration rate ↔ DB QPS 
  - lock 획득률 ↔ DB QPS 
  - stale ratio ↔ p99 latency

## 12. 마지막 방어선: Rate limit / Circuit breaker

Redis 장애나 대규모 cold start가 오면 결국 DB로 트래픽이 새는데, 이때 DB 보호 장치가 없으면 무조건 죽는다.

- endpoint 단위 rate limit
- key-space 단위 rate limit(핫키)
- DB timeout을 짧게 가져가고
- stale 허용 가능한 구간은 stale로 degrade

# 결론

읽기 캐시의 목표는 hit ratio가 아니라:
- p99 안정화 
- DB QPS 상한 생성 
- 동시 갱신 폭발 차단 
- 운영 메트릭 기반 튜닝 가능

정리하면, 운영에서 가장 안전한 형태는:

> 키 상태(Fresh/Stale/Expired) 기반으로 SWR을 적용하고, 갱신 트리거는 PER로 분산하며, 갱신 자체는 SETNX refresh lock으로 1회로 수렴시키는 구조
