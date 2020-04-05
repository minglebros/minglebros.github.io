---
title:  "Setup 및 Teardown 함수를 사용한 Golang 테스트"
excerpt: "Golang Unit 테스트 작성 시 Setup 및 Teardown 함수 이용하기"

categories:
  - golang
author: MinSik Kim
tags:
  - golang
  - Golang
  - test-driven
  - test
  - testing
  - setup
  - teardown
classes: wide
last_modified_at: 2020-04-05T22:00:00
---

## Golang Unit 테스트 (Setup 및 Teardown 함수 사용하기)

Golang으로 Unit 테스트 작성 시 간단한 함수의 경우에는 테스트할 Input값과
Input에 따른 expect 값을 작성하여 테스트할 수 있다. 하지만 HTTP Handler나 File을 읽고 쓰는 작업을 하는 함수를 테스트하기 위해서는 사전작업으로 Fake HTTP 서버를 올리거나 테스트하고자 하는 파일을 임시적으로 쓰고 삭제해야 할 것이다.
> 간단한 golang 테스트는 지난번 [포스트](https://minglebros.github.io/golang/golang-test-driven/)를 참조하자.

이번 포스트에서는 파일을 임시적으로 쓰고 테스트 후 파일을 삭제하는 Unit 테스트를 만들어보고자 한다. 마침 저번에 포스팅한 [golang으로 Ansible dynamic한 인벤토리](https://minglebros.github.io/golang/golang-ansible-dynamic-inventory/)에서 구현한 `generate` 함수가 파일을 읽고 JSON 포맷으로 데이터를 출력하므로 이번 포스트의 예제로 사용하고자 한다. 먼저 테스트 함수를 살펴보자.
```go
package testing

import (
	"strings"
	"testing"

	th "github.com/minglebros/golang-inventory/testhelper"
	"github.com/minglebros/golang-inventory/toolkit"
)

func TestGenerate(t *testing.T) {
	err := th.SetupFile()
	defer th.TearDownFile()
	if err != nil {
		t.Error(err)
	}
	output := toolkit.Generate(th.TestArgs)
	if output != strings.Replace(ExpectedJSONOutput, "\t", "", -1) {
		t.Error("Generated JsonOutput is not same with Expected JsonOutPut")
	}
}
```
`testhelper` package는 테스트하기 위한 helper 함수의 모음이라 보면 된다.
- `SetupFile()` 함수는 사전에 테스트하고자 내용을 담은 파일을 /tmp 디렉토리에 만든다.
- `TearDownFile()` 함수는 테스트하기 위해 임시적으로 만든 파일을 지운다.

만들어진 파일을 `Generate` 함수는 읽어와 JSON 데이터로 출력하여 `ExpectedJSONOutput` 결과값과 비교하여 성공인지 아닌지를 판단하게 된다. `testhelper` 패키지에 있는 `SetupFile()`와 `TearDownFile()` 함수를 살펴보자
```go
$ vim filehandler.go
package testhelper

import (
	"bufio"
	"os"
	"path/filepath"
	"strings"
)

type TestEnvironment struct {
	name    string
	content string
}

func WriteNewFile(filePath string, data string) error {
	f, err := os.Create(filePath)
	defer f.Close()
	if err != nil {
		return err
	}
	w := bufio.NewWriter(f)
	_, err = w.WriteString(data)
	w.Flush()
	return nil
}

func SetupFile() error {
	err := SetupUserConfig()
	if err != nil {
		return err
	}
	err = SetupEnvironment()
	if err != nil {
		return err
	}
	return nil
}

func TearDownFile() {
	filePath := filepath.Join(UserConfigFilePath, UserCnofigFileName)
	dirPath := filepath.Join(UserConfigFilePath, UserEnvironmentDir)
	if _, err := os.Stat(filePath); err == nil {
		_ = os.Remove(filePath)
	}

	if _, err := os.Stat(dirPath); err == nil {
		_ = os.RemoveAll(dirPath)
	}
}

func SetupUserConfig() error {
	filePath := filepath.Join(UserConfigFilePath, UserCnofigFileName)
	err := WriteNewFile(filePath, strings.Replace(TestUserConfig, "\t", "", -1))
	if err != nil {
		return err
	}
	return nil
}

func SetupEnvironment() error {
	dirPath := filepath.Join(UserConfigFilePath, UserEnvironmentDir)
	if _, err := os.Stat(dirPath); os.IsNotExist(err) {
		os.Mkdir(dirPath, 0755)
	}
	for _, env := range TestEnvironments {
		filePath := filepath.Join(dirPath, env.name)
		err := WriteNewFile(filePath, strings.Replace(env.content, "\t", "", -1))
		if err != nil {
			return err
		}
	}
	return nil
}
```
사실 복잡한 함수는 없다. 함수이름 그대로 `SetupFile`은 테스트하기 위한 임시 파일들을 만들고 `TearDownFile`은 만든 임시파일들을 지운다. 나는 파일을 테스트하기 위한 데이터들(파일의 내용, 파일 이름 등)은 다른 go 파일에 저장했다.
```go
$ vim fixture.go
package testhelper

import "github.com/minglebros/golang-inventory/toolkit"

const TestUserConfig = `
---
cidr_networks:
  pxe: 10.20.0.0/24
  public: 172.26.0.0/24
  mgmt: 192.168.0.0/24
  storage: 192.168.1.0/24

global_overrides:
  environment_name: minglebros

_kubernetes_infrastructure_hosts: &kubernetes_infrastructure_hosts
  - name: minglebors-k8s-master-1
    ip_addr: 192.168.0.20

_kubernetes_worker_hosts: &kubernetes_worker_hosts
  - name: minglebros-k8s-node-1
    ip_addr: 192.168.0.100

groups:
  - name: kubernetes_master_hosts
    hosts: *kubernetes_infrastructure_hosts

  - name: kubernetes_worker_hosts
    hosts: *kubernetes_worker_hosts

  - name: kubernetes_etcd_hosts
    hosts: *kubernetes_infrastructure_hosts
`

var (
	UserConfigFilePath = "/tmp"
	UserEnvironmentDir = "env.d"
	UserCnofigFileName = "k8s_user_config.yml"
	TestArgs           = toolkit.Args{
		Config:      UserConfigFilePath,
		List:        false,
		Check:       "",
		Debug:       false,
		Environment: UserConfigFilePath,
	}
	TestEnvironments = []TestEnvironment{
		TestEnvironment{
			name: "k8s-master.yml",
			content: `---
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
			`,
		},
		TestEnvironment{
			name: "k8s-worker.yml",
			content: `---
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
			`,
		},
		TestEnvironment{
			name: "etcd.yml",
			content: `---
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
			`,
		},
	}
)
```
테스트한 결과는 다음과 같다
```bash
$ go test github.com/minglebros/golang-inventory/toolkit/testing
ok  	github.com/minglebros/golang-inventory/toolkit/testing	0.399s
```
이번 포스트에서는 Golang으로 파일을 읽고 출력하는 함수를 어떻게 Unit 테스트 함수로 만드는지에 대해서 작성해 보았다. 다음 포스트에서는 HTTP Handler 함수를 테스트 하기 위해 Fake 서버를 만드는 방법을 공유하고자 한다.