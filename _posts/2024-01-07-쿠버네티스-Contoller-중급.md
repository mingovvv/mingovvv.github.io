---
title: ☸️ 쿠버네티스 Contoller 오브젝트 자세히 살펴보기
author: mingo
date: 2024-01-08 00:30:00 +0900
categories: [Kubernetes, Object]
tags: [controller, kubernetes, k8s]
#image:
#  path: /path/to/image
#  alt: image alternative text
---

-----------------
![33.png](/assets/img/post/202401/33.png){: .border w="800" h="550" }
_stateless application과 stateful application_

쿠버네티스에서 Controller 오브젝트는 다양한 다른 오브젝트들을 관리하고 원하는 환경을 유지할 수 있도록 도움을 주는 오브젝트이며 쿠버네티스의 핵심 기능 중 하나이다.
중요한 것은 Contoller 오브젝트는 사용자가 원하는 환경을 **유지**시켜 준다는 것이고 이것은 쿠버네티스 클러스터의 상태를 지속적으로 관리한다는 의미이다.
실제로 공식 문서에서도 Controller 오브젝트를 "**실내 온도 조절기**"로 비유하고 있으니 문자 그대로 이해하기 쉽게 생각하면 된다.

Pod는 Controller 오브젝트에 의해서 관리되며 Controller 오브젝트는 사용자가 정의한대로 Pod 오브젝트를 생성, 삭제, 유지 및 모든 라이프사이클에 관여하며 Pod의 상태를 관리한다.
 
Pod는 병렬로 확장이 가능한 **stateless application**과 Pod마다 역할군이 나뉘어져 있는 **stateful application**으로 구분할 수 있다. 

**stateless application**냐 **stateful application**냐 따라 적용해야 하는 Controller 오브젝트가 다르다.

- kubedoc - Contoller : <https://kubernetes.io/ko/docs/concepts/architecture/controller/>

![34.png](/assets/img/post/202401/34.png){: .border w="500" h="650" }
_ReplicaSet과 StatefulSet_

## 1. ReplicaSet
ReplicaSet 컨트롤러 오브젝트를 통해 병렬처리가 가능한 **stateless application**을 관리할 수 있다. 
Pod명은 랜덤으로 생성되고 replicas의 수만큼 scale-out이 진행될 때 한번에 Pod가 기동되는 것이 특징이다.

```yaml
# ReplicaSet 생성
apiVersion: apps/v1
kind: ReplicaSet # ReplicaSet
metadata:
  name: replica-web
spec:
  replicas: 1 # Pod 유지 개수
  selector:
    matchLabels:
      type: web
  template:
    metadata:
      labels:
        type: web
    spec:
      containers:
        - name: container
          image: kubetm/app
      terminationGracePeriodSeconds: 10 # 10s 후 삭제
```

## 2. StatefulSet
반면 StatefulSet 컨트롤러 오브젝트는 **stateless application**을 관리를 위해 탄생했다.
Pod명은 index를 통해 순차적으로 증가되고 문제가 발생한 Pod가 삭제되었다가 재기동되면 삭제된 Pod명을 그대로 이어서 사용된다.

```yaml
# StatefulSet 생성
apiVersion: apps/v1
kind: StatefulSet # StatefulSet
metadata:
  name: stateful-db
spec:
  replicas: 1 # Pod 유지 개수
  selector:
    matchLabels:
      type: db
  template:
    metadata:
      labels:
        type: db
    spec:
      containers:
        - name: container
          image: kubetm/app
      terminationGracePeriodSeconds: 10 # 10s 후 삭제
```

### > ReplicaSet vs StatefulSet
![36.png](/assets/img/post/202401/36.png){: .border w="800" h="650" }
_PersistentVolumeClaim 사용 시 비교_

![37.png](/assets/img/post/202401/37.png){: .border w="800" h="350" }
_Headless Service 사용 시 연결과정_

## Ingress
![35.png](/assets/img/post/202401/35.png){: .border w="800" h="550" }
_Ingress 오브젝트_
`Ingress` 오브젝트는 서비스 오브젝트 로드밸런싱과 카나리 업그레이드에서 주로 사용된다.

