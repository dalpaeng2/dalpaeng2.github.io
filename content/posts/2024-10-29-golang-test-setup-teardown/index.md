---
title: Golang Test 코드 작성할 때 Setup, Teardown 적용하기
date: 2024-10-29T20:23:37+09:00
tags: [golang, testing]
---

테스트 코드를 작성할 때 각 테스트를 수행하기 전/후에 실행할 작업을 `Setup`, `Teardown`이라고 합니다.
Golang으로 테스트 코드를 작성할 때 `Setup`, `Teardown` 동작을 수행하는 방법을 공유합니다.

## TestMain 함수 사용하기

golang 자체적으로 지원하는 기능으로 테스트 파일에 `TestMain(m *testing.M)` 함수가 있을 경우 해당 파일의 테스트를 실행하기 전/후에 특정 작업을 수행하도록 할 수 있습니다.

```go
package test_main_test

import (
  "fmt"
  "testing"
)

var (
  GlobalVar = 0
)

func TestFunct_1(t *testing.T) {
  t.Log("run test 1")
  if GlobalVar != 1 {
    t.Error("GlobalVar should be equal to 1")
  }
}

func TestFunc_2(t *testing.T) {
  t.Log("run test 2")
  if GlobalVar != 1 {
    t.Error("GlobalVar should be equal to 1")
  }
}

func setup() {
  fmt.Println("setup TestMain")
  GlobalVar = 1
}

func teardown() {
  fmt.Println("tear down TestMain")
  GlobalVar = 1
}

func TestMain(m *testing.M) {
  setup()
  m.Run()
  teardown()
}
```

```bash
setup TestMain
=== RUN   TestFunct_1
    test_main_test.go:13: run test 1
--- PASS: TestFunct_1 (0.00s)
=== RUN   TestFunc_2
    test_main_test.go:20: run test 2
--- PASS: TestFunc_2 (0.00s)
PASS
tear down TestMain
ok      go-test/test_main       0.238s
```

이 방식은 테스트 파일 기준으로 동작하기 때문에 파일 내부에 테스트 함수가 여러개있어도 각 테스트마다 `Setup`, `Teardown`이 실행되지 않고 한번만 실행됩니다.

## Testify suite 사용하기

`testify` 패키지의 `suite`를 사용하면 `Setup`, `Teardown` 동작을 수행할 수 있습니다. `Suite`, `Test`, `SubTest` 실행 전/후에 각각 `Setup`, `Teardown`을 수행할 수 있어서 세밀하게 조절이 가능합니다.
(SubTest는 Table Driven Test에서 각 테스트 Run을 의미합니다.)

```go
package testify_suite_test

import (
  "testing"

  "github.com/stretchr/testify/suite"
)

type UnitTestSuite struct {
  suite.Suite
}

func (s *UnitTestSuite) Test_TableTest1() {
  type testCase struct {
    name string
  }

  testCases := []testCase{
    {
      name: "1",
    },
    {
      name: "2",
    },
  }

  for _, tc := range testCases {
    s.Run(tc.name, func() {
      s.T().Log("run test: ", tc.name)
    })
  }
}

func (s *UnitTestSuite) Test_TableTest2() {
  type testCase struct {
    name string
  }

  testCases := []testCase{
    {
      name: "3",
    },
    {
      name: "4",
    },
  }

  for _, tc := range testCases {
    s.Run(tc.name, func() {
      s.T().Log("run test: ", tc.name)
    })
  }
}

func (s *UnitTestSuite) SetupSuite() {
  s.T().Log("setup suite")
}

func (s *UnitTestSuite) TearDownSuite() {
  s.T().Log("tear down suite")
}

func (s *UnitTestSuite) SetupTest() {
  s.T().Log("setup test")
}

func (s *UnitTestSuite) TearDownTest() {
  s.T().Log("tear down test")
}

func (s *UnitTestSuite) BeforeTest(suiteName, testName string) {
  s.T().Log("before test")
}

func (s *UnitTestSuite) AfterTest(suiteName, testName string) {
  s.T().Log("after test")
}

func (s *UnitTestSuite) SetupSubTest() {
  s.T().Log("setup sub test")
}

func (s *UnitTestSuite) TearDownSubTest() {
  s.T().Log("tear down sub test")
}

func TestUnitTestSuite(t *testing.T) {
  suite.Run(t, new(UnitTestSuite))
}
```

