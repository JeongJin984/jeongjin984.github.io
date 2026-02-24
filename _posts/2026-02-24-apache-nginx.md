---
title: 웹서버 nginx 와 apache
description: >-
  apache와 nginx에 대해서
author: jay
date: 2026-02-04 20:55:00 +0800
categories: [Study, server]
tags: [server, study]
pin: false
media_subpath: ''
---

# Nginx

> 고성능 **웹 서버(Web Server)**이자 리버스 프록시(Reverse Proxy), 로드 밸런서(Load Balancer), HTTP 캐시 서버

- 이벤트 기반(event-driven), 비동기(asynchronous) 아키텍처를 채택해 높은 동시성 처리에 강함

## 0. Nginx 구조

> NGINX는 멀티프로세스 + 이벤트 기반(non-blocking) + 모듈형 상태 머신 구조
> - 핵심 : Master가 관리하고, Worker가 epoll 이벤트 루프에서 모든 연결을 처리한다.

Nginx의 프로세스 구조는 아래의 형태를 띈다.

```text
                +------------------+
                |  Master Process  |
                +------------------+
                   |     |     |
        ------------     |     ------------
        |                |                |
+---------------+ +---------------+ +---------------+
| Worker Proc 1 | | Worker Proc 2 | | Worker Proc N |
|  (1 thread)   | |  (1 thread)   | |  (1 thread)   |
|  event loop   | |  event loop   | |  event loop   |
+---------------+ +---------------+ +---------------+
```

### 0.1 Master Process 역할

> Master는 데이터 처리 안 한다. 관리만 한다. 

- 설정 파일 로딩 및 파싱 
- worker 프로세스 생성 
- graceful reload 
- worker 장애 감지 및 재시작 
- signal 처리

### 0.2 Worker Process 구조

- 단일 스레드 
- epoll 기반 이벤트 루프 
- non-blocking 소켓 처리 
- HTTP 상태 머신 실행

개념적으로 아래의 형태로 동작한다.

```text
while (1) {
    events = epoll_wait(...)
    process_events(events)
    process_timers()
}
```

## 1. NIC에 패킷 도착

> **물리계층**

- 클라이언트가 보낸 TCP 세그먼트가 서버 NIC에 도착 
- NIC DMA 엔진이 패킷을 커널 메모리의 RX ring buffer에 적재

> 인터럽트 / NAPI
   - NIC가 인터럽트 발생 
   - Linux는 NAPI(New API)로 인터럽트 과다를 방지 
   - softirq(NET_RX)에서 패킷 처리 시작

## 2. 커널 네트워크 스택 처리

> **L2 → L3 → L4 처리** : 이 과정은 모두 커널 공간에서 수행됩니다.

- Ethernet header 제거 
- IP header 검사 
- TCP 세그먼트 재조립 
- ACK 처리 
- 윈도우 업데이트

## 3. 소켓 수신 버퍼 적재

> 커널은 4-tuple로 소켓 매칭
- (src_ip, src_port, dst_ip, dst_port)

> sk_buff → socket receive queue

- 데이터가 sk_buff 형태로 저장 
  - sk_buff(socket buffer): Linux 커널 네트워크 스택에서 패킷을 표현하는 핵심 메타데이터(포인터, 길이, 상태, 프로토콜) 구조체
- 해당 소켓의 receive buffer에 enqueue 
- 이 시점에서 소켓은 “readable 상태”

### 3.1 네트워크 소켓

> 네트워크 소켓은 “파일처럼 보이는 파일 디스크립터”일 뿐이고, 실제로는 소켓 파일에 쓰는 게 아니라 **커널의 소켓 버퍼에 enqueue**되는 것

Unix 철학에서 "소켓은 파일이다."의 의미
- 파일, 파이프, 소켓, 터미널 모두 file descripter(FD)로 다룬다.

| 항목         | 일반 파일              | 소켓                   |
| ---------- | ------------------ | -------------------- |
| 저장 위치      | 디스크                | 커널 메모리               |
| 지속성        | 영구                 | 휘발성                  |
| 구조         | inode + page cache | socket 구조체 + 버퍼      |
| read() 의미  | 디스크에서 읽기           | receive queue에서 복사   |
| write() 의미 | 디스크에 기록            | send buffer에 enqueue |


즉: 아래와 같이 동작한다.
```text
int fd = socket(...)
read(fd, ...)
write(fd, ...)
```

### 3.1 파일 디스크립트

> FD는 파일이 아니라 커널 내부 객체를 가리키는 인덱스(“프로세스가 열어둔 I/O 객체를 가리키는 정수 핸들”)다.

- FD는 단순히 “번호”다.(진짜 객체는 커널에 있다.)

**흐름 예시**
```text
int fd = open("a.txt", O_RDONLY);
```

1. 커널이 struct file 생성 
2. 프로세스의 FD 테이블에서 빈 슬롯 찾음 
   - 예: 3번 슬롯에 struct file* 저장
3. 사용자에게 3 반환

즉:
- fd = 3 은 “FD 테이블의 3번 인덱스”를 의미

## 4. epoll이 깨는 순간

NGINX worker는 다음 상태로 대기 중: epoll_wait(epfd, events, ...)

> epoll wakeup
> - epoll이 알려주는 건 “패킷이 enqueue됨” 그 자체가 아니라, 결과적으로 소켓이 read 가능한 상태라는 사실입니다. 
> - (TCP 스트림이면 “패킷” 단위가 아니라 바이트 스트림으로 누적)

