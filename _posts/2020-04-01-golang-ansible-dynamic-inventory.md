---
title:  "Golang으로 Ansible Dynamic Inventory 개발"
excerpt: "Golang으로 dynamic inventory를 개발해보자"

categories:
  - golang
author: MinSik Kim
tags:
  - golang
  - ansible
  - openstack-ansible
  - dynamic inventory
  - inventory
classes: wide
last_modified_at: 2020-04-01T11:47:00
---

## Golang으로 Ansible Dynamic Inventory 개발

### Ansible Dynamic Inventory 간단한 개념

Ansible은 동적인 환경(ex) 클라우드)에서 인프라에 대한 정보를 관리하기 위해 dynamic inventory를 사용한다. dynamic inventory는 실행파일로써 동적으로 변하는 환경에 따라 인프라에 대한 정보를 가져와 JSON같은 형태로 동적인 inventory를 출력한다.

Dynamic inventory를 실행하여 JSON 포맷으로 출력되는 간단한 예제를 보자. 이 JSON 포맷은 Ansible이 inventory로 해석할 수 있는 형태여야 한다.
> 이 문서에서 Inventory의 형태에 대해서는 설명하지 않는다. Dynamic Inventory 형태에 대해서는 [Ansible Developing Dynamic Inventory](https://docs.ansible.com/ansible/latest/dev_guide/developing_inventory.html)를 참조하면 된다.

```json
{
  "_meta": {
    "hostvars": {
      "mingle-k8s-master-1": {
        "ansible_host": "192.168.0.20",
        "ansible_ssh_port": 22,
        "ansible_ssh_user": "ubuntu"
      },
      "mingle-k8s-node-1": {
        "ansible_host": "192.168.0.100",
        "ansible_ssh_port": 22,
        "ansible_ssh_user": "ubuntu"
      }
    }
  },
  "all": {
    "vars": {
      "cidr_networks": {
        "mgmt": "192.168.0.0/24"
      },
      "environment_name": "minglebros"
    }
  },
  "etcd": {
    "hosts": [
      "mingle-k8s-master-1"
    ]
  },
  "hosts": {
    "children": [
      "kubernetes_etcd_host",
      "kubernetes_master_hosts",
      "kubernetes_worker_hosts"
    ]
  },
  "k8s-cluster": {
    "children": [
      "etcd",
      "kube-master",
      "kube-node"
    ]
  },
  "kube-master": {
    "hosts": [
      "mingle-k8s-master-1"
    ]
  },
  "kube-node": {
    "hosts": [
      "mingle-k8s-node-1"
    ]
  },
  "kubernetes_etcd_hosts": {
    "hosts": [
      "mingle-k8s-master-1"
    ]
  },
  "kubernetes_master_hosts": {
    "hosts": [
      "mingle-k8s-master-1"
    ]
  },
  "kubernetes_worker_hosts": {
    "hosts": [
      "mingle-k8s-node-1"
    ]
  }
}
```

각각의 그룹(ex) kube-master, kube-cluster)은 `hosts` 혹은 `children`을 dictionary로 가지고 있다. `hosts` 같은 경우에는 `_meta`에 정의된 각각의 host들에 대한 정보를 참조하고 `children`의 경우에는 정의된 그룹들을 참조한다.

```json
{
  "_meta": {
    "hostvars": {
      "mingle-k8s-master-1": {
        "ansible_host": "192.168.0.20",
        "ansible_ssh_port": 22,
        "ansible_ssh_user": "ubuntu"
      },
      "mingle-k8s-node-1": {
        "ansible_host": "192.168.0.100",
        "ansible_ssh_port": 22,
        "ansible_ssh_user": "ubuntu"
      }
    }
  },
  ...
  "k8s-cluster": {
    "children": [
      "etcd",
      "kube-master",
      "kube-node"
    ]
  },
  ...
  "kube-master": {
    "hosts": [
      "mingle-k8s-master-1"
    ]
  },
```

이제 실제로 Dynamic Inventory를 작성하는 예제를 살펴보자. Dynamic한 환경은 유저, 회사마다 다르기 때문에 오픈소스인 [Openstack Ansible](https://docs.openstack.org/openstack-ansible/latest/)를 참고하여 golang으로 dynamic inventory를 작성해 볼 것이다.
dynamic inventory를 개발하기에 앞서 고려해야할 점은 어떤 데이터 형태로 인프라에 대한 정보를 받아서 inventory를 출력할지 결정해야 한다. 예를 들어 외부 Database에서 환경정보를 가져올수도 있고 아니면 사용자가 정의한 파일로부터 가져올수도 있다. 데이터를 가져왔다면 이 데이터를 기반으로 Ansible이 해석할 수 있는 inventory 형태로 출력해야 한다. 이 글에서는 Openstack Ansible를 따라 유저가 정의한 파일을 읽어와 inventory를 출력하는 예제로 작성된다.
![Dynamic Inventory)](/files/golang-dynamic-inventory.png)

