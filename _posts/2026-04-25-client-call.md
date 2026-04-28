---
title: 외부 Client 호출 설계
description: >-
  외부 호출 시스템 설계
author: jay
date: 2026-04-25 20:55:00 +0800
categories: [Study, Architecture]
tags: [architecture, code]
pin: false
media_subpath: ''
---

# 외부 Client 호출 설계

> 외부 Client 호출하는 것은 단순히 “HTTP 요청을 보내고 응답을 받는 코드”로 접근하기 보단 시스템에서의 **일관성, 장애 전파, 자원 관리까지 포함된 설계 문제**로 바라봐야한다.

## 1. 로컬 트랜잭션 vs 외부 시스템

> 애플리케이션 내부는 DB 트랜잭션을 단위로 통제된다. 하지만 외부 API는 **트랜잭션 경계 밖의 불확실한 시스템**이다.

즉, 다음과 같은 상황이 발생할 수 있다.

- 우리 DB는 commit 되었지만 외부 호출 실패
- 외부는 성공했지만 우리 시스템은 timeout으로 실패 처리
- 네트워크 장애로 결과를 알 수 없음

### 1.1 트랜잭션 설계

> 외부 호출을 트랜잭션 내부에 넣는 순간 시스템은 병목 + 불일치 리스크를 동시에 가진다.

트랜잭션 내부의 외부 호출은 다음과 같은 문제를 야기한다.

- 트랜잭션 유지시간 증가 → DB 커넥션 고갈
- 외부 API latency = 전체 시스템 latency
- 실패 시 rollback → 이미 외부는 처리됐을 수 있음

따라서

핵심은 트랜잭션 경계와 외부 호출 경계를 분리하는 것이다.

### 1.2 데이터 일관성 전략

> 외부 시스템은 ACID 트랜잭션에 묶을 수 없다. 따라서 결국 비동기 + 보상 모델로 간다.

예를 들자면 아래와 같이 sequence를 설계할 수 있다.(아래의 sequence 전체를 하나의 Transaction으로 묶어선 안된다.)

- 일단 먼저 데이터를 저장(외부 호출 전)
- 외부 호출 Request 저장(외부 호출)
- 외부 호출 결과에 따라 분기(실패 시 보상, 성공 시 update, insert등)

즉 실패에 따라 rollback이 아닌 undo 작업을 정의해야한다.

### 1.3 Idempotency

외부 호출은 retry, timeout 후 재시도, 메시지 중복 처리와 같은 상황에서 반드시 중복으로 호출된다.

이와 같은 문제에 대한 해결으로:

1. idempotency key를 활용한 중복 요청 차단
2. 외부 시스템에서 지원하지 않는다면 내부에서 unique key 같은 전략을 활용하여 처리

그런데 idempotency key를 활용해서 요청을 필터링 하면 데이터에서 굳이 unique를 사용할 필요가 없다고 느낄 수도 있다.

두 layer에서 하는 역할이 똑같다고 해서 한 곳으로 문제를 해결하려할 경우 예상치 못한 문제를 겪을 수 있다.(왜냐하면 각 layer는 역할이 다르기 때문에 시스템에서 지원하는 일관성 정도도 다르고 작동 방식도 다르기 때문이다.)

예를 들자면 이번 문제의 경우 각 layer의 핵심적인 역할의 차이는 다음과 같다

| 구분                | 역할                     |
| ----------------- | ---------------------- |
| Idempotency Key   | “이 요청이 같은 요청인지” 식별     |
| Unique Constraint | “동시에 들어온 중복 저장”을 최종 차단 |

즉 애플리케이션에서 아래와 같이 작성해도

```java
if (repository.existsByIdempotencyKey(key)) {
    return existingResult;
}

repository.save(request);
```

아래와 같은 동시요청에 대해서는 중복을 처리할 수 없다.

```text
Thread A: exists false
Thread B: exists false
Thread A: insert
Thread B: insert
```

