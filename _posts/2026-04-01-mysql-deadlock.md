---
title: MySQL과 Deadlock
description: >-
  MySQL 시스템의 Deadlock 감지와 복구
author: jay
date: 2026-04-01 20:55:00 +0800
categories: [Study, Database]
tags: [database, operation, deadlock]
pin: false
media_subpath: ''
---

# MySQL 과 Deadlock

## 1. MySQL Deadlock 탐지 방식

> MySQL의 deadlock(교착 상태) 탐지는 InnoDB 스토리지 엔진 내부에서 수행되며, 기본적으로 wait-for graph 기반 사이클 탐지 알고리즘을 사용

### 1.1 기본 개념: Wait-for Graph

InnoDB는 트랜잭션 간 락 대기 관계를 그래프로 표현한다.

- 노드: 트랜잭션 (T1, T2, ...)
- 간선: “T1 → T2” = T1이 T2가 가진 락을 기다리는 상태

이 그래프에서 **사이클(cycle)**이 발생하면 deadlock이다.

예시:
```text
T1 → T2
T2 → T3
T3 → T1  ← cycle → deadlock
```

## 2. Deadlock 탐지 방식

### 2.1 즉시 탐지(Eager Detection)

InnoDB는 락을 기다리는 순간마다 deadlock을 검사한다.

어떤 트랜잭션이 lock wait 상태로 들어가면:

- wait-for graph에 edge 추가
- DFS 기반 cycle detection 수행
- 사이클 발견 시 deadlock 판단

즉, 주기적 검사(X), 이벤트 기반 검사(O)

### 탐지 알고리즘 (구현 관점)

> 핵심은 DFS 또는 트랜잭션 체인 추적이다.

1. 트랜잭션 T1이 lock wait 발생
2. T1이 기다리는 트랜잭션 T2 확인
3. T2가 또 기다리는 대상 T3 확인
4. 재귀적으로 탐색
5. 다시 T1로 돌아오면 → cycle → deadlock

## 3. Deadlock 발생 시 처리

### 3.1 Victim 선택

> 하나의 트랜잭션을 강제로 rollback

선정 기준:

- undo log가 가장 작은 트랜잭션
  - 즉, rollback cost 최소화

→ “least cost victim selection”

### 3.2 결과

victim 트랜잭션:
```text
ERROR 1213 (40001): Deadlock found when trying to get lock
```
- 나머지 트랜잭션은 계속 진행

## 4. 설정 옵션

### 4.1 deadlock detection ON/OFF

```text
innodb_deadlock_detect = ON
```
- ON (기본값): 즉시 deadlock 탐지
- OFF: 탐지 안 하고 timeout 기반 처리

### timeout fallback

```text
innodb_lock_wait_timeout = 50
```

deadlock detection이 꺼져 있거나 탐지 실패 시:
→ 일정 시간 후 timeout으로 rollback

## 5. 성능 관점 

> deadlock detection은 high contention 환경에서 wait-for graph 탐색이 빈번하게 발생

- high concurrency + hotspot row 환경에서는 
  - innodb_deadlock_detect = OFF
  - timeout 방식으로 운영하기도 함
- 특히 write-heavy 시스템

## 6. 실제 로그 확인

```text
SHOW ENGINE INNODB STATUS;
```
→ 가장 최근 deadlock 정보 출력

또는

```text
SET GLOBAL innodb_print_all_deadlocks = ON;
```

- 모든 deadlock을 error log에 기록
  - SHOW VARIABLES LIKE 'log_error';
  - 결과 예: /var/log/mysql/error.log
