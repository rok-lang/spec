# 8.4. Safepoint 프로토콜

[GC](../000-meta/007-glossary.md#gc-garbage-collector) 가 일시 STW (Stop-The-World) 작업을 하려면 모든 mutator thread 가 안전한 자리에
도달해야 한다. 본 문서는 그 동기화 프로토콜의 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 를 정의한다.

## 8.4.1. `yield` flag 검사: fast path

[TLS 슬롯](803-tls-slots.md) 의 `yield_flag` (offset 0, u8) 를 mutator 가 주기적으로 검사한다.

```
poll:
    flag = TLS[yield_flag]
    if flag != 0:
        call nok_runtime_yield()    // slow path
    // 정상 진행
```

LLVM IR:

```mlir
%flag = load i8, i8* @nok_tls_yield_flag
%check = icmp ne i8 %flag, 0
br i1 %check, label %slow, label %cont

slow:
  call void @nok_runtime_yield()
  br label %cont

cont:
  ; ... 계속
```

이 코드는 2 명령어 fast path 와 분기 1 개 (대부분 `false`).

## 8.4.2. 어디에 emit 되는가

컴파일러가 다음 위치에 safepoint poll 을 emit 한다.

| 위치                  | 이유                      |
| --------------------- | ------------------------- |
| 함수 진입             | 호출 후 즉시 GC 협력 가능 |
| 루프 back-edge        | 무한 루프가 GC 차단 막음  |
| 함수 호출 직후 (가능) | 호출이 길었으면 즉시 협력 |

자세한 emit 정책은 컴파일러의 결정. 본 사양은 최소 빈도만 약속한다.

- 모든 함수에 1 개 이상의 poll.
- 루프 back-edge 에 1 개 이상의 poll.

## 8.4.3. `nok_runtime_yield`: slow path

```
extern "C" {
    fn nok_runtime_yield();
}
```

mutator 가 GC 와 협력하기 위해 호출한다. 함수의 효과:

1. mutator 의 [Mutator TLS 슬롯 레이아웃](803-tls-slots.md) 의 `mutator_state` 를 `GC_COOPERATING` 로 set.
2. [shadow stack](../000-meta/007-glossary.md#shadow-stack) walk 등 GC 가 필요한 루트 정보 노출.
3. GC 가 끝날 때까지 wait (또는 GC worker 역할 참여).
4. GC 끝나면 `mutator_state` = `RUNNING` 복귀.

`yield_flag` 는 `nok_runtime_yield` 안에 있을 때 자동 클리어 (또는 GC 가 클리어).

## 8.4.4. `yield` flag 의 set / clear

GC 가 STW pause 가 필요할 때:

```
gc_initiate_stw():
  for each mutator thread:
    atomic store TLS[yield_flag] = 1

  // 모든 mutator 가 yield 할 때까지 wait
  for each mutator thread:
    wait until mutator_state == GC_COOPERATING
```

GC 가 STW 끝났을 때:

```
gc_finish_stw():
  for each mutator thread:
    atomic store TLS[yield_flag] = 0
  notify all (resume)
```

## 8.4.5. fast path 비용

mutator 가 받는 비용은 `yield` flag 한 번 load + 분기 1 개.

대부분의 경우 flag = 0 이라 분기는 not taken. 현대 CPU 의 분기 예측이 거의 비용 0 으로 처리한다.

이 비용으로 모든 thread 가 결정론적으로 협력하는 동기화 메커니즘을 구성한다.

## 8.4.6. STW 단계의 종류

GC 사이클 안에 짧은 STW 가 여러 번 일어날 수 있다 ([동시 수집](../007-runtime/705-concurrent-collection.md)).

| 단계             | 목표 시간 |
| ---------------- | --------- |
| Initial mark     | ~ 10 ms   |
| Final mark       | ~ 1 ms    |
| Initial evacuate | ~ 1 ms    |
| Final fixup      | < 1 ms    |

각 STW 시작 / 끝에 `yield` flag set / clear.

mutator 는 각 STW 동안 `nok_runtime_yield` 안에서 [block](../000-meta/007-glossary.md#block-블록). STW 가 짧기 때문에 사용자가
인지하지 않는다.

## 8.4.7. Safepoint 의 불변

safepoint 에 도달했을 때 mutator 의 모든 살아있는 GC 객체 변수가 shadow stack 에 등록되어
있어야 한다.

이 보장은 컴파일러가 [배리어와 루트 추적](../007-runtime/704-barriers-and-roots.md) 의 `llvm.gcroot` 를 함수 진입 시 emit 함으로써
이루어진다. LLVM 백엔드가 모든 gcroot 변수를 shadow stack 에 push 한 상태로 만든다.

함수 본체 안에서도 gcroot 변수의 값이 변하면 shadow stack 슬롯도 동기화. LLVM 의 코드
생성이 이를 자동 처리한다.

## 8.4.8. Loop back-edge poll

긴 루프가 GC 협력 안 하면 STW 가 무한히 대기한다. 그래서 컴파일러는 모든 루프의 back-edge
에 poll 을 emit 한다.

```
loop:
    body
    poll                              // back-edge poll
    branch loop
```

이 poll 의 비용은 위와 같이 2 명령어 + 분기. 루프 본체에 비해 무시 가능하다.

LLVM 의 루프 인식을 활용해 컴파일러가 모든 자연 루프를 검출해 poll emit 한다.

## 8.4.9. inline 함수와 poll

작은 함수가 인라인되면 그 함수의 진입 poll 도 사라질 수 있다. 다만 inline 결과가 원래
함수의 본체 안에 흡수되므로, 원래 함수의 진입 poll 이 충분하다.

긴 인라인이 일어나는 경우 (대량 인라인) 추가 poll 을 LLVM 의 pass 가 자동 추가한다 (구현
결정).

## 8.4.10. blocking syscall 의 자동 `yield`

mutator 가 blocking syscall (read, accept 등) 을 하면 그 thread 가 멈춰 GC 와 협력 못 한다.

해결: blocking syscall 진입 직전 자동으로 `mutator_state = PARKED` set. GC 는 PARKED
mutator 를 이미 협력 중으로 처리한다.

```
syscall_blocking:
    mutator_state = PARKED
    actual_syscall()
    mutator_state = RUNNING

    // 만약 GC 가 STW 였다면 — yield 처리
    if yield_flag:
        nok_runtime_yield()
```

이 메커니즘은 스케줄러가 emit 하고 처리한다. 본 ABI 는 PARKED 상태의 의미만 정의한다.

## 8.4.11. 사용자 코드의 직접 호출

`nok_runtime_yield()` 는 사용자 코드도 직접 호출 가능하다 (드물게).

```nok
extern "C" {
    fn nok_runtime_yield();
}

fn long_loop() {
    for i in 0..1_000_000 {
        ...
        if i % 1000 == 0 { nok_runtime_yield(); }
    }
}
```

다만 컴파일러가 이미 루프 back-edge 에 poll 을 emit 하므로, 사용자가 직접 호출할 필요는 거의
없다. 이 패턴은 특수한 상황 (인라인 어셈블리 안 등) 에만 사용한다.

## 8.4.12. 안정성

`yield_flag` 의 offset (TLS 0, u8) 은 ABI 의 일부, 변경 불가 (major `break` 만).

`nok_runtime_yield` 의 시그니처는 인자 없음, 반환 없음. 단순한 호출 규약.
