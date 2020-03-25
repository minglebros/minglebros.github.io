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

## Pod - 스케쥴링 & 스케일링 단위
Pod는 Kubernetes에서 스케쥴링되는 가장 작은 단위이자 스케일링되는 작은 단위이다. 쿠버네티스에서 배포한 어플리케이션이 스케일링된다는 뜻은 Pod안에 있는 컨테이너가 추가 혹은 삭제되는 것이 아니라 Pod이 추가 혹은 삭제 되는 것이다. Pod 단위로 스케쥴링 및 스케일링 되야 하는 이유를
예를 들어 설명해보자. 2개의 컨테이너가 메모리를 공유하는 어플리케이션으로 구현되어 있다면 만약 컨테이너들이 각각 다른 호스트로 스케쥴링된다면 Pod는 제대로 동작하지 못할 것이다.
![Pod - 스케일링 단위)](/files/kubernetes-pods-scale.png)
## Pod Deploy
쿠버네티스 클러스터에서 Pod를 배포하기 위해서는 먼저 manifest 파일을 정의한 후 manifest파일을 쿠버네티스 API 서버로 POST호출을 해야한다. API 서버는 호출을 받은 뒤 manifest 파일을 검사 후 올바르게 정의되어 있다면 etcd에 레코드로 저장하게 된다. 그리고 스케쥴러는 가용한 Worker노드를 찾아 Pod를 배포하게 된다(나중에 살펴보겠지만 실제 Pod를 배포하는 주체는 `kubelet`이다). Pod를 k8s 클러스터의 worker노드에 배포하기 위해서는 우선적으로 컨테이너의 image를 다운받고 manifest에 정의한 설정대로 컨테이너는 만들어진다. 배포되는 Pod는 `pending` state에서 모든 리소스가 배포되고 준비되면 Pod의 상태는 `Running` 상태로 이전된다.
![Pod - 배포)](/files/kubernetes-pods-deploy.png)
## Pod - 조금 더 깊숙히..(Deep Dive)
Pod는 실제로는 특별한 타입의 컨테이너(Pause Container)이다. Pod(Pause Container)는 Pod안에서
실행하고 있는 컨테이너들에게 시스템 자원을 공유하는 특별한 컨테이너다. 공유하는 시스템 자원들은 다음과 같다.
- Network Namespace: IP주소, 포트, 라우팅 테이블(Routing table)
- UTS Namespace: 호스트 이름
- IPC Namespace: 유닉스 도메인 소켓(Unix domain socket)

### Pod안의 컨테이너들은 네트워크를 공유한다
Pod가 생성되면 Pod는 자기 소유의 네트워크 네임스페이스를 갖는다. 그러므로 Pod안에 여러 컨테이너가 있다면 컨테이너들은 IP주소, 포트, 라우팅 테이블 등을 공유한다. 밑의 그림에서 보는바와 같이 외부에서 컨테이너와 통신하기 위해서는 공유된 IP주소와 포트`192.168.0.5:80`를 사용해야 한다. 만약 Pod안에 있는 컨테이너끼리 통신하기 위해서는 어떻게 해야할까? 앞서 말한바와 같이 localhost 주소`localhost:5000`에 포트번호로 통신하면 된다. Pod 끼리의 통신은 Pod Network를 통해 직접적으로 가능하다.
![Pod - 공유 네트워크)](/files/kubernetes-pods-shared-network.png)

### Pod안의 컨테이너들은 시스템 리소스를 공유한다
Pod은 자기만의 커널 네임스페이스뿐만 아니라 리눅스의 cgroups를 통해 자기만의 시스템 리소스들을 가질수 있다. 시스템 리소스에는 CPU, RAM, IOPS 등이 있다. Pod안에 있는 각각의 컨테이너별로 cgroups를 통해 시스템 리소스를 정의할 수 있어 컨테이너의 어플리케이션 성격에 따라 최적화된 리소스를 배분할 수 있다. 결과적으로 컨테이너에게 할당된 최적화된 리소스를 통해 시스템 자원들이 예기치 못하게 줄어드는 위험을 방지할 수 있다.

### Pod만으로는 한계가 있다
k8s에서 Pod 자체로는 장애나 부하 발생 시 자동 복구나 자동 스케일링이 동작하지 않는다. 이후에 살펴볼 `Deployments`나 `DemonSets` 컨트롤러를 통해 Pod의 장애가 발생 시 Pod를 재스케쥴링하여 장애를 처리하게 된다. 이번 글에서는 Pod 자체를 설명하기 위해 `Deployments`나 `DemonSets` 컨트롤러를 사용하지 않고 `kind: Pod`만을 사용하여 예제를 작성하였다.
