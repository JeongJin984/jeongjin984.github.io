---
title: redis 영속성
description: >-
  홈서버에 k8s 구성하기 1편
author: jay
date: 2026-02-01 20:55:00 +0800
categories: [Study, Redis]
tags: [redis, study]
pin: false
media_subpath: ''
---

# Redis 영속성(Persistence)

Redis는 기본적으로 **in-memory 데이터 저장소**이다.  
프로세스가 종료되면 메모리 데이터는 사라진다. 이를 방지하기 위해 영속성(Persistence) 기능을 제공한다.

영속성 전략은 다음 세 가지 조합으로 이해하면 된다.

| 방식 | 개념 | 성능 영향 | 데이터 손실 가능성 | 운영 난이도 |
|------|------|------------|--------------------|--------------|
| RDB  | 시점 스냅샷 저장 | 낮음 | 높음 (스냅샷 이후 유실) | 낮음 |
| AOF  | 모든 write 로그 기록 | 중간~높음 | 낮음 | 중간 |
| RDB + AOF | 혼합 전략 | 중간 | 매우 낮음 | 중간 |


# RDB (Redis Database Snapshot)

RDB는 특정 시점의 메모리 상태를 통째로 덤프하여 파일로 저장하는 방식이다.

## 동작 원리 – BGSAVE

RDB는 BGSAVE 명령을 통해 생성된다. 내부 흐름은 다음과 같다.

1. fork() 호출
2. 자식 프로세스 생성 
3. 자식이 현재 메모리 상태를 RDB 파일로 write 
4. write는 OS page cache에 적재 
5. fsync를 통해 디스크 flush

### 핵심 메커니즘: Copy-on-Write (CoW)

> 데이터를 직접 수정하지 않고, 변경이 발생하는 시점에 기존 데이터를 복사한 뒤 수정하는 방식이다.

- 기존 블록을 건드리지 않음
- 수정 시 새 블록을 생성
- 포인터만 새 블록으로 전환

### CoW로 인한 추가 메모리 사용

fork() 이후 부모와 자식은 메모리 페이지를 공유합니다. 따라서 부모 프로세스에서 write가 발생하면 자식과 차이가 발생할 수 있는데, 그 경우 차이만큼 메모리를 추가로 사용합니다.
왜냐하면 RDB 생성 중에 발생한 데이터 변경을 반영하기 위해서 변경된 내역에 해당하는 페이지를 또 복사해야하기 때문입니다.

즉
```markdown
기존 페이지 공유
→ write 발생
→ 해당 페이지 복사
→ 수정본은 새 페이지에 기록
```

따라서 RDB 생성 중 write 트래픽이 많으면 다음이 발생합니다.

```markdown
추가 메모리 ≈ RDB 수행 시간 동안 변경된 페이지 수 × 페이지 크기
```

## RDB의 주요 리스크

RDB를 생성할 때는 리스크가 있으므로 아래의 내용을 확인합니다.

1. 스냅숏을 사용하기 위한 메모리가 충분한지 확인
2. 문제없이 레플리카에서 스냅숏을 가져올 수 있는지 확인
3. 서비스 지장이 없는 시간에 스냅숏을 생성하는지

### 1. fork latency

- fork는 부모 프로세스의 페이지 테이블을 복사
- 메모리 크기에 비례하여 시간 증가 
- 대용량 인스턴스(수십 GB 이상)에서는 수백 ms~수초 block 가능 

이 구간에서 latency spike 발생 가능.

### 2. CoW 메모리 폭증

- RDB 수행 중 write가 많으면 OOM 위험
- 특히 maxmemory 근접 환경에서 치명적

따라서 백업할때 아래의 사항을 확인해야 합니다.

### 3. 디스크 I/O burst

- 대용량 RDB 파일을 한 번에 write 
- 마지막 fsync 시 대량 flush 
- I/O spike → 서비스 지연

이를 완화하기 위해 아래를 통해 4MB 단위로 점진적 fsync 수행.

```markdown
rdb-save-incremental-fsync yes
```

### Cow(Copy-on-Write) 방식

> 데이터를 직접 수정하지 않고, 변경이 발생하는 시점에 기존 데이터를 복사한 뒤 수정하는 방식이다. 

- 기존 블록을 건드리지 않음
- 수정 시 새 블록을 생성
- 포인터만 새 블록으로 전환

```text
Block A (유지)
Block A' (수정본 생성)
→ 메타데이터가 A'를 가리킴
```

## AOF

> 모든 WRITE 명령 로그를 작성 중인 파일 끝에 계속 추가하며 기록하는 방식

AOF를 이해하기 위해 7.0 이전의 재작성 처리 흐름은 다음과 같습니다.(메모리 할당은 스냅숏과 마찬가지로 CoW 매커니즘을 사용) AOF 재작성 중 중간에 실패해도 문제가 없도록 현재 사용 중인 AOF파일에 지속적으로 쓰기 작업 명령이 추가됩니다. 
또한 파일의 데이터 손실이 확인되면 기본적으로 많은 데이터를 불러오지만 무결성을 위해 오류를 만들어 서버가 실행되지 않도록 할 수 있습니다. 이 경우에는 redis-check-aof 도구를 사용하여 AOF 파일을 수정해야합니다.(aof-load-truncated 지시자를 사용해 관련 설정을 할 수 있으며 매개변수는 파일은 형식은 정상이지만 중간에 잘려있는 데이터가 있을 때 유효합니다.)

