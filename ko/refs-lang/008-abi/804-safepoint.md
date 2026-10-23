# 8.4. Safepoint 프로토콜

[GC](../000-meta/007-glossary.md#gc-garbage-collector) 가 일시적으로 STW (Stop-The-World) 작업을 수행하려면 모든 mutator thread 가 안전한 자리에 도달해 있어야 합니다. 이 문서에서는 그러한 동기화를 위한 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 프로토콜을 정의합니다.

## 8.4.1. `yield` flag 검사: fast path

mutator 는 [TLS 슬롯](803-tls-slots.md) `rok_tls_yield_flag` (u8) 을 주기적으로 검사합니다. 검사 흐름은 다음과 같이 구성됩니다.

```
poll:
    flag = TLS[yield_flag]
    if flag != 0:
        call rok_runtime_yield()    // slow path
    // 정상 진행
```

이를 LLVM IR 로 표현하면 다음과 같습니다.

```mlir
%t = call ptr @llvm.threadlocal.address(ptr @rok_tls)
%flag_p = getelementptr i8, ptr %t, i64 88    ; yield_flag at 10p+8, p = 8
%flag = load i8, ptr %flag_p
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
| 함수 진입 (또는 각 경로의 첫 safepoint 앞) | 호출 후 즉시 GC 협력 가능 |
| 루프 back-edge        | 무한 루프가 GC 차단 막음  |
| 함수 호출 직후 (가능) | 호출이 길었으면 즉시 협력 |

자세한 emit 정책은 컴파일러가 결정하며, 본 사양에서는 최소 빈도만 다음과 같이 약속합니다.

- 모든 루프 back-edge 에 poll 을 둡니다. 단, 반복 횟수가 64 이하의 컴파일 시점 상수이고 본체가 safepoint 에 닿지 않는 루프는 예외입니다. 모든 호출 순환에 poll 이 들어갑니다.
- safepoint 에 닿을 수 없는 함수 (그런 유한 루프 말고는 루프가 없고, 할당, 간접 호출, lazy `static`, safepoint 에 닿을 수 있는 함수의 호출이 없음) 에는 poll 이 필요 없습니다.

이 둘이 함께 yield 요청에서 다음 safepoint 까지의 시간을 유한하게 묶습니다.

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

STW 수집 (minor, evacuation 이 있는 full, `rok_gc_collect`) 은 pause 한 번입니다. 수집을 시작한 thread 가 다른 모든 mutator 를 멈추고 (`GC_COOPERATING`, 또는 blocking 런타임 호출 중이면 `PARKED`), 수집을 마친 뒤 재개시킵니다.

동시 mark 를 하는 full 수집은 시작 pause 와 마지막 pause 두 번입니다 ([7.5.2](../007-runtime/705-concurrent-collection.md#752-단계)). 그 사이에는 marker thread 하나가 mutator 와 함께 돕니다.

## 8.4.7. Safepoint 의 불변

safepoint 에 도달했을 때, mutator 의 살아있는 모든 GC 객체 변수가 shadow stack 에 등록되어 있어야 합니다.

이 보장은 컴파일러가 [배리어와 루트 추적](../007-runtime/704-barriers-and-roots.md) 의 shadow stack frame 을 함수를 지나는 각 경로의 첫 safepoint 앞에서 (진입 시, 또는 safepoint 에 닿는 경로에서는 그보다 나중에) link 함으로써 이루어집니다. safepoint 에 닿지 않는 경로는 link 하지 않아도 되고, frame 은 link 된 곳에서만 반환 시 unlink 합니다. 마찬가지로 직접 rok 호출로만 들어오는 함수는 진입 poll 을 link 바로 뒤로 옮길 수 있습니다. 호출 순환을 닫는 호출이 safepoint 이므로 모든 호출 순환에는 여전히 poll 이 있습니다. C 나 함수 값으로 호출되는 함수는 진입 시 poll 합니다. safepoint 에서 GC 참조를 가진 지역 변수는 frame 에 등록되며, 등록된 word 는 쓸 때마다 최신으로 유지합니다. back-edge poll 만 보는 지역 변수는 poll 의 slow path 가 런타임 호출 동안만 등록해도 됩니다 ([7.4.1](../007-runtime/704-barriers-and-roots.md#741-루트-추적-컴파일러가-직접-emit-하는-shadow-stack) 규칙 4). 그래서 poll 말고는 safepoint 가 없는 루프는 도는 동안 frame 을 link 하지 않고 루트 word 도 쓰지 않습니다. 수집은 루트 word 를 다시 쓰지 않으므로 (루트에서 바로 닿는 객체는 옮기지 않음, [7.4.1](../007-runtime/704-barriers-and-roots.md#741-루트-추적-컴파일러가-직접-emit-하는-shadow-stack)) 컴파일러는 (레지스터에) 사본을 따로 둘 수도 있습니다.

## 8.4.8. Loop back-edge poll

긴 루프가 GC 와 협력하지 않으면 STW 가 무한히 대기합니다. 그래서 컴파일러는 모든 루프의 back-edge 에 poll 을 emit 합니다 (8.4.2 의 유한 루프는 예외).

```
loop:
    body
    poll                              // back-edge poll
    branch loop
```

이 poll 의 비용 역시 위에서 본 것처럼 2 개 명령어와 분기 1 개로, 루프 본체에 비하면 무시할 수 있는 수준입니다.

poll 은 back edge 에 남습니다. 그 slow path (`rok_runtime_yield` 호출) 는 edge 로 돌아오는 대신 루프 header 앞의 재진입 블록으로 분기할 수 있습니다.

컴파일러는 모든 자연 루프를 검출해, 8.4.2 의 유한 루프를 뺀 각각에 poll 을 emit 합니다.

## 8.4.9. inline 함수와 poll

작은 함수가 인라인되면 그 함수의 진입 poll 도 함께 사라질 수 있습니다. 다만 인라인된 결과는 원래 함수의 본체 안에 흡수되므로, 원래 함수의 진입 poll 만으로도 충분합니다.

긴 인라인이 일어나는 경우 (대량 인라인) 에는 LLVM 의 pass 가 추가 poll 을 자동으로 추가합니다 (구현 결정 사항입니다).

## 8.4.10. native 호출

C 나 blocking 런타임 호출 안의 스레드는 safepoint 에 닿을 수 없으므로 `PARKED` 로 셉니다. GC 는 이를 이미 협력 중인 것으로 봅니다.

- 컴파일러는 모든 `extern "C"` 호출을 감쌉니다. 호출 동안 `PARKED`, 뒤에 `RUNNING`, 그다음 yield flag 를 봅니다. 호출자의 인자 루트는 그 shadow-stack frame 에 남습니다. fast path 는 inline 이고, `rok_enter_native` / `rok_leave_native` 가 slow path 입니다 ([8.5.2](805-extern-symbols.md#852-runtime-영역)).
- 모든 `extern "C" fn` 정의는 `rok_native_reenter` 로 시작해 `rok_native_resume` 로 끝나며, C callback 이 mutator 로 돕니다.
- 런타임은 자기의 blocking 호출 (I/O 대기, `rok_sync_wait`, join, sleep) 을 같은 방식으로 감싸며, GC 객체에는 닿지 않고 호출자의 frame 이 살려 두는 buffer 에만 닿는 긴 계산 (`int` 의 곱셈, 나눗셈, gcd kernel) 도 그렇게 감쌉니다. 그것들의 `extern "rok"` 선언은 `#[no_safepoint]` ([5.9.2](../005-modules/509-extern-block.md#592-abi-명시)) 가 아니므로, 그 호출은 safepoint 이며 buffer 는 root 됩니다.
- `rok_*` 심볼 ([8.5](805-extern-symbols.md), 런타임의 예약 접두사) 의 `extern "C"` 선언을 통한 호출은 감싸지 않습니다: 런타임 함수는 mutator 로 실행됩니다.

```
call_extern_c:
    mutator_state = PARKED      // seq_cst
    c_function()
    mutator_state = RUNNING     // seq_cst
    if yield_flag:
        rok_leave_native()      // 진행 중인 수집을 기다림
```

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

`rok_runtime_yield` 는 `rok_*` 심볼이므로 호출을 감싸지 않으며 (8.4.10), 스레드는 실행 중인 mutator 로서 양보합니다. 다만 컴파일러가 이미 루프 back-edge 에 poll 을 emit 하므로, 사용자가 직접 호출해야 할 일은 거의 없습니다. 이 패턴은 인라인 어셈블리 안과 같은 특수한 상황에서만 사용합니다.

## 8.4.12. 안정성

ABI 는 `rok_tls` 의 offset 10p+8 슬롯 (`rok_tls_yield_flag`) 과 그 타입 (u8) 입니다 ([8.3.1](803-tls-slots.md#831-tls-슬롯-구조)).

`rok_runtime_yield` 의 시그니처는 인자도 반환값도 없는 단순한 호출 규약을 따릅니다.
