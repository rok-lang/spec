# 8.4. Safepoint 프로토콜

[GC](../000-meta/007-glossary.md#gc-garbage-collector)가 일시적으로 STW(Stop-The-World) 작업을 하려면 모든 mutator thread가 안전한 지점에 도달해 있어야 합니다. 이 문서는 그 동기화를 위한 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 프로토콜을 정의합니다.

## 8.4.1. `yield` flag 검사: fast path

mutator는 [TLS 슬롯](803-tls-slots.md) `rok_tls_yield_flag`(u8)를 주기적으로 검사합니다. 검사 흐름은 다음과 같습니다.

```
poll:
    flag = TLS[yield_flag]
    if flag != 0:
        call rok_runtime_yield()    // slow path
    // 정상 진행
```

LLVM IR로는 다음과 같습니다.

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

명령어 2개짜리 fast path와 분기 1개이며, 분기는 거의 항상 `false`입니다.

## 8.4.2. emit 위치

컴파일러는 다음 위치에 safepoint poll을 emit합니다.

| 위치                  | 이유                      |
|-----------------------|---------------------------|
| 함수 진입 (또는 각 경로의 첫 safepoint 앞) | 호출 후 즉시 GC 협력 가능 |
| 루프 back-edge        | 무한 루프가 GC 차단 막음  |
| 함수 호출 직후 (가능) | 호출이 길었으면 즉시 협력 |

자세한 emit 정책은 컴파일러가 정하며, 이 사양은 다음 최소 빈도만 보장합니다.

- 모든 루프 back-edge에 poll을 둡니다. 다만 반복 횟수가 64 이하인 컴파일 타임 상수이고 본체가 safepoint에 닿지 않는 루프는 예외입니다. 모든 호출 순환에는 poll이 들어갑니다.
- safepoint에 닿을 수 없는 함수(그런 유한 루프 말고는 루프가 없고, 할당, 간접 호출, lazy `static`, safepoint에 닿을 수 있는 함수의 호출이 없는 함수)에는 poll이 필요 없습니다.

이 두 규칙이 함께 yield 요청부터 다음 safepoint까지의 시간에 상한을 둡니다.

## 8.4.3. `rok_runtime_yield`: slow path

slow path의 시그니처는 다음과 같습니다.

```
extern "C" {
    fn rok_runtime_yield();
}
```

mutator는 GC와 협력하려고 이 함수를 호출합니다. 효과는 다음과 같습니다.

1. mutator의 [Mutator TLS 슬롯 레이아웃](803-tls-slots.md)에 있는 `mutator_state`를 `GC_COOPERATING`으로 set합니다.
2. [shadow stack](../000-meta/007-glossary.md#shadow-stack) walk 등 GC에 필요한 루트 정보를 노출합니다.
3. GC가 끝날 때까지 기다립니다.
4. GC가 끝나면 `mutator_state`를 다시 `RUNNING`으로 되돌립니다.

`yield_flag`는 `rok_runtime_yield` 안에 있는 동안 자동으로 지워지거나, GC가 지웁니다.

## 8.4.4. `yield` flag 의 set / clear

GC가 STW pause를 필요로 할 때는 다음과 같이 동작합니다.

```
gc_initiate_stw():
  for each mutator thread:
    atomic store TLS[yield_flag] = 1

  // 모든 mutator 가 yield 할 때까지 wait
  for each mutator thread:
    wait until mutator_state == GC_COOPERATING
```

GC가 STW를 끝낼 때는 다음과 같이 동작합니다.

```
gc_finish_stw():
  for each mutator thread:
    atomic store TLS[yield_flag] = 0
  notify all (resume)
```

## 8.4.5. fast path 비용

mutator가 치르는 비용은 `yield` flag load 한 번과 분기 1개입니다.

거의 모든 경우 flag가 0이므로 분기는 not taken이고, 현대 CPU의 분기 예측기가 거의 비용 없이 처리합니다.

이 비용으로 모든 thread가 결정론적으로 협력하는 동기화 메커니즘을 얻습니다.

## 8.4.6. STW 단계

STW 수집(minor, evacuation이 있는 full, `rok_gc_collect`)은 pause 한 번입니다. 수집을 시작한 thread가 다른 모든 mutator를 멈추고(`GC_COOPERATING`, 또는 blocking 런타임 호출 중이면 `PARKED`), 수집을 마친 뒤 다시 실행시킵니다.

동시 mark를 하는 full 수집은 시작 pause와 마지막 pause, 두 번입니다([7.5.2](../007-runtime/705-concurrent-collection.md#752-단계)). 그 사이에는 marker thread 하나가 mutator와 함께 돕니다.

## 8.4.7. Safepoint 의 불변

safepoint에 도달했을 때 mutator의 살아 있는 모든 GC 객체 변수는 shadow stack에 등록되어 있어야 합니다.

컴파일러는 [배리어와 루트 추적](../007-runtime/704-barriers-and-roots.md)의 shadow stack frame을 함수를 지나는 각 경로의 첫 safepoint 앞에서(진입 시, 또는 safepoint에 닿는 경로에서는 그보다 나중에) link해서 이를 보장합니다. safepoint에 닿지 않는 경로는 link하지 않아도 되고, frame은 link한 곳에서만 반환 시 unlink합니다. 마찬가지로 직접 Rok 호출로만 들어오는 함수는 진입 poll을 link 바로 뒤로 옮길 수 있습니다. 호출 순환을 닫는 호출이 safepoint이므로 모든 호출 순환에는 여전히 poll이 있습니다. C나 함수 값으로 호출되는 함수는 진입 시 poll합니다. safepoint에서 GC 참조를 가진 지역 변수는 frame에 등록하고, 등록한 word는 쓸 때마다 최신으로 유지합니다. back-edge poll에서만 보이는 지역 변수는 poll의 slow path가 런타임 호출 동안만 등록해도 됩니다([7.4.1](../007-runtime/704-barriers-and-roots.md#741-루트-추적-컴파일러가-직접-emit-하는-shadow-stack) 규칙 4). 그래서 poll 말고는 safepoint가 없는 루프는 도는 동안 frame을 link하지 않고 루트 word도 쓰지 않습니다. 수집은 루트 word를 다시 쓰지 않으므로(루트에서 바로 닿는 객체는 옮기지 않음, [7.4.1](../007-runtime/704-barriers-and-roots.md#741-루트-추적-컴파일러가-직접-emit-하는-shadow-stack)) 컴파일러는 (레지스터에) 사본을 따로 둘 수도 있습니다.

## 8.4.8. Loop back-edge poll

긴 루프가 GC와 협력하지 않으면 STW가 끝없이 기다립니다. 그래서 컴파일러는 모든 루프의 back-edge에 poll을 emit합니다(8.4.2의 유한 루프는 예외).

```
loop:
    body
    poll                              // back-edge poll
    branch loop
```

이 poll의 비용도 앞에서 본 것처럼 명령어 2개와 분기 1개이며, 루프 본체에 비하면 무시할 만합니다.

poll은 back edge에 남습니다. poll의 slow path(`rok_runtime_yield` 호출)는 edge로 돌아오는 대신 루프 header 앞의 재진입 블록으로 분기할 수 있습니다.

컴파일러는 모든 자연 루프를 찾아, 8.4.2의 유한 루프를 뺀 각 루프에 poll을 emit합니다.

## 8.4.9. inline 함수와 poll

작은 함수가 인라인되면 그 함수의 진입 poll도 함께 사라질 수 있습니다. 하지만 인라인된 코드는 호출한 함수의 본체에 흡수되므로, 호출한 함수의 진입 poll로 충분합니다.

대량 인라인이 일어나면 LLVM pass가 poll을 자동으로 더 넣습니다(구현이 정하는 사항).

## 8.4.10. native 호출

C나 blocking 런타임 호출 안에 있는 스레드는 safepoint에 닿을 수 없으므로 `PARKED`로 셉니다. GC는 이 스레드를 이미 협력 중인 것으로 봅니다.

- 컴파일러는 모든 `extern "C"` 호출을 감쌉니다. 호출 동안은 `PARKED`, 호출 뒤에는 `RUNNING`이며, 그다음 yield flag를 확인합니다. 호출자의 인자 루트는 호출자의 shadow-stack frame에 남습니다. fast path는 inline이고, `rok_enter_native` / `rok_leave_native`가 slow path입니다([8.5.2](805-extern-symbols.md#852-runtime-영역)).
- 모든 `extern "C" fn` 정의는 `rok_native_reenter`로 시작해 `rok_native_resume`로 끝나므로, C callback은 mutator로 실행됩니다.
- 런타임은 자신의 blocking 호출(I/O 대기, `rok_sync_wait`, join, sleep)을 같은 방식으로 감쌉니다. GC 객체에는 닿지 않고 호출자의 frame이 살려 두는 buffer에만 닿는 긴 계산(`int`의 곱셈, 나눗셈, gcd kernel)도 마찬가지로 감쌉니다. 이들의 `extern "rok"` 선언은 `#[no_safepoint]`([5.9.2](../005-modules/509-extern-block.md#592-abi-명시))가 아니므로, 이들을 호출하는 곳은 safepoint이고 buffer는 root됩니다.
- `#[no_safepoint]` 런타임 함수 호출은 감싸지 않습니다. debug 빌드의 rokc는 그 호출을 `rok_nosp_enter()` / `rok_nosp_leave()`([8.5.2](805-extern-symbols.md#852-runtime-영역))로 감싸 스레드의 `no_safepoint` 깊이([8.3.1](803-tls-slots.md#831-tls-슬롯-구조))를 올리고 내립니다. 깊이가 0이 아닌 동안 스레드가 수집을 위해 멈추려 하면 debug 런타임은 fatal 에러로 실패합니다. release rokc는 두 호출 모두 내보내지 않습니다.
- `rok_*` 심볼([8.5](805-extern-symbols.md), 런타임의 예약 접두사)의 `extern "C"` 선언을 거치는 호출은 감싸지 않습니다. 런타임 함수는 mutator로 실행됩니다.

```
call_extern_c:
    mutator_state = PARKED      // seq_cst
    c_function()
    mutator_state = RUNNING     // seq_cst
    if yield_flag:
        rok_leave_native()      // 진행 중인 수집을 기다림
```

## 8.4.11. 사용자 코드의 직접 호출

사용자 코드도 (드물게) `rok_runtime_yield()`를 직접 호출할 수 있습니다.

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

`rok_runtime_yield`는 `rok_*` 심볼이므로 호출을 감싸지 않으며(8.4.10), 스레드는 실행 중인 mutator로서 양보합니다. 컴파일러가 이미 루프 back-edge에 poll을 emit하므로 사용자가 직접 호출할 일은 거의 없습니다. 이 패턴은 인라인 어셈블리 안 같은 특수한 상황에서만 씁니다.

## 8.4.12. 안정성

ABI에 속하는 것은 `rok_tls`의 offset 10p+8 슬롯(`rok_tls_yield_flag`)과 그 타입(u8)입니다([8.3.1](803-tls-slots.md#831-tls-슬롯-구조)).

`rok_runtime_yield`는 인자도 반환값도 없는 단순한 호출 규약을 따릅니다.