1. 요청 처리 중인 프로세스에서 AOF를 재작성하기 위해 작식 프로세스 fork
2. 자식 프로세스는 새로운 AOF 파일을 생성한 후 재작성 결과를 저장
3. 자식 프로세스가 새로운 AOF 파일의 재작성을 완료하면 부모 프로세스에 신호를 보냄
4. 신호를 받은 부모는 포크 이후의 쓰기 작업 내용을 AOF 재작성 버퍼에 저장했다가 자식에 전송(자식은 데이터 차이를 새롭게 생성된 AOF 파일에 반영)
5. 오래된 AOF 파일 교체

스냅숏과 비슷하게 AOF 파일을 재작성할 때 4MB마다 디스크로 플러시 할 수 있습니다.(aof-rewrite-incremental-fsync로 제어)

주의할 점은 ElasticCache같은 관리형 서비스를 사용할 경우 복원 효과가 없을 수 있습니다. 장애가 발생하여 노드 교체가 이루어지면 AOF 파일은 임시 저장소에 저장되므로 새로운 노드에는 파일이 없어 적용할 수 없습니다.

### 7.0 이후

멀티파트 AOF 기능이 도입되었습니다. 단일 AOF 파일 구조에서는 다음과 같은 문제가 발생하였습니다. 이는 파일을 정리(compact)하기 위해서 전체를 재성하기 때문으로 

```markdown
**1. 디스크 I/O 폭증**

- 큰 기존 AOF 파일을 통째로 다시 써야함
- 디스크 write amplification 심각
- SSD 수명 문제 발생 가능

즉 rewrite 중 부모가 이중(기존 AOF, rewrite buffer)으로 기록하고,
rewrite 완료 시 rewrite buffer를 새 AOF에 추가로 붙이는 과정에서 추가 I/O가 발생하는 것
```

멀티 파트 AOF 기능으로 AOF 단일 파일을 다음과 같은 구성으로 분리합니다. 여기서 주요한 차이는 **기존 AOF 파일을 merge하지 않는다**는 점입니다. 즉 7.0 이전에는 rewrite 중 발생한 변경을 rewrite buffer에 모았다가 새 AOF에 “붙이는 과정”이 필요했습니다.
7.0 이후에는 그걸 별도 INCR 파일에 바로 기록하고, rewrite 완료 시 새 BASE와 병합하지 않는다.(전체 메모리를 scan해서 저장하는 건 같음)

1. BASE AOF : rewrite로 생성된 기준 파일
2. INCREMENTAL AOF : 이후에 write가 append되는 파일들
3. MANIFEST 파일 : 어떤 AOF 파일들이 유효한지 관리(MANIFEST만 교체하여 쉽게 BASE는 폐기)

**rewrite buffer 병합 단계 제거**

> rewrite 완료 시점에 대량 데이터를 다시 붙이는 단계가 없어졌다는 것

- rewrite 중 변경을 INCR 파일에 직접 기록
- rewrite 완료 시 새 BASE에 붙이는 단계 없음
- 마지막 대량 flush 제거
- 이미 변경은 INCR에 append 되었고 BASE 생성 완료 후 메타데이터 전환만 수행 
- → I/O spike 완화(쓰기 패턴 균등)

그럼 rewrite buffer는 완전히 없나?

- 내부적으로 클라이언트 write를 처리하는 AOF 버퍼는 존재
- 하지만 “rewrite 전용 병합 버퍼” 개념은 사라졌다 

## 스냅숏 + AOF

AOF가 활성화되어 있으면:

- Redis restart 시 AOF 파일을 우선 사용 (AOF가 더 일관성 유지)
- RDB 파일은 무시됨

옵션:
```text
aof-use-rdb-preamble yes
```

이 경우:
- AOF 파일 앞부분은 RDB snapshot 형식 
- 이후는 incremental AOF 로그

장점:
- 빠른 로딩 (RDB 기반)
- 낮은 데이터 손실 (AOF 기반)

## 실무 선택 기준

위 개념들을 바탕으로 아래와 같은 전략을 구성해 볼 수 있습니다.

| 방식 | 개념 |
|------|------|
| 캐시 전용  | RDB 또는 persistence off | 
| 세션 저장  | RDB + AOF (everysec) | 
| 중요 데이터 | AOF everysec + replica + 외부 백업 |
| 금융/주문 시스템 | AOF + 복제 + 외부 스냅샷 필수 |

## 실제 장애 원인 Top 3

1. fork latency spike
2. CoW 메모리 폭증 → OOM
3. 디스크 I/O saturation

Persistence는 단순히 “데이터를 남기는 기능”이 아닙니다. CPU, 메모리, 디스크, latency 특성을 모두 건드리는 시스템 레벨 기능입니다.
