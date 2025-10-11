---
title: 홈서버 구성 일지 1
description: >-
  홈서버에 k8s 구성하기 1편
author: jay
date: 2025-10-01 20:55:00 +0800
categories: [Study, Report]
tags: [k8s, devops]
pin: false
media_subpath: ''
---

## 간단한 k8s 개념 체크

- **kubelet :** 노드에서 Pod 상태를 원하는 상태로 유지(스케줄러가 배치한 Pod이 실제로 뜨고 건강한지 확인).
  - API Server와 통신 → PodSpec을 받아 컨테이너 런타임에 실행/중지 지시 → Liveness/Readiness/Startup Probe 체크, 로그/이벤트 보고.
  - “마스터의 명령”이라기보다 클러스터의 선언적 상태(API Server)에 맞춰 조정(Reconcile) 하는 에이전트.
  - 노드 리소스(캐파, 디스크, 볼륨 마운트, CSI/CNI 연동 일부) 관여.
- **kube-proxy :** “서비스 트래픽 분배기” — Service IP 유지, iptables/IPVS 규칙으로 로드밸런싱.
  - Service/Endpoint 변화를 감지 → iptables 또는 IPVS 규칙 생성/유지 → ClusterIP/NodePort/ExternalTrafficPolicy 등 처리.
- **Container Runtime :** “실행 엔진” — 컨테이너를 진짜로 돌리고 멈춤
  - Docker는 과거 사용했으나 dockershim 제거 이후 일반적으로 containerd를 직접 사용.
  - kubelet ↔ 런타임 간 CRI(Container Runtime Interface) gRPC 프로토콜 사용.
  - 이미지 풀, 컨테이너 라이프사이클, 로그/STDIO, CSI/CNI와의 실무적 연계 수행.

## CNI

> Kubernetes에서 kubelet이 Pod를 띄울 때 CNI 플러그인을 호출해 네트워크 인터페이스 생성, IP 할당, 라우팅/캡슐레이션 등을 처리


### 현재 CNI 설정 정보

현재 사용중인 CNI Plugin : Flannel

```yaml
# root@node1:~# kubectl -n kube-flannel get cm kube-flannel-cfg -o yaml
apiVersion: v1
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "EnableNFTables": false,
      "Backend": {
        "Type": "vxlan"
      }
    }
kind: ConfigMap
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"cni-conf.json":"{\n  \"name\": \"cbr0\",\n  \"cniVersion\": \"0.3.1\",\n  \"plugins\": [\n    {\n      \"type\": \"flannel\",\n      \"delegate\": {\n        \"hairpinMode\": true,\n        \"isDefaultGateway\": true\n      }\n    },\n    {\n      \"type\": \"portmap\",\n      \"capabilities\": {\n        \"portMappings\": true\n      }\n    }\n  ]\n}\n","net-conf.json":"{\n  \"Network\": \"10.244.0.0/16\",\n  \"EnableNFTables\": false,\n  \"Backend\": {\n    \"Type\": \"vxlan\"\n  }\n}\n"},"kind":"ConfigMap","metadata":{"annotations":{},"labels":{"app":"flannel","k8s-app":"flannel","tier":"node"},"name":"kube-flannel-cfg","namespace":"kube-flannel"}}
  creationTimestamp: "2025-09-15T09:22:43Z"
  labels:
    app: flannel
    k8s-app: flannel
    tier: node
  name: kube-flannel-cfg
  namespace: kube-flannel
  resourceVersion: "293478"
  uid: c1f8e69e-8713-459c-bf95-9189a13f5fa2
```

### 한줄 정리
- Pod 대역: 10.244.0.0/16
- 백엔드: vxlan 오버레이(노드 간 캡슐화, 인터페이스 flannel.1)
- CNI 체인: flannel → portmap
- 게이트웨이/헤어핀: Pod의 기본 GW는 노드 브리지(cni0), hairpin 허용
- NAT 프레임워크: EnableNFTables: false → iptables 경로 사용

### 용어 정리

- **hairpin** : 컨테이너/Pod가 **자기 자신이 위치한 같은 노드의 Service(IP/포트)** 를 통해 다시 자신의 Pod로 트래픽을 보낼 수 있게 해 주는 기능
  - Pod 기본 게이트웨이가 cni0이므로 Pod↔Pod(동일 노드) 는 L2 브리징으로 처리
  - Pod↔외부/상이노드는 라우팅/VXLAN 경로