### 1. Service Object Load Balancing
![39.png](/assets/img/post/202401/39.png){: .border w="550" h="800" }
_Service Object Load Balancing_
`Ingress` 오브젝트는 Host 도메인 그리고 path와 Service 오브젝트를 매핑하는 것으로 간단한 룰을 작성할 수 있다.
이를 통해 해당 도메인으로 접근하는 트래픽을 path에 따라 구분하여 매핑되는 Service 오브젝트로 트래픽을 넘기는 것이다.

하지만 생성한 `Ingress` 오브젝트를 정상적으로 동작시키려면 구현체 플러그인을 설치해야만 한다. 
이러한 플러그인을 `Ingress Controller`라고 하며 대표적으로 NGINX와 KONG이 있다.

`Ingress Controller`의 플러그인 중 하나를 선택하여 설치하면 네임스페이스가 생성되고 그 안에 Deployment와 ReplicaSet, Ingress 오브젝트의 구현체인 Pod가 만들어진다.
구현체 Pod는 `Ingress` 오브젝트의 룰이 있는지 확인하고 룰에 맞춰서 Service 오브젝트와 연결을 수행한다.  
 
**외부에서 접근하는 사용자의 트래픽을 구현체 Pod와 연결하기 위해 프론트 Service 오브젝트가 필요하다. 
최종적으로 구현체 Pod가 Ingress 룰을 통해 path값에 맞춰 매핑시킨 Service 오브젝트로 로드밸런싱을 해야 하므로 구현체 Pod와 프론트 Service 오브젝트의 연결은 필수이다.**

```shell
# `Ingress Controller`의 플러그인 NginX 설치 
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/baremetal/deploy.yaml
```
 - `Ingress Controller`의 플러그인 Nginx를 설치하면 프론트 Service 오브젝트가 자동으로 생성된다.
 - 자동으로 생성된 Service 오브젝트 NodePort를 통해 외부에서 접근할 수 있다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-shopping
  labels:
    category: shopping
spec:
  containers:
  - name: container
    image: kubetm/shopping
---
apiVersion: v1
kind: Service
metadata:
  name: svc-shopping
spec:
  selector:
    category: shopping
  ports:
  - port: 8080
```
 - Ingress 로드밸런싱을 통해 트래픽 분산을 전달받을 Service와 Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-customer
  labels:
    category: customer
spec:
  containers:
  - name: container
    image: kubetm/customer
---
apiVersion: v1
kind: Service
metadata:
  name: svc-customer
spec:
  selector:
    category: customer
  ports:
  - port: 8080
```
 - Ingress 로드밸런싱을 통해 트래픽 분산을 전달받을 Service와 Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-order
  labels:
    category: order
spec:
  containers:
  - name: container
    image: kubetm/order
---
apiVersion: v1
kind: Service
metadata:
  name: svc-order
spec:
  selector:
    category: order
  ports:
  - port: 8080
```
 - Ingress 로드밸런싱을 통해 트래픽 분산을 전달받을 Service와 Pod

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress # Ingress 오브젝트
metadata:
  name: service-loadbalancing
spec:
  ingressClassName: nginx
  rules: # 룰 설정
  - http:
      paths:
      - path: / # path
        pathType: Prefix
        backend:
          service:
            name: svc-shopping # 'svc-shopping' 이라는 Service 오브젝트에게 위임 
            port:
              number: 8080 # port
      - path: /customer
        pathType: Prefix
        backend:
          service:
            name: svc-customer
            port:
              number: 8080
      - path: /order
        pathType: Prefix
        backend:
          service:
            name: svc-order
            port:
              number: 8080
```
 - `Ingress` 오브젝트 정의 (매핑 룰을 지정하는 것과 동일하다.)
 - **실제 동작하는 것은 Ingress Controller 플러그인(Nginx)의 구현체 Pod가 정의된 매핑 룰에 따라 트래픽이 처리되는 것이다.**
 - Nginx를 설치할 때, 프론트 Service 오브젝트가 NodePort로 생성되며 구현체 Pod에 연결되어 있어 외부에서 접근할 수 있다.