```bash
=== RUN   TestUnitTestSuite
    testify_suite_test.go:56: setup suite
=== RUN   TestUnitTestSuite/Test_TableTest1
    testify_suite_test.go:64: setup test
    testify_suite_test.go:72: before test
=== RUN   TestUnitTestSuite/Test_TableTest1/1
    testify_suite_test.go:80: setup sub test
    testify_suite_test.go:29: run test:  1
    testify_suite_test.go:84: tear down sub test
=== RUN   TestUnitTestSuite/Test_TableTest1/2
    testify_suite_test.go:80: setup sub test
    testify_suite_test.go:29: run test:  2
    testify_suite_test.go:84: tear down sub test
=== NAME  TestUnitTestSuite/Test_TableTest1
    testify_suite_test.go:76: after test
    testify_suite_test.go:68: tear down test
=== RUN   TestUnitTestSuite/Test_TableTest2
    testify_suite_test.go:64: setup test
    testify_suite_test.go:72: before test
=== RUN   TestUnitTestSuite/Test_TableTest2/3
    testify_suite_test.go:80: setup sub test
    testify_suite_test.go:50: run test:  3
    testify_suite_test.go:84: tear down sub test
=== RUN   TestUnitTestSuite/Test_TableTest2/4
    testify_suite_test.go:80: setup sub test
    testify_suite_test.go:50: run test:  4
    testify_suite_test.go:84: tear down sub test
=== NAME  TestUnitTestSuite/Test_TableTest2
    testify_suite_test.go:76: after test
    testify_suite_test.go:68: tear down test
=== NAME  TestUnitTestSuite
    testify_suite_test.go:60: tear down suite
--- PASS: TestUnitTestSuite (0.00s)
    --- PASS: TestUnitTestSuite/Test_TableTest1 (0.00s)
        --- PASS: TestUnitTestSuite/Test_TableTest1/1 (0.00s)
        --- PASS: TestUnitTestSuite/Test_TableTest1/2 (0.00s)
    --- PASS: TestUnitTestSuite/Test_TableTest2 (0.00s)
        --- PASS: TestUnitTestSuite/Test_TableTest2/3 (0.00s)
        --- PASS: TestUnitTestSuite/Test_TableTest2/4 (0.00s)
PASS
```

## 수동으로 `Setup`, `Teardown`호출하기

테스트를 수행할 때 직접 `Setup`, `Teardown`함수를 호출하는 방법입니다. Table Driven Test 방식에서도 적용할 수 있는 방법입니다.
`Setup`함수에서 필요한 초기화를 수행하고 테스트에 필요한 값을 반환하는 형태로 제공하고, `Teardown`함수를 `t.Cleanup`을 통해 호출해서 자원을 해제할 수 있습니다.

```go
package manual_test

import "testing"

func setup(t *testing.T) func() {
  t.Log("setup")

  return func() {
    t.Log("teardown")
  }
}

func TestManualSetupTeardown(t *testing.T) {
  type testCase struct {
    name string
  }

  testCases := []testCase{
    {
      name: "1",
    },
    {
      name: "2",
    },
  }

  for _, tc := range testCases {
    t.Run(tc.name, func(t *testing.T) {
      teardown := setup(t)
      t.Cleanup(teardown)

      t.Logf("run test: %s", tc.name)
    })
  }
}
```

```bash
=== RUN   TestManualSetupTeardown
=== RUN   TestManualSetupTeardown/1
    /Users/dalpaeng2/go-test/manual/manual_test.go:6: setup
    /Users/dalpaeng2/go-test/manual/manual_test.go:32: run test: 1
    /Users/dalpaeng2/go-test/manual/manual_test.go:9: teardown
--- PASS: TestManualSetupTeardown/1 (0.00s)
=== RUN   TestManualSetupTeardown/2
    /Users/dalpaeng2/go-test/manual/manual_test.go:6: setup
    /Users/dalpaeng2/go-test/manual/manual_test.go:32: run test: 2
    /Users/dalpaeng2/go-test/manual/manual_test.go:9: teardown
--- PASS: TestManualSetupTeardown/2 (0.00s)
--- PASS: TestManualSetupTeardown (0.00s)
PASS
```
