---
title: Golang에서 TLS를 사용하는 라이브러리를 사용할 떄 고려할 사항
date: 2024-09-06T19:42:02+09:00
tags: [golang, tls]
---

Golang으로 작성한 프로그램에서 공유 라이브러리를 사용하는데, 공유 라이브러리에서 tls를 사용하는 경우 어떤 문제가 발생할 수 있고, 어떤 해결 방법이 있는지 알아보겠습니다.

## Thread Local Storage

Thread Local Storage (TLS)는 프로그래밍 언어에서 각 스레드가 변수의 고유한 복사본을 가질 수 있는 메커니즘입니다.
이는 각 스레드가 다른 스레드에 영향을 주지 않고 자체 상태를 유지해야 하는 다중 스레드 애플리케이션에서 유용합니다.
TLS는 스레드별 데이터를 저장하고 액세스하고 수정할 수 있는 방법을 제공합니다.

Golang에서 TLS를 사용하는 예제입니다.

```go
package main

// #include <pthread.h>
// static __thread int tls_var = 0;
// void set_tls(int val) { tls_var = val; }
// int get_tls() { return tls_var; }
import "C"

func main() {
  C.set_tls(C.int(x))
  y := int(C.get_tls())
}
```

## Goroutine Work Stealing

하지만 CGO에서 TLS를 사용할 경우 문제가 발생할 수 있습니다.

Golang에서의 Goroutine Work Stealing은 병렬 처리를 위한 스케줄링 메커니즘입니다. 일반적으로, Goroutine은 여러 개의 스레드에 분산되어 실행됩니다.

Goroutine Work Stealing은 스레드 간의 작업 부하를 균형있게 분산시키는 방법입니다. 작업이 완료되지 않은 스레드는 다른 스레드로부터 추가 작업을 훔쳐옵니다.

즉, 하나의 Goroutine이 여러 Thread에서 실행될 수 있습니다.

아래 코드는 대부분의 경우 panic이 발생합니다.

```go
package main

// #include <pthread.h>
// static __thread int tls_var = 0;
// void set_tls(int val) { tls_var = val; }
// int get_tls() { return tls_var; }
import "C"
import "runtime"

func main() {
  for i := range 100 {
    go func(x int) {
      C.set_tls(C.int(x))

      runtime.Gosched()

      y := int(C.get_tls())

      if x != y {
        panic("x != y")
      }
    }(i)
  }
}
```

## runtime.LockOSThread, runtime.UnlockOSThread

이렇게 Go에서 의존해야 하는 외부 라이브러리가 TLS를 사용할 경우 runtime.LockOSThread, runtime.UnlockOSThread를 사용해서 Goroutine이 항상 같은 Thread에서 실행되도록 고정할 수 있습니다.

아래의 예제에서 Goroutine이 같은 Thread에서 실행되도록 고정하면 panic이 발생하지 않습니다.

```go
package main

// #include <pthread.h>
// static __thread int tls_var = 0;
// void set_tls(int val) { tls_var = val; }
// int get_tls() { return tls_var; }
import "C"
import "runtime"

func main() {
  for i := range 100 {
    go func(x int) {
      runtime.LockOSThread()
      defer runtime.UnlockOSThread()

      C.set_tls(C.int(x))

      runtime.Gosched()

      y := int(C.get_tls())

      if x != y {
        panic("x != y")
      }
    }(i)
  }
}
```