여기서 application은 중복 **요청**을 처리하고 unique는 **중복 처리를 물리적으로 방지하는 마지막 방어선**이기 때문에 둘다 처리를 해야한다.

## 2. 장애 전파 차단: Circuit Breaker 

> Circuit Breaker는 외부 시스템 장애가 내부 시스템으로 전파되는 것을 막기 위한 장애 차단 장치다.

- “실패가 일정 수준 이상이면, 더 이상 호출하지 않는다.”

Circuit Breaker 외부 API가 장애 상태인데 계속 호출하면 생기는 다음과 같은 문제를 방지한다.

```text
외부 API 지연
→ 우리 서버 Thread 대기
→ DB Connection / Thread Pool 점유
→ 요청 적체
→ 전체 서비스 장애
```

즉 외부 장애가 내부 서비스 장애로 번지는 장애 전파를 끊는다.

### 2.1 상태 모델

Circuit Breaker는 보통 3가지 상태를 가진다.

```text
CLOSED → OPEN → HALF_OPEN → CLOSED
```

#### CLOSED : 정상 상태

> 요청을 그대로 외부 API로 보낸다.

- 이때 실패율을 계속 집계한다.

예)

```text
slidingWindowSize: 10    // 최근 10건 중
minimumNumberOfCalls: 5  // 최소 5건 이상 집계됐고
failureRateThreshold: 50 // 실패율이 50% 이상이면 OPEN
```

#### OPEN : 차단 상태

> 외부 API를 호출하지 않는다.

- Resilience4j에서는 보통 CallNotPermittedException이 발생한다.

이 상태의 목적은 명확하다.

- 외부 시스템 보호
- 우리 서버 리소스 보호
- 불필요한 timeout 대기 제거

#### HALF_OPEN : 복구 확인 상태

> 일정 시간이 지나면 Circuit Breaker는 일부 요청만 테스트로 통과시킨다.

예)
```text
waitDurationInOpenState: 10s                // OPEN 상태로 10초 대기 후 HALF_OPEN 전환
permittedNumberOfCallsInHalfOpenState: 3    // 테스트 요청 3건 허용
```

여기서 결과가 좋으면 CLOSED, 다시 실패하면 OPEN


### 2.2 Circuit Breaker가 잡는 실패

보통 다음을 실패로 본다.

- HTTP 5xx
- timeout
- connection refused
- socket timeout
- 외부 시스템 장애 응답
- RestClientException
- WebClientRequestException

반대로 보통 실패로 보면 안 되는 것:

- validation error
- business error
- 사용자 입력 오류
- 4xx 중 일부

예를 들어 외부 KYC 업체가 이렇게 응답했다고 하자

```text
{
  "status": "REJECTED",
  "reason": "INVALID_DOCUMENT"
}
```

이건 외부 장애가 아니다. 정상 응답이다. 따라서 Circuit Breaker 실패율에 포함하면 안된다.

### 2.3 Circuit Breaker 설정 예시

```yaml
resilience4j:
  circuitbreaker:
    instances:
      amlGuard:
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 20
        minimumNumberOfCalls: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 10s
        permittedNumberOfCallsInHalfOpenState: 3
        automaticTransitionFromOpenToHalfOpenEnabled: true
```

| 설정                                      | 의미                       |
| --------------------------------------- | ------------------------ |
| `slidingWindowType`                     | 실패율 계산 방식                |
| `slidingWindowSize`                     | 최근 몇 건 기준으로 판단할지         |
| `minimumNumberOfCalls`                  | 판단을 시작하기 위한 최소 호출 수      |
| `failureRateThreshold`                  | 실패율 임계값                  |
| `waitDurationInOpenState`               | OPEN 상태 유지 시간            |
| `permittedNumberOfCallsInHalfOpenState` | HALF_OPEN에서 허용할 테스트 호출 수 |

### 2.4 Retry와 같이 쓸 때

> Circuit Breaker와 Retry를 같이 쓰면 순서가 중요하다.

