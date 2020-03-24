---
title:  "Kubernetes Pods 소개"
excerpt: "Kubernetes의 Pod 개념을 알아보자"

categories:
  - kubernetes
author: MinSik Kim
tags:
  - kubernetes
  - pods
  - pod
  - docker
  - 쿠버네티스
classes: wide
last_modified_at: 2020-03-20T15:00:00
---

## Kubernets pod의 정의

Docker 세계에서 스케쥴링되는 가장작은 단위는 container이지만 쿠버네티스 세계에서는
Pod이다. 쿠버네티스가 컨테이너화된 어플리케이션을 실행하는건 맞지만 직접적으로 쿠버네티스
클러스터에서 컨테이너를 실행하는것은 아니다. 컨테이너들은 반드시 Pod 안에서 실행되어야 한다.
조금 더 자세히 살펴보면 Pod안에 있는 하나 혹은 여러개의 컨테이너들은 같은 네트워크 스택 및 Kernel Namespace 등의 실행환경을 공유하고 있다. 이 의미는 같은 Pod안에 있는 컨테이너들은 아래 그림과 같이 하나의 IP를 공유하고 있는 것이다. 만약 같은 Pod안에서 컨테이너간의 통신이 필요하다면 localhost 주소에 port를 사용하면 된다.
![Pod안에서의 네트워크)](/files/kubernetes-pods.png)

### Pod - 스케쥴링 & 스케일링 단위
Pod는 Kubernetes에서 스케쥴링되는 가장 작은 단위이자 스케일링되는 작은 단위이다. 쿠버네티스에서 배포한 어플리케이션이 스케일링된다는 뜻은 Pod안에 있는 컨테이너가 추가 혹은 삭제되는 것이 아니라 Pod이 추가 혹은 삭제 되는 것이다. Pod 단위로 스케쥴링 및 스케일링 되야 하는 이유를예를 들어 설명해보자. 2개의 컨테이너가 메모리를 공유하는 어플리케이션으로 구현되어 있다면 만약 컨테이너들이 각각 다른 호스트로 스케쥴링된다면 Pod는 제대로 동작하지 못할 것이다.
![Pod - 스케일링 단위)](/files/kubernetes-pods-scale.png)
### Pod Deploy
쿠버네티스 클러스터에서 Pod를 배포하기 위해서는 먼저 manifest 파일을 정의한 후 manifest파일을 쿠버네티스 API 서버로 POST호출을 해야한다. API 서버는 호출을 받은 뒤 manifest 파일을 검사 후 올바르게 정의되어 있다면 etcd에 레코드로 저장하게 된다. 그리고 스케쥴러는 가용한 Worker노드를 찾아 Pod를 배포하게 된다(나중에 살펴보겠지만 실제 Pod를 배포하는 주체는 `kubelet`이다).
