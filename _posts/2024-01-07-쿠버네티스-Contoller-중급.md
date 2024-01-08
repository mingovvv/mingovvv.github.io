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
여기서 중요한 것은 원하는 환경을 **유지**시켜 준다는 개념이고 이것은 쿠버네티스 클러스터의 상태를 지속적으로 관리한다는 의미이다.
실제로 공식 문서에서도 Controller 오브젝트를 "**실내 온도 조절기**"로 비유하고 있으니 이해하기 쉽게 생각하면 된다.

Pod는 Controller 오브젝트에 의해서 관리되며 Controller 오브젝트는 사용자가 정의한대로 Pod 오브젝트를 생성, 삭제, 유지 및 모든 라이프사이클에 관여하며 Pod의 상태를 관리한다.

하지만 이렇게 Controller 오브젝트가 Pod에 관여를 한다고 해도 위 스크린샷을 통해 알 수 있듯이 
단순히 Pod를 병렬로 확장하는 scale-out을 통해 애플리케이션 환경을 유지할 수 있는 **stateless application**과 특정 Pod속 컨테이너에 따라 확장시켜야만 하는 **stateful application**이 있다.

**stateless application**냐 **stateful application**냐 따라 적용해야 하는 Controller 오브젝트가 다르다.

![34.png](/assets/img/post/202401/34.png){: .border w="500" h="650" }
_ReplicaSet과 StatefulSet_

## ReplicaSet
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
      terminationGracePeriodSeconds: 10
```

## StatefulSet
반면 StatefulSet 컨트롤러 오브젝트는 **stateless application**을 관리할 수 있다.
Pod명은 index를 통해 순차적으로 증가되고 문제가 발생한 Pod가 삭제되었다가 재기동되면 기존 Pod명을 그대로 사용된다.

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
      terminationGracePeriodSeconds: 10
```

- kubedoc - Contoller : <https://kubernetes.io/ko/docs/concepts/architecture/controller/>

## Ingress
![35.png](/assets/img/post/202401/35.png){: .border w="800" h="550" }
_Ingress 오브젝트_



> 출처
- <https://www.inflearn.com/course/%EC%BF%A0%EB%B2%84%EB%84%A4%ED%8B%B0%EC%8A%A4-%EA%B8%B0%EC%B4%88/>
- <https://kubernetes.io/ko/docs/concepts/overview/working-with-objects/kubernetes-objects/>