일반적으로 아래와 같은 순서로 호출한다.(하나의 사용자 요청 안에서 짧게 재시도, 최종 실패만 CircuitBreaker에 기록) 

```text
Retry → CircuitBreaker → External API
```

반대의 순서로 하게되면 retry 각각이 실패로 집계되어 Circuit Breaker가 너무 빨리 열릴 수 있기 때문이다.

### 2.5 Timeout과 관계

> Circuit Breaker만으로는 느린 호출을 끊지 못한다.

- Circuit Breaker = 실패율 기반 차단
- Timeout = 오래 걸리는 호출 차단

### 2.6 Bulkhead와의 관계

> Circuit Breaker가 실패율 기반 차단이라면, Bulkhead는 동시 호출 수 제한이다.

- CircuitBreaker: 장애가 많으면 차단
- Bulkhead: 동시에 너무 많이 못 들어오게 차단

## 3. 장애 전파 차단: Timeout 

> Timeout은 외부 Client 호출 설계에서 가장 먼저 설정해야 하는 방어선이다.

- Circuit Breaker가 “실패율이 높으니 앞으로 막자”라면, Timeout은 “이번 호출이 너무 오래 걸리니 지금 끊자”다.

```text
Timeout = 단일 요청의 최대 대기 시간 제한
Circuit Breaker = 반복 실패에 대한 차단
Bulkhead = 동시 호출 수 제한
Retry = 일시 실패에 대한 재시도
```

외부 요청에 대해 timeout이 없으면 아래와 같은 문제가 발생할 수 있다.

```text
외부 API 지연
→ 애플리케이션 Thread 대기
→ Thread Pool 고갈
→ DB Connection 점유
→ 요청 적체
→ 전체 서비스 장애
```

### 3.1 Timeout 종류

#### Connection Timeout

> 외부 서버와 TCP 연결을 맺는 데 허용하는 시간

- 서버에 연결 자체가 안 됨
- DNS / 네트워크 / 방화벽 / 서버 다운

이 값은 짧게 잡는 게 맞다. 연결도 못 하는 상황에서 오래 기다릴 이유가 없다.

#### Read Timeout / Response Timeout

> 연결은 됐지만 응답을 기다리는 시간

- 연결 성공
  - 요청 전송
  - 응답이 늦음

외부 API의 SLA와 업무 중요도에 맞춰 정한다.

#### Call Timeout / TimeLimiter

> 전체 호출에 허용하는 총 시간(Resilience4j의 TimeLimiter가 여기에 가깝다.)

- connection + request write + response read + 내부 처리

### 3.2 중요한 차이

- HTTP Client Timeout : HTTP 레벨에서 실제 네트워크 I/O를 끊는다.
  - connect timeout
  - read timeout
  - response timeout
- Resilience4j TimeLimiter : 애플리케이션 레벨에서 Future 대기를 끊는다.
  - 호출하는 쪽은 TimeoutException을 받음
  - 하지만 실제 작업 Thread가 즉시 멈춘다는 보장은 약함

> 그래서 TimeLimiter만 믿으면 안 된다.

### 3.3 실무 원칙

#### 원칙 1. HTTP Client Timeout은 반드시 설정한다

- RestClient를 쓴다면 내부 Client에 timeout을 설정해야 한다.

예)

```java
@Bean
RestClient restClient() {
    RequestConfig requestConfig = RequestConfig.custom()
        .setConnectTimeout(Timeout.ofSeconds(1))
        .setResponseTimeout(Timeout.ofSeconds(3))
        .build();

    CloseableHttpClient httpClient = HttpClients.custom()
        .setDefaultRequestConfig(requestConfig)
        .build();

    return RestClient.builder()
        .requestFactory(new HttpComponentsClientHttpRequestFactory(httpClient))
        .build();
}
```

#### 원칙 2. TimeLimiter는 외부 호출 전체 보호용으로 사용한다

> 이건 HTTP timeout의 대체재가 아니다. 둘 다 있어야 한다.

예)

