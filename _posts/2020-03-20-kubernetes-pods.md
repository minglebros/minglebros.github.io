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

### Pod Manifest File
Pod의 Manifest 파일을 간단하게 살펴보자.
```yml
---

apiVersion: v1
kind: Pod
metadata:
  name: mingle-pod
  namespace: default
  labels:
    version: v1
    zone: mingle
spec:
  containers:
  - name: mingle-web
    image: registry.hub.docker.com/library/nginx:1.15
    ports:
    - containerPort: 80
```
Manifest 파일을 살펴보면 상위의 4개의 리소스를 볼수 있을 것이다.
- apiVersion
- kind
- metadata
- spec

`apiVersion`은 API group과 API version으로 구성된다. `<api-group>/<api-version>` 같은 방식으로 정의되는데 위의 예제에서는 `v1`으로 정의된 것을 볼수 있다. Pod같은 경우에는 `api-group`을 생략할 수 있는 core 그룹에 속해 있기 때문에 `v1`으로 정의할 수 있다. 나중에 살펴보겠지만 storage 클래스같은 경우에는 `storage.k8s.io/v1`으로 정의해야 한다.

`kind`는 k8s 클러스터에 배포하는 object를 말한다. (ex) Pod, Service, DeamonSets, Deployments)

`metadata`는 클러스터에 배포된 object를 구분시켜 주는 역할을 한다. 위의 예제에서는 이름과 라벨을 붙여 object간의 의존성을 낮추고 구분시켜 주었다. 라벨은 단순히 key-value 쌍으로 구성되지만 나중에 라벨이 굉장히 중요한 역할을 한다는 것을 알게 될 것이다. 이름과 라벨외에도 다수의 가상 클러스터를 제공할 수 있는 개념인 `namespace`를 정의하여 `mingle` 공간에 pod를 배포하였다. `namespace`를 정의하지 않으면 default namespace로 배포된다.

`spec`는 Pod에 배포되는 container의 정보를 정의하는 리소스이다. 간단히 이름과 배포할 docker 이미지와 노출할 포트를 정의하였다. 여기서는 하나의 container를 정의했지만 Pod안에 여러개의 container를 배포할려면 하나의 container를 더 정의하면 된다.

### Manifest파일로 Pod 배포
그럼 이제 Manifest파일을 생성하여 kubernetes 클러스터에 Pod를 배포해보자.
```bash
$ kubectl create -f pod.yml
pod/mingle-pod created

$ kubectl get pods
NAME         READY   STATUS    RESTARTS   AGE
mingle-pod   1/1     Running   0          40m

$ kubectl get pods mingle-pod -o yaml
apiVersion: v1
kind: Pod
metadata:
  ....

$ kubectl describe pods mingle-pod
Name:         mingle-pod
Namespace:    default
Priority:     0
Node:         kinx-k8s-node-1/192.168.88.24
Start Time:   Thu, 26 Mar 2020 22:06:21 +0900
Labels:       version=v1
              zone=mingle
Annotations:  <none>
Status:       Running
IP:           10.233.67.14
Containers:
  mingle-web:
    ....
Events:
  Type    Reason     Age   From                      Message
  ----    ------     ----  ----                      -------
  Normal  Scheduled  38m   default-scheduler         Successfully assigned default/mingle-pod to kinx-k8s-node-1
  Normal  Pulled     38m   kubelet, kinx-k8s-node-1  Container image "registry.hub.docker.com/library/nginx:1.15" already present on machine
  Normal  Created    38m   kubelet, kinx-k8s-node-1  Created container mingle-web
  Normal  Started    38m   kubelet, kinx-k8s-node-1  Started container mingle-web
```
kubectl `create` 명령어와 `-f`옵션에 manifest 파일을 참조하여 Pod를 생성하였다. 생성된 Pods의 상태를 보려면 `get` 명령어를 사용하면 되고 더 자세한 정보를 보려면  `describe` 명령어를 사용하면 된다. 위의 manifest파일에서 정의되지 않은 속성들은 kubernetes에서 자동으로 채워넣는다. kubernetes에서 가지고 있는 모든속성에 대한 정보를 보려면 `get` 명령어에 `-o` 옵션을 넣어 yaml이라든지 json으로 출력하면 된다.

이번에는 배포된 Pod에 접근하여 명령어를 실행해보자.
```bash
$ kubectl exec mingle-pod ls /usr/share/nginx/html
50x.html
index.html

$ kubectl exec mingle-pod -it bash
root@mingle-pod:/# ls /usr/share/nginx/html/
50x.html  index.html

$ kubectl exec mingle-pod -c mingle-web ls /usr/share/nginx/html
50x.html
index.html
```
`kubectl exec` 명령어를 통해 Pod에 접속해 html 파일들을 볼 수 있다. 만약 Pod에 로그인하여 명령어를 실행하고 싶은 경우 `-it` 옵션을 사용하여 접속할 수 있다. `-it`의 `i`는 Interactive의 간략한 표현으로 Pod안의 STDIN과 STDOUT을 `t` 터미널의 STDIN과 STDOUT로 연결한다는 의미이다.  마지막으로 Pod안에 여러개의 container가 배포된 경우에는 -c 옵션으로 container의 이름을 적어 특정한 container로 접속할 수 있다.

마지막으로 Pod의 log 보는 방법과 Pod를 지우는 방법을 알아보자.
```bash
$ kubectl port-forward mingle-pod 80
Forwarding from 127.0.0.1:80 -> 80
Forwarding from [::1]:80 -> 80

$ curl localhost:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
....

$ kubectl logs mingle-pod
127.0.0.1 - - [26/Mar/2020:14:15:12 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.58.0" "-"

$ kubectl delete pods mingle-pod
pod "mingle-pod" deleted
```
`kubectl port-forward`를 사용하여 해당 Pod에 80번 포트를 열어 접근하게 만든 다음 `curl` 명령어를 통해 접근하면 `kubectl logs` 명령어를 통해 Pod의 로그를 확인해 볼 수 있다. 마지막으로 `kubectl delete`를 사용하여 Pod를 삭제해 보았다.
