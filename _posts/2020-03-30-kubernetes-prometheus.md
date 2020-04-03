---
title:  "클알못의 Prometheus 소개"
excerpt: "Prometheus 개념을 알아보자"

categories:
  - kubernetes
author: ChanHo Park
tags:
  - cloud
  - kubernetes
  - prometheus
  - monitoring
  - cloud native
  - grafana
classes: wide
---

## Legacy Monitoring vs Cloud Native Monitoring

![Legacy-vs-Cloud-Monitoring](/files/legacy-vs-cloud.png)

Legacy 환경에서 모니터링은 server(vm)마다 agent가 설치되어 정보를 수집하는 형태이다. server(vm) 내부에 설치된 agent에서 applcation 관련 metric이나 os 관련 metric 등을 수집해 backend로 보내는 방식이다. 이를 push 방식이라고 한다.

Cloud Native 모니터링은 Monitoring Backend (ex. prometheus) 에서 동적으로 확장하는 server(pod)들을 discovery하여 api를 통해 application이나 node 관련 metric 등을 backend가 수집해가는 방식이다. 이를 pull 방식이라고 한다.

Legacy 모니터링은 몇가지 단점이 존재한다. 우선 각 server(vm)에 agent가 존재해야한다. server(vm)을 새로 생성할때마다 agent를 설치해줘야하는 불편함이 있다. Cloud Native 환경에서는 동적으로 server(pod)들이 자주 생성되고 삭제되는데 이런 환경에서는 부적합하다. 또한 각 server(vm)에 설치된 agent에서 backend로 metric을 전송하는 방식이기 때문에 Legacy 모니터링 솔루션을 실행하기 위해 많은 양의 컴퓨팅 성능, 메모리 및 서버 하드웨어가 필요하다. 비용 효율적인 운영을 원하는 기업 입장에서는 초기 구축 및 운영에 많은 부담이 된다. 


## Prometheus Architecture

![Prometheus-Architecture](/files/prometheus-architecture.png)

프로메테우스는 CNCF의 메인 프로젝트로서 container 기반 monitoring 시스템으로 가장 인기를 끌고 있다. Kubernetes 외에 다른 cloud provider에 service discovery 기능 제공, 자체 alert 엔진을 가지고 있어 외부 시스템과 연동하여 alarm 송신 가능, web ui와 Grafana를 통한 data 시각화, 자체 TSDB(Time Series Database) 보유하여 metric 데이터 저장 및 관리에 최적화, 다양한 exporter를 제공하여 외부 시스템 모니터링 제공 등 Cloud Native 환경에 적합한 모니터링 시스템이다. 

### Service Discovery
 Prometheus Server가 metric을 수집해오려면 먼저 모니터링 대상을 알아야한다. Kubernetes 환경에서는 모니터링 대상의 ip가 동적으로 변경되기때문에 일일이 설정파일에 값을 넣어주기 힘들다. 이런 문제를 해결하기 위해 Service Discovery를 사용하여 모니터링 대상을 가져온다. 기본적으로 kubernetes api를 호출해서 모니터링할 리소스(pod, node, service, ingress, endpoint 등)들을 label selector를 이용하여 가져온다. 그런 다음 모니터링할 리소스로부터 metric 정보를 가져와야하는데 프로메테우스는 기본적으로 /metric 이라는 URL을 통해서 metric을 수집한다. Service Discovery는 Kubernetes API, DNS, Consul, etcd 등을 이용해 다양한 mechanism으로 모니터링 대상을 가져올 수 있다. 

### Pull Metrics
프로메테우스는 pull 방식을 사용한다. agent가 backend로 metric을 보내는 것이 아니라 Prometheus Server가 주기적으로 모니터링 대상으로부터 metric을 읽어 온다. 모니터링 대상이 프로메테우스의 데이터 포맷을 지원할 경우 바로 metric을 가져올 수 있고 지원하지 않는다면 exporter를 설치해야한다.

### Push Gateway
batch 작업이나 스케쥴 작업 같이 수명이 짧은, 필요한 경우에만 실행되고 끝나면 사라지는 작업들이 있다. 이러한 작업들 같은 경우 프로메테우스의 pull 방식으로 metric을 가져오기 어렵다. 이런 문제점을 해결하기위한 것이 Push Gateway다. Push Gateway에 metric을 전송하면 Push Gateway가 보관하고 있다가 Prometheus Server가 pulling할때 저장된 metric을 리턴한다.

### Exporter
프로메테우스 포맷을 지원하지 않는 리소스의 경우, exporter를 설치하여 metric 정보를 보낸다. Kubernetes 클러스터에 배포하려는 많은 애플리케이션이 기본적으로 프로메테우스 포맷으로 metric을 지원하지 않을 수 있다. 이런 경우 서비스의 상태, 로그, 다른 metric 정보를 가져올 수 있는 exporter가 필요하다. 모니터링 대상의 metric을 수집하고 Prometheus Server한테 metric을 전송한다. Prometheus Server의 요청을 받으면 필요한 metric을 수집하여 적절한 포맷으로 변경후 리턴해준다. mysql, nginx, redis 등 많은 패키지를 위한 exporter가 있어서 exporter를 이용하여 다양한 서비스의 metric 정보를 수집할 수 있다.

### Data Visualization
수집된 metric들은 프로메테우스 내부의 TSDB에 저장이 되고, 프로메테우스 자체 웹 콘솔을 통해 visualization 되거나 또는 API를 제공해서 Grafana와 같은 visualization 툴을 통해서 metric을 visualization해서 볼 수 있다.

### Alert Manager
알림을 받을 규칙을 만들어서 Alert Manager로 보내면 Alert Manager가 규칙에 따라 알림을 보낸다. email, slack 등으로 알림을 보낼 수 있고 pagerduty와 같은 nitofication 서비스와도 연동이 가능하다.

## Integration with Kubernetes

<p align="center"><img src="/files/integration-kubernetes.png"></p>

Kubernetes 기본 환경에서 프로메테우스를 설치할 경우 위와 같은 구조를 갖는다. Service Discovery 메커니즘을 통해 master 노드에 있는 kubnernetes api를 호출하여 모니터링할 리소스를 가져온다. node 자체에 대한 정보는 api를 통해 가져오기 어렵기 때문에 exporter를 설치하여 하드웨어와 OS에 대한 정보를 수집한다. pod에 대한 정보는 각 노드에 배포되어 있는 kublet안에 cAdvisor를 통해 pod 정보를 수집하여 프로메테우스에 제공한다. pod에서 돌아가는 application에 대한 metric은 전용 exporter를 이용하여 수집할 수 있다.

## 참고
https://sysdig.com/blog/kubernetes-monitoring-prometheus/

https://bcho.tistory.com/1270?category=731548

https://medium.com/@tkdgy0801/prometheus-%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%9C-%EB%AA%A8%EB%8B%88%ED%84%B0%EB%A7%81-part-1-69de3e87d427

https://blog.outsider.ne.kr/1254

https://yunsangjun.github.io/kubernetes/2018/07/20/kubernetes-monitoring.html

https://blog.naver.com/alice_k106/221521978267

https://coreos.com/blog/the-prometheus-operator.html
