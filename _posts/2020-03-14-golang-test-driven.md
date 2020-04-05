---
title:  "Golang Simple Unit Test"
excerpt: "Golang에서 어떻게 Unit 테스트를 개발하는지 살펴보자"

categories:
  - golang
author: MinSik Kim
tags:
  - golang
  - Golang
  - test-driven
  - testing
  - simple
  - unit test
classes: wide
last_modified_at: 2020-03-14T20:00:00
---

## Golang Simple Unit Test

 자동 테스트는 기본 프로그램의 구성 요소를 실행하고 예상대로 작동하는지 확인하는 별도의 프로그램이다.
  자동 테스트는 코드를 변경 한 후에도 프로그램 구성 요소가 올바르게 작동하는지 확인한다. Go의 테스트 패키지 및 Go 테스트 도구를 사용하면 자동 테스트를 쉽게 작성할 수 있다. 자동화 테스트를 통해 메뉴얼하게 테스트하는 시간을 줄이고 더 철저하게 테스트를 할 수 있다.
아래 코드는 headfirstgo 책에 나온 예제 코드로써 strings.Join 함수를 사용한다.이 함수는 문자열 조각과 문자열을 모두 함께 결합한다. Join은 슬라이스의 모든 항목이 결합 된 단일 문자열을 반환한다.
다음은 소스코드가 위치하고 있는 패키지 디렉토리의 모습니다.
HOME/go/src/github.com/headfirstgo/prose/join.go
HOME/go/src/github.com/headfirstgo/prose/join_test.go
`go test` 명령어는 패키지에 위치하고 있는 xxx_test.go의 test case를 실행한다.
먼저 `join.go`를 살펴보자.
```go
package prose

import "strings"

func JoinWithCommas(phrases []string) string {
    if len(phrases) == 1 {
        return phrases[0]
    } else if len(phrases) == 2 {
        return phrases[0] + " and " + phrases[1]
    } else {
        result := strings.Join(phrases[:len(phrases)-1], ", ")
        result += ", and "
        result += phrases[len(phrases)-1]
        return result
    }
}
```

테스트 파일 내의 코드는 일반적인 Go 함수로 구성되어 있지만 go 테스트 도구를 사용하려면 특정 규칙을 따라야한다.
1. 테스트 기능 이름은 Test로 시작해야한다. 나머지 이름은 원하는대로 할 수 있지만 대문자로 시작해야한다.
2. 테스트 함수는 하나의 매개 변수, `testing.T` 값에 대한 포인터를 받아야 한다.
3. `testing.T` 값에서 메소드를 호출하여 테스트가 실패했음을 설명하는 메시지가 포함되어야 한다.

join_test.go를 살펴보자
```go
package prose

import (
    "fmt"
    "testing"
)

type testData struct {
    list []string
    want string
}

func TestJoinWithCommas(t *testing.T) {
    tests := []testData{
        testData{list: []string{"apple"}, want: "apple"},
        testData{list: []string{"apple", "orange"}, want: "apple and orange"},
        testData{list: []string{"apple", "orange", "pear"}, want: "apple, orange, and pear"},
    }
    for _, test := range tests {
        got := JoinWithCommas(test.list)
        if got != test.want {
            t.Error(errorString(test.list, got, test.want))
        }
    }
}

func errorString (list []string, got string, want string) string {
    return fmt.Sprintf("JoinWithCommas(%#v) = \"%s\", want \"%s\"", list, got, want)
}
```
테스트를 실행하기 위해 테스트 파일이 있는 패키지를 `go test`명령어와 함께 실행한다
```sh
$ go test github.com/headfirstgo/prose
ok  	github.com/headfirstgo/prose	0.743s
```
위 코드는 테스트할 데이터와 테스트가 성공할 때 return하는 데이터를 가질 수 있는 `testData` 구조체를 정의한다. `tests` slice 변수에 `testData` 구조체 형식의 테스트 데이터를 할당하여 `JoinWithCommas` 함수를 테스트한다.

## Test-driven Development

테스트 중심 개발
1. 테스트 작성 : 아직 존재하지 않더라도 원하는 기능에 대한 테스트를 작성한다. 그런 다음 테스트를 실행하여 실패했는지 확인한다.
2. 전달 : 기본 코드에서 기능을 구현한다. 작성하는 코드가 비효율적인지 걱정하지 말고 작동시키는 코드를 만들어 테스트를 실행하여 통과하는지 확인한다.
3. 코드 리팩토링 : 이제 코드를 리팩토링하고 코드를 변경 및 개선 할 수 있다.

### Example

`JoinWithCommas` 함수에 데이터가 들어있지 않는 빈 `testData`가 들어가면 어떻게 될까? 테스트 중심 개발은 `JoinWithCommas` 함수를 수정하기 전에 테스트 코드를 작성한다. 비어있는 `testData`를 넣어 테스트를 해보자.
```go
...
func TestJoinWithCommas(t *testing.T) {
    tests := []testData{
        testData{list: []string{}, want: ""},
        testData{list: []string{"apple"}, want: "apple"},
        testData{list: []string{"apple", "orange"}, want: "apple and orange"},
        testData{list: []string{"apple", "orange", "pear"}, want: "apple, orange, and pear"},
    }
    ...
```
테스트 코드를 수정 후 테스트 실행
```sh
$ go test github.com/headfirstgo/prose
--- FAIL: TestJoinWithCommas (0.00s)
panic: runtime error: slice bounds out of range [:-1] [recovered]
	panic: runtime error: slice bounds out of range [:-1]

goroutine 7 [running]:
testing.tRunner.func1(0xc000090100)
	/usr/local/go/src/testing/testing.go:874 +0x3a3
panic(0x11340c0, 0xc0000161a0)
	/usr/local/go/src/runtime/panic.go:679 +0x1b2
github.com/headfirstgo/prose.JoinWithCommas(0x1256d00, 0x0, 0x0, 0xf, 0x10bebe0)
	/Users/Beatitudo/go/src/github.com/headfirstgo/prose/join.go:14 +0x1aa
github.com/headfirstgo/prose.TestJoinWithCommas(0xc000090100)
	/Users/Beatitudo/go/src/github.com/headfirstgo/prose/join_test.go:21 +0x207
testing.tRunner(0xc000090100, 0x11500c8)
	/usr/local/go/src/testing/testing.go:909 +0xc9
created by testing.(*T).Run
	/usr/local/go/src/testing/testing.go:960 +0x350
FAIL	github.com/headfirstgo/prose	0.528s
FAIL
```
Error의 메시지를 보면 `JoinWithCommas` 함수에서 slice에서 데이터를 접근할 수 없다는 메시지를 볼 수 있다. `JoinWithCommas` 함수를 수정해 보자
```go
...
func JoinWithCommas(phrases []string) string {
    if len(phrases) == 0 {
        return ""
    } else if len(phrases) == 1 {
        return phrases[0]
    } else if len(phrases) == 2 {
        return phrases[0] + " and " + phrases[1]
    } else {
        result := strings.Join(phrases[:len(phrases)-1], ", ")
        result += ", and "
        result += phrases[len(phrases)-1]
        return result
    }
}
```
`phrases` slice 변수에 element가 없다면 빈 문자열을 return하면 문제를 해결할 수 있다.
테스트를 실행하면 성공함을 알 수 있다.
```go
$ go test github.com/headfirstgo/prose
ok  	github.com/headfirstgo/prose	(cached)
```


