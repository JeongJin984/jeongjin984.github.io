---
title: redis 캐시 아키텍처
description: >-
  redis의 데이터 영속성 관리
author: jay
date: 2026-02-02 20:55:00 +0800
categories: [Study, Redis]
tags: [redis, study]
pin: false
media_subpath: ''
---

# Redis 읽기 관점 아키텍처

읽기(Read)는 대개 QPS가 높고(팬아웃), 지연에 민감합니다.

## 지연 로딩 패턴

> 요청 → Redis GET → miss면 DB 조회 → Redis SET → 응답

- 장점 : 단순, 운영 편함, RDBMS가 소스 오브 트루스
- 단점: miss 시 DB 부하 급증(스탬피드), 만료 순간 폭발, “Hot Key”에 취약

### 필수 보강

- TTL 랜덤 지터 
- single-flight(동시 miss 합치기)
- negative caching(없는 값도 짧게 캐시)

## Read-Through 패턴

동작 자체는 지연 로딩과 같지만 요청 → 캐시 계층이 miss면 내부적으로 DB에서 채움

- 장점: 애플리케이션 로직 단순화
- 단점: 구현/운영 복잡(라이브러리/프록시 필요), 장애 격리 어려움

## Cache stampede

> Cache Stampede는 특정 캐시 키가 TTL 만료된 순간, 다수의 클라이언트 요청이 동시에 캐시 미스를 일으켜 백엔드(DB, 서비스 계층)에 집중적인 부하를 발생시키는 현상

예로
- 동일한 TTL(예: 300초)을 가진 10만 개 키가 거의 동시에 생성됨
- 300초 후 대량의 키가 동시에 만료
- 순간적으로 DB/백엔드에 트래픽이 폭증

다수의 키 혹은 Single Key에 대한 Stampede 방지를 위해서는 4단계의 성숙도를 거친다.

| 단계      | 전략                    | 목적          |
| ------- | --------------------- | ----------- |
| Level 1 | TTL Jitter            | 동시 만료 완화    |
| Level 2 | Mutex / Single-flight | 중복 갱신 차단    |
| Level 3 | SWR (Soft TTL)        | 지연 안정화      |
| Level 4 | PER                   | 만료 직전 부하 분산 |


## TTL Jitter

> TTL(Random Jitter)는 캐시 키의 만료 시간을 고정값이 아니라 무작위 오프셋을 포함한 값으로 설정하는 기법
> - TTL Jitter는 “완화” 전략일 뿐, 해결책은 아니다.
> - Jitter는 “동시 만료 타이밍 분산”일 뿐, “갱신 동시성 제어”가 아니다.

실무 기준 Jitter의 크기 설정은 Jitter는 Base TTL의 10~30% 범위가 적절하다.

| Base TTL | 권장 Jitter |
| -------- | --------- |
| 60초      | 10~20초    |
| 5분       | 30~90초    |
| 1시간      | 2~5분      |

**주의**
- TTL이 짧은데 Jitter를 크게 주는 것(TTL 전략이 깨져 예측이 불가능해짐)
- 재고/권한/금융 데이터에 과도한 Jitter(보안 stale 리스크 증가)

**또한 다음 상황에서는 추가 전략이 필요**
- Hot Key 1개가 초당 수만 QPS
- 특정 이벤트 이후 대량 재조회 발생
- TTL이 동일한 시점에 강제로 invalidate 되는 경우

**운영 관점에서는 다음을 체크하자**
- Redis Keyspace expiration rate 모니터링 
- DB QPS와 expiration correlation 분석 
- 특정 시간대에 spike가 보이면 Jitter 범위 재조정 
- 캐시 히트율이 과도하게 흔들리면 Jitter 과다 가능성

## Soft TTL + SWR

### Hot Key 스탬피드

> 키 1개가 다 빨아먹고 DB로 새는 순간 터짐

- 특정 키(예: 메인 페이지 구성, 인기상품 TopN, 환율/지표)가 QPS의 상당 부분 차지
- TTL 만료 시점에 DB QPS가 순간적으로 피크 + p99 지연 급등
- **원인**
  - 핫키는 “만료 순간”에 동시 미스가 대량 발생
  - TTL 지터는 “동시 만료”를 줄여도, 핫키 1개는 여전히 만료 이벤트가 큼

### Stale-While-Revalidate (SWR)

> 데이터가 soft TTL을 지나 stale 상태가 되더라도 즉시 응답을 제공하고, 동시에 **백그라운드에서 재검증/재갱신(revalidate)**을 수행 

- Hard TTL : 300초 이후 → 무조건 miss
  - 동시 요청 많으면 → DB 폭발
- Soft TTL : 값과 함께 “만료 시각”을 저장, 
  - 본질 :갱신을 지연시키는 게 아니라, 응답을 안정화
  - softExpireAt : 이 시점 이후는 “stale”
  - hardExpireAt : 이 시점 이후는 “완전 만료”

Soft TTL 예시:

```json
{
  "data": {...},
  "softExpireAt": 1700000000,
  "hardExpireAt": 1700000600
}
```

### SWR + single-flight

아래와 같은 문제 발생 가능하기 때문에 **로컬 single-flight**나 Redis 분산락을 활용할 수 있다.

```text
stale 1000 갱신 요청
→ 1000개가 비동기 갱신 시작
→ DB 과부하
```

> single-flight란 동일한 키(key)에 대한 중복 요청을 하나의 실행으로 합치고, 그 결과를 동일 프로세스 내의 모든 대기 요청이 공유하도록 만드는 동시성 제어 패턴
> - 로컬 single-flight의 경우 모놀리식 기반의 단일 인스턴스라면 적용 가능하다.

1. 캐시 조회
2. fresh면 그대로 반환
3. stale이면
   - stale 데이터를 즉시 반환
   - 동시에 background(또는 inline)로 single-flight 리프레시 트리거
4. 리프레시가 끝나면 캐시 갱신
5. 이후 요청은 fresh를 받음

이러한 동작은 아래의 효과가 있다.
- 트래픽 피크에서도 latency를 안정적으로 유지
- 갱신 폭주를 single-flight로 차단
- “stale 요청이 갱신하면 안 된다” 요구를 정확히 만족

### 로컬 single-flight의 한계

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

## Probabilistic Early Refresh

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

### 확률 함수 설계 방식

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
- beta = 조정 파라미터
- 트래픽이 많을수록 자연스럽게 조기 갱신 발생

### 그래도 “락/싱글플라이트”는 필수

> PER은 트리거 시도를 줄이는 장치지, 중복 갱신을 막는 장치가 아니다. 

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