- **vxlan** : 다른 노드 Pod로 가는 패킷을 L2 프레임 → UDP 캡슐화(표준 VXLAN) 해서 물리망 위로 전송. 수신 노드가 디캡슐 → 대상 Pod로.
  - 패킷 플로우 : Pod veth → cni0 → (라우팅) → flannel.1(VXLAN 캡슐화) → 물리 NIC → 원격 노드 flannel.1 → cni0 → 대상 Pod
  - MTU: 캡슐화 오버헤드(~50B) 때문에 Pod MTU ≈ NIC-50 (1500 NIC면 1450 권장).
불일치 시 대용량 패킷 손실/세그멘테이션 → gRPC/DB 지연/타임아웃 유발.
- **CNI 체인** : 나의 Pod 네트워크 인터페이스를 만들 때 여러 개의 CNI 플러그인을 “순서대로” 실행해서 기능을 합치는 방식
  - portmap : Pod의 hostPort/포트 매핑을 위해 iptables DNAT/SNAT 규칙을 추가.
  - hostPort를 많이 쓰면 노드 포트 충돌/보안 정책을 신중히 관리.
- **NAT Framework** : Flannel/portmap/kube-proxy가 iptables 테이블(nat, mangle, filter)에 규칙을 쌓는다.
  - 서비스/엔드포인트가 많아질수록 iptables 체인 규모 ↑ → 규칙 동기화 지연/캐싱 비용 ↑.

### 종합 패킷 플로우

> MTU/iptables 규모가 성능·안정성의 핵심 변수

**A) 같은 노드 Pod → Pod**
1. PodA → PodB IP
2. veth → cni0 브리지 내 L2 스위칭 → PodB veth (hairpin 여부와 무관)

**B) 다른 노드 Pod → Pod**
1. PodA → PodB IP
2. cni0 → 라우팅 테이블에서 대상 서브넷이 원격 노드 VTEP으로 매핑
3. flannel.1 VXLAN 캡슐화 → 물리망 → 원격 flannel.1 디캡슐 → cni0 → PodB

**C) Pod → ServiceIP**
1. kube-proxy(iptables/IPVS)가 ServiceIP를 백엔드 Pod IP로 DNAT/리다이렉션.
2. 같은 노드에서 자기 자신을 ServiceIP로 호출해도 hairpin + NAT로 정상 동작.

## Load Balancer

> 로드밸런싱 : 애플리케이션을 지원하는 리소스 풀 전체에 네트워크 트래픽을 정해진 규칙에 따라 균등 혹은 sticky하게 배포하는 방법

현재 홈서버는 총 3대의 서버로 구성되어 있으며 각 서버는 하나의 공유기 아래 같은 게이트웨이를 통해 연결되어 있음.
따라서 metal-lb를 사용하여 lb를 구성할 수 있도록 설정하였다.(CNI: Flannel)

> metal-lb : 베어메탈 환경에서 Kubernetes LoadBalancer 기능을 제공하는 오픈소스 프로젝트

### 트래픽 경로

> 요점: MetalLB는 “노드까지” 책임, CNI는 “노드 안에서 Pod까지” 책임.

1. 사용자가 Service(type: LoadBalancer) 생성 → MetalLB가 IP 풀에서 LB IP 할당
2. MetalLB가 LB IP를 외부에 광고
   1. L2 모드: 스피커가 ARP/NDP 응답으로 “LB IP는 이 노드 MAC”이라고 알림(활성-백업 또는 ECMP).
   2. BGP 모드: 스피커가 업스트림 라우터에 /32(/128) 라우트로 LB IP를 BGP로 광고.
3. 외부 클라이언트 → LB IP로 패킷 전송 → L2/BGP 결과에 따라 특정 노드(들)로 도착
4. 노드에서 **kube-proxy(ipvs/iptables) 또는 eBPF LB(Cilium/Calico eBPF 등)**가 서비스 백엔드 Pod로 선택.
5. CNI 데이터플레인이 실제로 패킷을 **그 Pod 네임스페이스(veth/터널/RPF/정책)**까지 운반.
6. 응답은 CNI 경로로 노드를 거쳐 외부로 회신(필요 시 SNAT/정책 적용).

여기서 광고란 '외부 네트워크에 “이 IP로 가려면 이쪽으로 보내라”는 도달 가능성 정보를 퍼뜨리는 행위'
노드→Pod로 운반하는 건 CNI가 맡음