### 사용자 정의 파일(yaml) 작성

[Openstack Ansible](https://github.com/openstack/openstack-ansible/tree/master/etc/openstack_deploy)에서 작성한 사용자 정의파일을 작성한 것을 기반으로 간단한 사용자 정의 파일을 작성해 보겠다. 나중에 쿠버네티스 클러스터를 배포하는 자동화 오픈소스인 [kubespay](https://github.com/kubernetes-sigs/kubespray)를 이용하기 위해 쿠버네티스 클러스터와 연관하여 예제를 작성해 보겠다.
```yaml
$ vim /etc/minglebros/k8s-user-config.yml
---
cidr_networks:
  mgmt: 192.168.0.0/24
global_overrides:
  environment_name: minglebros
_kubernetes_infrastructure_hosts: &kubernetes_infrastructure_hosts
  - name: mingle-k8s-master-1
    ip_addr: 192.168.0.20
_kubernetes_worker_hosts: &kubernetes_worker_hosts
  - name: mingle-k8s-node-1
    ip_addr: 192.168.0.100
groups:
  - name: kubernetes_master_hosts
    hosts: *kubernetes_infrastructure_hosts
  - name: kubernetes_worker_hosts
    hosts: *kubernetes_worker_hosts
  - name: kubernetes_etcd_hosts
    hosts: *kubernetes_infrastructure_hosts
```
Easily Human Readable한 yml파일에 사용자 정의 데이터를 작성해 보았다. `cidr_networks`와 `global_overrides`는 실제 모든 호스트에 적용되는 variable로 출력이 된다.
```json
  "all": {
    "vars": {
      "cidr_networks": {
        "mgmt": "192.168.0.0/24"
      },
      "environment_name": "minglebros"
    }
  },
```
`_`으로 시작하는 변수는 `&{Linked Name}`를 사용하여 `groups`에서 참조할 호스트들을 정의한다. 실제 `groups`에서는 `*`를 이용하여 각 그룹에 호스트들을 연결하여 그룹을 만들도록 정의했다.

### 환경 파일(yaml) 작성

환경 파일은 각 호스트들을 논리적인 그룹을 만들기 위해 사용된다. Ansible에서 호스트들은 하나 혹은 다수의 그룹에 포함될 수 있다. 예를들어 Kubernetes 같은 경우 부하가 크지 않으면 control plane와 data store(etcd)를 같은 노드에 배포하는 경우처럼 말이다. 밑의 예시를 살펴보자.
```yml
$ vim /etc/minglebros/env.d/k8s-master.yml
---

component_skel:
  kube-master:
    belongs_to:
      - k8s-cluster

service_skel:
  kube-master:
    belongs_to:
      - kubernetes_master_hosts

physical_skel:
  kubernetes_master_hosts:
    belongs_to:
      - hosts

$ vim /etc/minglebros/env.d/etcd.yml
---

component_skel:
  etcd:
    belongs_to:
      - k8s-cluster

service_skel:
  etcd:
    belongs_to:
      - kubernetes_etcd_hosts

physical_skel:
  kubernetes_etcd_host:
    belongs_to:
      - hosts

$ vim /etc/minglebros/env.d/k8s-worker.yml
---

component_skel:
  kube-node:
    belongs_to:
      - k8s-cluster

service_skel:
  kube-node:
    belongs_to:
      - kubernetes_worker_hosts

physical_skel:
  kubernetes_worker_hosts:
    belongs_to:
      - hosts
```
위의 예시를 분석해 보면 `componet_skel`와 `physical_skel` 같은 경우에는 그룹들을 다시 그룹으로 mapping하는 역할을 하고 `service_skel`은 사용자 정의 파일에 있는 호스트 그룹들을 새로운 그룹으로 mapping한다. Skel의 이름을 component, physical, service로 나눈 이유는 사용자가 더 편리하게 논리적인 그룹을 만들도록 하기 위해서이다.

> 이 `env.d` 개념은 Openstack Ansible이 dynamic inventory를 만들때 사용한다. 실제 Openstack Ansible은  linux container로 Openstack을 배포하기 때문에 container 관련된 부분은 빼고 필요한 부분만 추출하여 `env.d` 개념을 가져와 사용하였다. Openstack Ansible의 `env.d`에 대한 자세한 설명은 [Openstack Ansible Env.d](https://docs.openstack.org/project-deploy-guide/openstack-ansible/newton/app-custom-layouts.html#understanding-container-groups)를 참조하기 바란다.

### Golang Dynamic Inventory 작성

Golang Dynamic Inventory를 작성하기에 앞서 모든 코드를 공개해 알려주기 보다는 Inventory를 작성할때 어떠한 것들을 고려해야 하는지를 알려주고 자기 환경에 맞게 직접 Dynamic Inventory를 작성하는게 낫다고 생각한다. 사실 Dynamic Inventory를 개발하는것은 생각보다 어렵지 않다. 위에서 말한바와 같이 결과적으로만 본다면 Ansible이 해석할 수 있는 inventory 형태로 JSON 문자열을 출력하면 된다. 좀 더 나은 dynamic inventory를 개발하길 원하다면 option(argument)을 만들어 사용자가 더 쉽게 원하는 inventory를 출력할 수 있도록 하면 된다. 간단하게 main함수를 살펴보자.

```go
package main

import (
	"flag"
	"fmt"

	"github.com/minglebros/golang-inventory/toolkit"
)

const (
	defaultConfigDir = "/etc/minglebros"
)

var (
	config      = flag.String("config", defaultConfigDir, "Print version information and exit")
	list        = flag.Bool("list", false, "List all entries")
	check       = flag.String("check", "", "Configuration check only, don't generate inventory")
	debug       = flag.Bool("debug", false, "Output debug messages to log file")
	environment = flag.String("environment", defaultConfigDir, "Directory that contains the base env.d directory")
)

func getArgs() toolkit.Args {
	args := toolkit.Args{
		Config:      *config,
		List:        *list,
		Check:       *check,
		Debug:       *debug,
		Environment: *environment,
	}
	return args
}

func main() {
	flag.Parse()
	output := toolkit.Generate(getArgs())
	fmt.Println(output)
}
```
main 함수에는 몇가지 필요한 argument를 넣어 사용자가 보다 편리하게 inventory를 출력할 수 있도록 하였다. `config`와 `environment`는 사용자 정의 파일과 `env.d` 디렉토리의 위치를 지정할 수 있도록 했고 `list`는 전체 inventory를 출력한다. 이제 실제 사용자 정의파일과 env.d를 해석하여 inventory를 만드는 코드를 살펴보자

```go
package toolkit

import (
	"encoding/json"
	"log"
)

type UserConfig struct {
	CidrNetwork     map[string]string `yaml:"cidr_networks,omitempty"`
	GlobalOverrieds map[string]string `yaml:"global_overrides,omitempty"`
	Groups          []Group           `yaml:"groups,omitempty"`
}

type Group struct {
	Name  string `yaml:"name,omitempty"`
	Hosts []Host `yaml:"hosts,omitempty"`
}

type Host struct {
	Name      string `yaml:"name,omitempty"`
	IPAddress string `yaml:"ip_addr,omitempty"`
	SSHUser   string `yaml:"ssh_user,omitempty"`
	SSHPort   int    `yaml:"ssh_port,omitempty"`
}

type Environments []Skeleton

type Skeleton map[string]SkelType

type SkelType map[string]GroupMapper

type GroupMapper map[string][]string

func Generate(args Args) string {
	userConfig, err := LoadUserConfiguration(args.Config)
	if err != nil {
		log.Fatal(err)
	}
	environments, err := LoadEnvironments(args.Config)
	if err != nil {
		log.Fatal(err)
	}
	inventory := initInventory()

	// TODO Validate userConfig and Environment

	err = globalVariablesSetup(inventory, userConfig)
	if err != nil {
		log.Fatal(err)
	}

	err = userDefineSetup(inventory, userConfig)
	if err != nil {
 		log.Fatal(err)
	}

	err = groupSetup(inventory, userConfig, environments)
	if err != nil {
		log.Fatal(err)
	}

	inventoryJSON, err := json.MarshalIndent(inventory, "", "  ")
	if err != nil {
		log.Fatal(err)
	}
	return string(inventoryJSON)
}
```
모든 코드를 설명할 수는 없지만 간략하게 말해서 파일들을 읽어 해석하여 JSON 포맷으로 데이터를 출력하는 함수이다. Ansible이 해석할 수 있는 JSON 포맷으로 inventory를 출력하면 Ansible은 inventory 정보를 가지고 playbook을 실행할 것이다. 이 Return되는 JSON Format이 [첫번째](#ansible-dynamic-inventory-간단한-개념) chapter에서 보여준 JSON Format이다. 이 글을 읽는 독자의 환경에 맞는 dynamic inventory를 개발하기에 이 포스트가 도움이 조금이라도 되길 바란다.
```sh
$ cd ~/minglebros
$ go build main.go

# kubspray를 실행시킴 (Kubespray에 대해서는 다른 블로그 글에 더 자세히 작성하겠습니다)
$ cd ~/kubespary
$ ansible-playbook --become -i ~/minglebros/main cluster.yml -vv
``` 

### 참조
- [Ansible Developing Dynamic Inventory](https://docs.ansible.com/ansible/latest/dev_guide/developing_inventory.html)
- [Openstack Ansible Dynamic Inventory](https://github.com/openstack/openstack-ansible/tree/master/inventory)
- [Kubespray](https://github.com/kubernetes-sigs/kubespray)