- 커널이 해당 FD에 EPOLLIN 이벤트 설정
- worker 프로세스 깨움
- epoll_wait 반환

이제 제어가 user space(NGINX worker)로 이동.

## 5. NGINX worker 이벤트 처리

> 이벤트 루프 진입
> - 이벤트 루프는 handler dispatch 역할

### 5.1 worker 핵심 루프

> 이벤트 루프는 커널로부터 “어떤 FD가 준비되었는지” 어떻게 통지받는가?
> - epoll_wait()를 통해 커널의 ready list를 가져옵니다.

epoll의 내부 구조:
- interest list : 우리가 감시하겠다고 등록한 FD 목록
- ready list : 우리가 감시하겠다고 등록한 FD 목록

```text
int n = epoll_wait(epfd, events, maxevents, timeout);
``` 

이 함수가 하는 일
1. ready list가 비어있으면 → sleep
2. 네트워크 이벤트 발생
3. 커널이 해당 FD를 ready list에 추가
4. epoll_wait()가 깨어남
5. 준비된 FD 배열을 user space로 복사
6. n(이벤트 개수) 반환
7. 각 fd에 연결된 ngx_event_t를 꺼냄
8. event->handler(event) 호출

### 5.3 왜 이렇게 설계했는가?

각 단계마다 handler가 바뀐다:
```text
rev->handler = ngx_http_process_request_line;
rev->handler = ngx_http_process_request_headers;
rev->handler = ngx_http_upstream_process_header;
```

이 구조 덕분에:
- if/else 상태 분기 대신 
- 함수 포인터 교체로 상태 전이 구현

그리고 각 상태에서 필요할 때만 recv()를 호출한다.(하나의 HTTP 요청은 여러 핸들러를 단계적으로 거친다.)

## 6. read() 호출

> 이벤트 루프는 read 이벤트의 handler를 호출하고, 그 handler 내부에서 recv()가 호출
> - non-blocking read
> - NGINX가 호출 : recv(fd, buffer, ...)

- 커널 receive buffer에서 user buffer로 복사
- 더 읽을 데이터 없으면 → EAGAIN 
- 블로킹 없음

### 6.1 Nginx의 핸들러

Nginx에는 서로 다른 레벨의 핸들러가 있다.

1. Connection / Event 레벨 핸들러(I/O 상태 머신) : 소켓 I/O 상태에 따라 바뀌는 함수들:
2. HTTP Phase 핸들러 : 요청이 완성되면 실행되는 phase engine 기반 핸들러 체인(각 phase에는 여러 모듈이 핸들러를 등록한다.)

핸들러 호출에 따라 FD readiness 상태 변화(커널 상태)가 발생하고 그 변화를 다음 epoll_wait()에서 반영하는 것이다.

### 6.2 핸들러 호출 흐름

> 커널이 receive 소켓의 readiness가 변경하면 EPOLLIN이 발생하고 이에 따라 read 핸들러를 이벤트 루프가 호출한다. 
> - read 핸들러가 처리 과정에서 EPOLLOUT을 interest list에 등록해서 다음 epoll_wait에서 write 핸들러를 호출

정확히는
1. 커널이 소켓 recv buffer에 데이터 적재
2. 해당 FD가 readable 상태가 됨
3. (interest list에 EPOLLIN이 등록돼 있다면)
4. ready list에 추가
5. epoll_wait()가 반환
6. 이벤트 루프가 read 핸들러 호출(여기서 EPOLLOUT을 interest list에 등록)

그리고

1. read 핸들러가 write가 필요하다고 판단하면 EPOLLOUT을 interest list에 등록한다.
2. 그 FD가 writable 상태라면 다음 epoll_wait에서 EPOLLOUT이 반환된다.

```text
handle_read(fd) {
    n = read(fd, buf, ...);
    append_to_peer_write_buffer(buf, n);

    if (peer_has_pending_write)
        enable_epollout(peer_fd);
}
```

> read 핸들러 안에서, 애플리케이션이 직접 epoll_ctl(EPOLL_CTL_MOD, ...)을 호출합니다.

## 7. HTTP 상태 머신 진입

이제 바이트 스트림이 HTTP 모듈로 전달됨.

> 상태 전이 : 파싱은 점진적(incremental)으로 이루어짐

```text
STATE: READ_REQUEST_LINE
STATE: READ_HEADERS
STATE: READ_BODY
```

## 8. 요청 완성

- 헤더 파싱 완료 시:
  - location 매칭 
  - rewrite phase 
  - access phase 
  - content phase

이것은 NGINX의 phase engine이라고 부릅니다.

## 9. 정적 파일 vs upstream

**정적 파일**

1. open()
2. sendfile()
3. write 이벤트 등록 
4. EPOLLOUT 발생 시 전송

**upstream proxy**

1. non-blocking connect()
2. epoll에 upstream FD 등록 
3. 요청 write 
4. upstream read 
5. client write

## 10. 응답 전송 경로(반대로)

> send() 또는 sendfile()

1. user space → kernel socket send buffer 
2. TCP segmentation 
3. NIC TX ring buffer 
4. DMA → wire

write가 EAGAIN이면:
- EPOLLOUT 등록 
- 다음 루프에서 재시도

## 11. keepalive 또는 종료

- Connection: keep-alive → 타이머 등록 
- 아니면 close 
- FD epoll에서 제거
