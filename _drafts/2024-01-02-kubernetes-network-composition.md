---
title: 쿠버네티스 네트워크 구성 정리
author: mingo
date: 2024-01-02 00:30:00 +0900
categories: [Kubernetes]
tags: [network, cni]
pin: false
---

-----------------

쿠버네티스 클러스터 환경을 직접 구축하다 보면 Pod 사이의 통신 방법에 대해 생각하게 된다. 구글링을 하다보면 CNI라는 용어를 자주 접하게 되는데, CNI가 무엇인지에 대해 정리해보고자 한다.

## 시작

## 네트워크 구성
1. Pod 내부에 구성된 컨테이너 간의 통신
2. 싱글 노드 Pod 간의 통신
3. 멀티 노드 Pod 간의 통신
4. Pod와 Service 간의 통신
5. 외부와 Service 간의 통신


## CNI(Container Network Interface) 란
**CNI**는 CNCF(Cloud Native Computing Foundation)의 프로젝트 중 하나로, 컨테이너 런타임과 네트워크를 연결하는 플로그인 위한 표준 인터페이스이다.  
다양한 컨테이너 런타임과 오케스트레이터 사이에서 네트워크 계층을 구현하는 방식이 달라서 표준화된 인터페이스가 필요했고, 이를 위해 CNI가 만들어졌다.  
따라서 표준 인터페이스로써 단순히 쿠버네티스 뿐만 아니라 Amazone ECS, Azure AKS, Cloud Foundry 등 다양한 컨테이너 런타임에서 사용할 수 있다.    

쿠버네티스는 기본적으로 `kubenet`이라는 자체 CNI 플러그인을 사용한다.

## CNI 종류

### Kubenet

### Calico

### Flannel

### Weave Net

## CNI 네트워크 모델

### VXLAN(Virtual EXtensible Local Area Network)

### IP-in-IP

### BGP(Border Gateway Protocol)

## Container

### pause 컨테이너
하나의 Pod가 생성되면 Pod 하위의 컨테이너들은 **pause 컨테이너**에 의하여 모두 동일한 네트워크 네임스페이스(veth0)를 공유한다. 
동일한 네트워크 네임스페이스를 공유하는 컨테이너들은 동일한 IP 주소를 가지게 되는데, 이는 Pod 내부에서 `localhost(127.0.0.1):port`로 통신할 수 있도록 하며 서로를 port로 구분할 수 있게 해준다.  
**따라서 Pod 내부의 컨테이너는 서로 다른 port를 사용해야만 한다.**
  
그러나 여러 노드로 구성된 클러스터의 경우 Pod 간의 연결이 구성되어 있지 않기 때문에 Pod 간의 통신이 불가능하다. 이것을 해결하기 위해 CNI를 구현한 CNI 플러그인이 등장하였다.

pod -> veth1 -> CNI -> eth0 -> 라우터 VPC -> eth0 -> CNI -> veth0 -> pod 
## 정리
