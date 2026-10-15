# 8.4. Safepoint 프로토콜

[GC](../000-meta/007-glossary.md#gc-garbage-collector) 가 일시적으로 STW (Stop-The-World) 작업을 수행하려면 모든 mutator thread 가 안전한 자리에 도달해 있어야 합니다. 이 문서에서는 그러한 동기화를 위한 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 프로토콜을 정의합니다.

## 8.4.1. `yield` flag 검사: fast path

mutator 는 [TLS 슬롯](803-tls-slots.md) 의 `yield_flag` (offset 0, u8) 을 주기적으로 검사합니다. 검사 흐름은 다음과 같이 구성됩니다.

```
poll:
    flag = TLS[yield_flag]
    if flag != 0:
        call rok_runtime_yield()    // slow path
    // 정상 진행
```

이를 LLVM IR 로 표현하면 다음과 같습니다.

```mlir
%flag = load i8, i8* @rok_tls_yield_flag
%check = icmp ne i8 %flag, 0
br i1 %check, label %slow, label %cont

slow:
  call void @rok_runtime_yield()
  br label %cont

cont:
  ; ... 계속
```

이 코드는 2 개 명령어로 이루어진 fast path 와 분기 1 개로 구성되며, 분기는 대부분 `false` 로 평가됩니다.

## 8.4.2. emit 위치

컴파일러는 다음 위치에 safepoint poll 을 emit 합니다.

| 위치                  | 이유                      |
|-----------------------|---------------------------|
| 함수 진입             | 호출 후 즉시 GC 협력 가능 |
| 루프 back-edge        | 무한 루프가 GC 차단 막음  |
| 함수 호출 직후 (가능) | 호출이 길었으면 즉시 협력 |

자세한 emit 정책은 컴파일러가 결정하며, 본 사양에서는 최소 빈도만 다음과 같이 약속합니다.

- 모든 함수에 1 개 이상의 poll 을 둡니다.
- 루프 back-edge 에 1 개 이상의 poll 을 둡니다.

## 8.4.3. `rok_runtime_yield`: slow path

slow path 의 시그니처는 다음과 같습니다.

```
extern "C" {
    fn rok_runtime_yield();
}
```

mutator 는 GC 와 협력하기 위해 이 함수를 호출하며, 함수의 효과는 다음과 같습니다.

1. mutator 의 [Mutator TLS 슬롯 레이아웃](803-tls-slots.md) 에 있는 `mutator_state` 를 `GC_COOPERATING` 로 set 합니다.
2. [shadow stack](../000-meta/007-glossary.md#shadow-stack) walk 등 GC 가 필요로 하는 루트 정보를 노출합니다.
3. GC 가 끝날 때까지 기다립니다.
4. GC 가 끝나면 `mutator_state` 를 다시 `RUNNING` 으로 복귀시킵니다.

`yield_flag` 는 `rok_runtime_yield` 안에 있는 동안 자동으로 클리어되거나, 또는 GC 가 클리어합니다.

## 8.4.4. `yield` flag 의 set / clear

GC 가 STW pause 를 필요로 할 때의 동작은 다음과 같습니다.

```
gc_initiate_stw():
  for each mutator thread:
    atomic store TLS[yield_flag] = 1

  // 모든 mutator 가 yield 할 때까지 wait
  for each mutator thread:
    wait until mutator_state == GC_COOPERATING
```

GC 가 STW 를 끝냈을 때의 동작은 다음과 같습니다.

```
gc_finish_stw():
  for each mutator thread:
    atomic store TLS[yield_flag] = 0
  notify all (resume)
```

## 8.4.5. fast path 비용

mutator 가 부담하는 비용은 `yield` flag 한 번 load 와 분기 1 개로 끝납니다.

대부분의 경우 flag 가 0 이므로 분기는 not taken 으로 처리되며, 현대 CPU 의 분기 예측이 거의 비용 0 으로 처리합니다.

이 정도 비용으로 모든 thread 가 결정론적으로 협력할 수 있는 동기화 메커니즘을 구성합니다.

## 8.4.6. STW 단계

수집 하나는 STW 한 번입니다. 수집을 시작한 thread 가 다른 모든 mutator 를 멈추고 (`GC_COOPERATING`, 또는 blocking 런타임 호출 중이면 `PARKED`), mark / sweep / evacuation 을 마친 뒤 재개시킵니다. 동시 수집 단계는 없습니다 ([7.5](../007-runtime/705-concurrent-collection.md)).

## 8.4.7. Safepoint 의 불변

safepoint 에 도달했을 때, mutator 의 살아있는 모든 GC 객체 변수가 shadow stack 에 등록되어 있어야 합니다.

이 보장은 컴파일러가 [배리어와 루트 추적](../007-runtime/704-barriers-and-roots.md) 의 shadow stack frame 을 함수 진입 시 직접 link 함으로써 이루어집니다. GC 포인터 지역 변수는 frame 의 슬롯에 살기 때문에, 본체에서 값이 바뀌면 슬롯도 함께 바뀝니다.

## 8.4.8. Loop back-edge poll

긴 루프가 GC 와 협력하지 않으면 STW 가 무한히 대기합니다. 그래서 컴파일러는 모든 루프의 back-edge 에 poll 을 emit 합니다.

```
loop:
    body
    poll                              // back-edge poll
    branch loop
```

이 poll 의 비용 역시 위에서 본 것처럼 2 개 명령어와 분기 1 개로, 루프 본체에 비하면 무시할 수 있는 수준입니다.

컴파일러는 LLVM 의 루프 인식을 활용해 모든 자연 루프를 검출한 뒤 poll 을 emit 합니다.

## 8.4.9. inline 함수와 poll

작은 함수가 인라인되면 그 함수의 진입 poll 도 함께 사라질 수 있습니다. 다만 인라인된 결과는 원래 함수의 본체 안에 흡수되므로, 원래 함수의 진입 poll 만으로도 충분합니다.

긴 인라인이 일어나는 경우 (대량 인라인) 에는 LLVM 의 pass 가 추가 poll 을 자동으로 추가합니다 (구현 결정 사항입니다).

## 8.4.10. blocking syscall 의 자동 `yield`

mutator 가 blocking syscall (read, accept 등) 을 호출하면 그 thread 가 멈춰 GC 와 협력하지 못합니다.

이를 해결하기 위해, blocking syscall 진입 직전에 자동으로 `mutator_state` 를 `PARKED` 로 set 합니다. GC 는 PARKED 상태인 mutator 를 이미 협력 중인 것으로 처리합니다.

```
syscall_blocking:
    mutator_state = PARKED
    actual_syscall()
    mutator_state = RUNNING

    // 만약 GC 가 STW 였다면 — yield 처리
    if yield_flag:
        rok_runtime_yield()
```

이 메커니즘은 스케줄러가 emit 하고 처리하며, 본 ABI 에서는 PARKED 상태의 의미만 정의합니다.

## 8.4.11. 사용자 코드의 직접 호출

사용자 코드도 (드물게) `rok_runtime_yield()` 를 직접 호출할 수 있습니다.

```rok
extern "C" {
    fn rok_runtime_yield();
}

fn long_loop() {
    for i in 0..1_000_000 {
        ...
        if i % 1000 == 0 { rok_runtime_yield(); }
    }
}
```

다만 컴파일러가 이미 루프 back-edge 에 poll 을 emit 하므로, 사용자가 직접 호출해야 할 일은 거의 없습니다. 이 패턴은 인라인 어셈블리 안과 같은 특수한 상황에서만 사용합니다.

## 8.4.12. 안정성

`yield_flag` 의 offset (TLS 0, u8) 은 ABI 의 일부이므로 변경할 수 없으며, major `break` 에서만 변경 가능합니다.

`rok_runtime_yield` 의 시그니처는 인자도 반환값도 없는 단순한 호출 규약을 따릅니다.