### 2. Canary Upgrade
![38.png](/assets/img/post/202401/38.png){: .border w="550" h="800" }
_Canary Upgrade_

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-v1
  labels:
    app: v1
spec:
  containers:
  - name: container
    image: kubetm/app:v1
---
apiVersion: v1
kind: Service
metadata:
  name: svc-v1
spec:
  selector:
    app: v1
  ports:
  - port: 8080
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress # Ingress 오브젝트
metadata:
  name: app
spec:
  ingressClassName: nginx
  rules: # 룰 설정
  - host: www.app.com # 호스트명 설정...!
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: svc-v1
            port:
              number: 8080
```

```yaml
# Centos HostName 등록
cat << EOF >> /etc/hosts
192.168.4.4 www.app.com
EOF

curl www.app.com:30431/version
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-v2
  labels:
    app: v2
spec:
  containers:
  - name: container
    image: kubetm/app:v2
---
apiVersion: v1
kind: Service
metadata:
  name: svc-v2
spec:
  selector:
    app: v2
  ports:
  - port: 8080
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: canary-v2
  annotations:
    nginx.ingress.kubernetes.io/canary: "true" # 카나리 테스트를 하기 위한 옵션
    nginx.ingress.kubernetes.io/canary-weight: "10" # 10% 트래픽 처리
spec:
  ingressClassName: nginx
  rules:
  - host: www.app.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: svc-v2
            port:
              number: 8080
```
 - 트래픽 분산 확인 : `while true; do curl www.app.com:30431/version; sleep 1; done`

### 3. Https 인증서
![40.png](/assets/img/post/202401/40.png){: .border w="550" h="800" }
_Https 인증서_

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-https
  labels:
    app: https
spec:
  containers:
  - name: container
    image: kubetm/app
---
apiVersion: v1
kind: Service
metadata:
  name: svc-https
spec:
  selector:
    app: https
  ports:
  - port: 8080
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress # Ingress 오브젝트
metadata:
  name: https
spec:
  ingressClassName: nginx
  tls: # tls 속성
  - hosts:
    - www.https.com
    secretName: secret-https # 시크릿 지정
  rules:
  - host: www.https.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: svc-https
            port:
              number: 8080
```

```shell
# 인증서 생성
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=www.https.com/O=www.https.com"

# Secret 생성
kubectl create secret tls secret-https --key tls.key --cert tls.crt

# Windows HostName 등록
파일 위치 : C:\Windows\System32\drivers\etc\hosts
192.168.4.4 www.https.com

# 브라우저에서 접속
https://www.https.com:30798/hostname
```

## Autoscaler

### HPA
![41.png](/assets/img/post/202401/41.png){: .border w="800" h="550" }
_HPA_
Pod의 개수를 늘려주는 **HPA(Horizontal Pod Autoscaler)**는 컨트롤러와 연결되어 replicas를 조절하며 Pod의 자원을 적절하게 사용하도록 도와주는 기능이다.
병렬 확산이 가능한 stateless application에 주로 사용된다.

### VPA
Pod의 리소스를 늘려주는 **VPA(Vertical Pod Autoscaler)**는 컨트롤러와 연결되어 Pod 재시작을 통해 Pod의 자원을 Scale-Up 또는 Scale-Down 시키는 기능이다. 
stateful application에 사용되고 

### CA
쿠버네티스 클러스터의 Node를 추가해주는 CA(Cluster Autoscaler)는 AWS, Azure, GCP 등 클라우드 플랫폼과 연결하여 사용할 수 있다.
클라우드 프로바이더를 통해 신규 Node를 추가하고 생성이 미뤄졌던 쿠버네티스 오브젝트들이 Node에 생성된다.

> 출처
- <https://www.inflearn.com/course/%EC%BF%A0%EB%B2%84%EB%84%A4%ED%8B%B0%EC%8A%A4-%EA%B8%B0%EC%B4%88/>
- <https://kubernetes.io/ko/docs/concepts/overview/working-with-objects/kubernetes-objects/>