```java
TimeLimiter timeLimiter = TimeLimiter.of(
    "provider",
    TimeLimiterConfig.custom()
        .timeoutDuration(Duration.ofSeconds(3))
        .cancelRunningFuture(true)
        .build()
);
```

#### 원칙 3. Timeout은 Retry보다 짧고 명확해야 한다

> Retry를 쓸 경우 전체 소요 시간이 커진다.


```text
timeout: 3s
retry:
  maxAttempts: 3
  waitDuration: 500ms
```

위 설정에 따르면 아래와 같은 문제(사용자 요청 하나가 10초 이상 붙잡힘)가 발생할 수 있다.

```text
1차 3s
    대기 0.5s
2차 3s
    대기 0.5s
3차 3s
= 총 10s
```

### 3.4 Timeout 값 잡는 법

```markdown
우리 API SLA

- 내부 처리 시간
- 네트워크 여유
- Retry 대기 시간

= 외부 호출에 줄 수 있는 최대 시간
```


예를 들어 우리 API SLA가 2초라면:
``` text
전체 SLA: 2s
내부 처리: 300ms
DB 처리: 200ms
여유: 300ms
외부 API 허용 시간: 약 1.2s
```

### 3.5 Timeout과 Transaction

그럼 Timeout이 있으면 내부 Transaction에서 호출해도 되는 것이 아닌가 하는 의문이 들 수 있다.

결론은, 외부 호출 timeout이 있더라도 트랜잭션 내부에서 호출하면 안 된다.

> 왜냐하면 timeout 동안 DB 트랜잭션과 커넥션이 계속 잡혀 있기 때문이다.

올바른 구조:
```java
public void process() {
    Long id = service.saveRequest(); // transaction
    providerClient.call(id);         // non-transaction
}
```

### 3.6 Timeout과 Retry

> Timeout이 발생했다고 무조건 retry하면 안 된다.

- 우리 쪽은 timeout
- 외부 시스템은 실제로 성공 처리
- 이 상황에서는 중복으로 처리될 수 있다.

따라서 retry는 다음과 같은 조건이 필요하다.

1. Idempotency Key 있음
2. 외부 API가 멱등 처리 보장
3. timeout 이후 상태 조회 API가 있음

없으면 retry보다 “결과 확인”이 먼저다.

## 4. 장애 전파 차단: Timeout

> Bulkhead는 외부 Client 호출 설계에서 리소스 격리(Concurrency Isolation)를 담당한다.

- “특정 외부 시스템이 느리거나 장애가 나도, 그 영향이 전체 시스템으로 퍼지지 않게 막는다.”
- 이 개념은 선박 구조에서 왔다. 격벽(Bulkhead)으로 구획을 나눠 한쪽이 침수돼도 전체가 가라앉지 않게 한다.

외부 API 호출은 보통 다음 자원을 소비한다.

- Thread
- Connection
- CPU (serialization, parsing)

문제는 외부 API가 느려지면 이 자원이 오래 점유된다는 점이다.

```text
외부 API 지연
→ Thread 오래 점유
→ Thread Pool 고갈
→ 다른 API까지 대기
→ 전체 서비스 장애
```

이걸 막는 게 Bulkhead다.

Resilience4j 에서는 두 가지를 제공한다.

### 4.1 Semaphore Bulkhead

- 동시 호출 수만 제한
- Thread는 호출자 Thread 사용

예)

```java
Bulkhead bulkhead = Bulkhead.of(
    "provider",
    BulkheadConfig.custom()
        .maxConcurrentCalls(10)
        .maxWaitDuration(Duration.ofMillis(0))
        .build()
);
```

#### 동작

- 동시 호출 10개까지 허용
- 11번째 요청 → 바로 실패 (또는 대기)

#### 특징

- 가볍다
- Thread pool 추가 필요 없음
- 일반적인 동기 호출에 적합

### 4.2 ThreadPool Bulkhead

- 별도 ThreadPool로 격리
- 호출을 다른 Thread에서 실행

