# 7.8. 코루틴

[제너레이터](../004-functions/411-generators.md), [비동기](../004-functions/412-async.md), [스트림](../004-functions/413-stream.md)은 같은 코루틴 lowering을 씁니다.

## 7.8.1. 상태 기계 lowering

Rok의 코루틴은 stackless이며, 컴파일러가 직접 만든 상태 기계로 lower합니다. LLVM coroutine intrinsic(`llvm.coro.*`)은 쓰지 않습니다. 컴파일러가 frame 레이아웃을 알아야 frame의 ScanTable을 만들 수 있기 때문입니다. arch별 context-switch 코드는 없으며, 모든 Tier 1 타깃에서 같습니다.

```
yield / async / async yield 블록
  → body 함수로 들어 올림 (HIR, 클로저처럼)
  → 상태 기계 (MIR): resume 함수 + frame 타입
  → 일반 MLIR / LLVM
```

`yield fn` / `async fn`은 본문을 그런 블록으로 감싸고 래퍼 타입을 돌려주는 `fn`입니다.

## 7.8.2. 코루틴 프레임

```
Frame {
    state  : i32        // 다음에 이어갈 suspend 지점
    locals : ...        // suspend 지점을 넘어 살아 있는 지역만
}
```

- frame은 일반 GC 객체이며, 할당 fast / slow path([7.2](702-allocation.md))로 만듭니다. 타입마다 frame의 [ScanTable](../008-abi/802-scan-table.md)이 있습니다.
- 중단된 동안에는 frame의 GC 참조를 ScanTable로 추적하고, resume 함수가 실행되는 동안에는 그 지역 변수를 shadow stack([7.4.1](704-barriers-and-roots.md#741-루트-추적-컴파일러가-직접-emit-하는-shadow-stack))으로 추적합니다.
- 오래 사는 task의 frame은 sticky mark-bit로 old 객체가 됩니다([7.3.2](703-gc-algorithm.md#732-sticky-mark-bit-세대-수집)).
- 호출 결과(`GenIter<T>`, `AsyncFuture<T>`, `AsyncStream<T>`와 그 `Try*` 변형)는 frame을 감싼 값입니다([4.1.11](../004-functions/401-function-declaration.md#코루틴-함수의-결과-타입)).

## 7.8.3. Generator

```
1. count() 호출 → frame 할당, 래퍼 반환.
2. next() → resume 함수가 state 에서 이어 진행. yield 에서 값과 state 를 저장하고 돌아옴.
3. 다음 next() → 저장된 state 에서 이어 진행. 끝나면 none.
```

진행은 호출자가 주도합니다(`for x in g { .. }`).

## 7.8.4. Async

```
1. fetch() 호출 → frame 할당, Future 반환.
2. poll → resume 함수가 진행. await 한 future 가 Pending 이면 state 를 저장하고 Pending.
3. 깨움 (parking registry, I/O driver, timer) → task 가 다시 큐에 들어가고 worker 가 poll.
```

진행은 스케줄러와 런타임이 주도합니다([7.6](706-scheduler.md)).

## 7.8.5. Stream

`async yield`는 둘을 합친 것입니다. 같은 frame 안에서 `yield`는 값을 내고, `await`는 Pending으로 중단합니다.

## 7.8.6. 할당 생략

frame은 코루틴을 호출할 때마다 할당합니다. 호출자가 끝까지 진행하는 코루틴을 위한 전용 생략은 없습니다. 다만 밖으로 빠져나가지 않는 frame은 다른 할당처럼 없어질 수 있습니다([7.2.12](702-allocation.md#7212-할당-생략)).

## 7.8.7. `yield` / `await` 의 lowering

| 키워드         | lowering                                                                  |
|----------------|---------------------------------------------------------------------------|
| `yield expr;`  | state 저장 + 값 반환 (Iterator) / `Ready(값)` 반환 (Stream)               |
| `await expr;`  | 대상 future poll. Pending이면 state 저장 + `Pending` 반환                  |
| `return expr;` | 상태 기계 종료 + 결과 반환                                                 |

## 7.8.8. 런타임 심볼

`await`의 대기와 깨움은 `rok::core::task`의 parking registry가 맡습니다(`rok::core::task`가 `rok_task_hooks_install`로 설치한 hook, [8.5.4](../008-abi/805-extern-symbols.md#854-scheduler-영역)). 런타임의 `rok_coro_frame_alloc` / `rok_coro_park` / `rok_coro_unpark`는 C와 저수준 코드용이며, 컴파일러는 쓰지 않습니다([8.5](../008-abi/805-extern-symbols.md)).

## 7.8.9. 스택 깊이

중첩된 `await`는 바깥 resume 함수가 안쪽 future의 poll을 호출하는 native 호출로 진행합니다. 그래서 `await` 재귀의 깊이는 native stack 크기에 묶이며, 아주 깊은 재귀는 stack overflow를 일으킵니다([런타임 오류](README.md#런타임-오류-스택-오버플로)). 깊은 재귀가 필요하면 task로 나누세요.

## 7.8.10. 코루틴 프레임의 수명

frame은 다른 GC 객체처럼 도달 가능한 동안 살아 있습니다. 실행 중인 task는 task 큐가, 대기 중인 task는 parking registry가, 결과를 기다리는 쪽은 자기 frame이 참조합니다. 어디서도 참조하지 않으면 GC가 회수합니다. 명시적 해제는 없습니다.