예)

```text
ThreadPoolBulkheadConfig config = ThreadPoolBulkheadConfig.custom()
    .maxThreadPoolSize(10)
    .coreThreadPoolSize(10)
    .queueCapacity(0)
    .build();
```

#### 동작

- 요청 → 별도 ThreadPool로 위임
- Pool 꽉 차면 → 거절

#### 특징

- 강한 격리 (Thread까지 분리)
- 대신 오버헤드 있음
- I/O 작업에 적합

### 4.3 핵심 설정

1. maxConcurrentCalls : 동시에 허용할 최대 호출 수
   - 너무 크면 의미 없고, 너무 작으면 성능 제한된다.
2. maxWaitDuration
   - 이렇게 0에 가깝게 두는 게 일반적으로 맞다.
   - 외부 API 호출을 “대기열에 쌓는 것” 자체가 위험하기 때문

#### 왜 queue를 최소화해야 하나

아래와 같은 문제가 발생할 수 있기 때문이다.

```text
요청 100개
→ Bulkhead 10
→ 나머지 90개 대기
→ 외부 장애 지속
→ 대기열 폭발
→ 시스템 지연 증가
```

그래서 보통 전략은: “기다리지 말고 바로 실패”

### 4.4 예외 처리

Bulkhead가 꽉 차면 BulkheadFullException이 발생한다.

```text
catch (BulkheadFullException e) {
    throw new BusinessException(PROVIDER_BUSY);
}
```

### 4.5 Redis 기반의 Rate Limiting과의 차이

> 둘 다 “요청을 제한한다”는 점은 같지만, 제어 대상과 목적이 완전히 다르다.

- 같은 문제를 푸는 도구가 아니라 서로 다른 레이어의 보호 장치다.
- Bulkhead = 내부 자원 보호 (동시 실행 수 제한)
- Rate Limiting = 외부 유입 제어 (요청 속도 제한)

| 구분 | Bulkhead                | Rate Limiting (Redis) |
| -- | ----------------------- | --------------------- |
| 기준 | 동시 실행 수 (Concurrency)   | 시간당 요청 수 (Throughput) |
| 단위 | “지금 몇 개 돌고 있냐”          | “초당 몇 개 들어오냐”         |
| 위치 | 내부 (Service → External) | 경계 (Client → Service) |
| 목적 | Thread / Connection 보호  | 트래픽 제어 / 남용 방지        |


#### 제어 기준

| 구분 | Bulkhead                | Rate Limiting (Redis) |
| -- | ----------------------- | --------------------- |
| 기준 | 동시 실행 수 (Concurrency)   | 시간당 요청 수 (Throughput) |
| 단위 | “지금 몇 개 돌고 있냐”          | “초당 몇 개 들어오냐”         |
| 위치 | 내부 (Service → External) | 경계 (Client → Service) |
| 목적 | Thread / Connection 보호  | 트래픽 제어 / 남용 방지        |

#### 동작 방식 차이

Bulkhead:

```text
동시에 10개만 실행 가능
11번째 요청 → 즉시 실패

[요청 100개]
→ 동시에 10개만 실행
→ 나머지는 거절
```

Redis Rate Limiting(예: 초당 100건 제한):

```text
1초에 100개까지 허용
101번째 요청 → 차단
```

#### 핵심 차이: 시간 vs 점유

- Rate Limiting은 시간 기반
  - 초당 100건 OK
  - 각 요청이 10초씩 걸림(이건 고려하지 않음)
- Bulkhead 동시에 10개만 실행
  - 요청이 오래 걸리면 자연스럽게 제한됨

#### 실제 장애 시 차이

Rate Limiting만 있는 경우
```text
초당 100 요청 계속 들어옴
→ 요청 100개씩 계속 쌓임
→ Thread 100개 점유
→ 결국 고갈
```

Bulkhead 있는 경우
```text
동시 10개 제한
→ 나머지 요청 즉시 실패
→ 시스템 안정 유지
```
