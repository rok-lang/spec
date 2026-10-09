# 8.5. Extern "C" 심볼

이 절에서는 `librokr.a` 가 외부에 노출하는 모든 [extern "C" 심볼](../000-meta/007-glossary.md#extern-c-심볼) 의 시그니처를 정의합니다. 컴파일러가 생성한 코드는 이 심볼들을 호출하게 됩니다.

> [!IMPORTANT]
> 모든 심볼은 `rok_` 접두사로 시작합니다. 이는 외부 라이브러리나 사용자 코드와의 심볼 충돌을 회피하기 위함입니다.

## 8.5.0. Thread-safety 분류

각 심볼의 정의 옆에는 **Thread-safety** 표기가 따라붙습니다. 이는 컴파일러가 emit 한 코드와 런타임 사이의 contract 의 일부로, 어떤 호출 컨텍스트에서 안전하게 호출할 수 있는지를 명시합니다. 사용 가능한 분류는 다음 6 가지입니다.

| 분류                  | 의미                                                                         |
|-----------------------|------------------------------------------------------------------------------|
| `thread-safe`         | 다중 thread 가 동시에 호출해도 안전 (런타임이 내부 동기화)                   |
| `caller-thread-only`  | 호출자 thread 의 컨텍스트에 묶임 — 다른 thread 가 그 호출자를 대신할 수 없음 |
| `process-init`        | 프로세스 시작 시점, mt 진입 전 단 한 번                                      |
| `process-shutdown`    | 프로세스 종료 시점, mt 종료 후 단 한 번                                      |
| `per-thread-init`     | 각 worker thread 의 시작 시점에 그 thread 가 자기 자신에 대해 한 번          |
| `per-thread-shutdown` | 각 worker thread 의 종료 시점에 그 thread 가 자기 자신에 대해 한 번          |

`caller-thread-only` 는 함수의 효과가 호출자의 TLS 슬롯이나 stack frame 에 결합되어 있어 다른 thread 에서 그 호출자를 대신할 수 없다는 뜻입니다. 함수 자체가 reentrant 하지 않다는 의미는 아니며, 다중 thread 가 각자 자기 호출자에 대해 동시에 호출하는 것은 허용됩니다.

## 8.5.1. GC 영역

### `rok_gc_alloc_slow`

[할당](../007-runtime/702-allocation.md) 의 [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) slow path 에 해당합니다. **Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_gc_alloc_slow(size: usize, align: usize, type_id: u32) -> Ptr<u8>;
}
```

각 인자와 반환값의 의미는 다음과 같습니다.

- `size`: 객체의 총 크기로, 16 바이트의 헤더에 페이로드를 더한 뒤 정렬한 값입니다.
- `align`: 객체의 정렬 요구로, 보통 8 이며 `#[repr(align(N))]` 가 지정되면 더 커집니다.
- `type_id`: 객체의 타입 ID 입니다.
- 반환: 헤더 자리부터 시작하는 포인터를 돌려주며, type_id 와 gc_flags 는 함수가 채워둡니다.

null 을 반환하는 경우는 out of memory 와 같은 극단적 상황입니다. 실제로는 fatal error 를 던지게 되며, 이는 구현이 결정합니다.

### `rok_gc_alloc_bytes`

헤더 뒤에 `size` 바이트의 payload 를 가진 GC 객체를 할당합니다. **Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_gc_alloc_bytes(size: usize, traced: bool) -> str;
}
```

- `traced = false` 이면 type id `1` (GC 문자열), `true` 이면 type id `2` (word-traced buffer) 로 할당합니다 ([8.2.12](802-scan-table.md#8212-런타임-예약-type-id)).
- 반환값은 payload 를 가리키는 `(ptr, len)` 쌍 (`str` 의 표현, [8.8](808-lowering-types.md)) 입니다.

### `rok_gc_wb_slow`

[배리어와 루트 추적](../007-runtime/704-barriers-and-roots.md) 의 write barrier slow path 에 해당합니다. **Thread-safety: `caller-thread-only`** (TLS 의 wb log buffer 에 기록).

```
extern "C" {
    fn rok_gc_wb_slow(obj: Ptr<u8>);
}
```

인자 `obj` 는 변경된 슬롯이 속한 객체의 포인터이며, 별도의 반환값은 없습니다. 효과로는 remembered set 에 obj 를 추가하고, log 가 차면 flush 합니다.

### `rok_gc_register_type`

[ScanTable](802-scan-table.md) 의 타입을 등록합니다. **Thread-safety: `process-init`**.

```
extern "C" {
    fn rok_gc_register_type(type_id: u32, scan_table: ConstPtr<u8>);
}
```

호출 시점은 프로세스 시작 시점이며, mt 진입 전이므로 동기화 불필요.

### `rok_gc_pin` / `rok_gc_unpin`

[FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 객체의 pinning 을 담당합니다 ([객체 레이아웃과 메모리 공간](../007-runtime/701-object-layout.md), [`extern` 블록 (FFI 선언)](../005-modules/509-extern-block.md) 참조). **Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_gc_pin(obj: Ptr<u8>) -> Ptr<u8>;       // 새 위치 (Non-Moving 공간) 반환
    fn rok_gc_unpin(obj: Ptr<u8>);
}
```

pin 은 객체를 Non-Moving 공간으로 옮겨 외부 코드가 들고 있는 동안 안전하게 유지합니다. unpin 은 객체를 다시 일반 [Immix](../000-meta/007-glossary.md#immix) 공간으로 돌릴 수 있도록 하며, 그대로 두는 것도 가능합니다.

### `rok_gc_collect`

명시적인 [GC](../000-meta/007-glossary.md#gc-garbage-collector) 트리거이며, 드물게 사용됩니다. **Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_gc_collect();
}
```

호출하면 즉시 GC 사이클이 시작됩니다. 동시 수집 방식이므로 호출자도 mutator 로서 계속 작동하게 됩니다.

## 8.5.2. Runtime 영역

### `rok_runtime_init`

런타임을 초기화합니다. **Thread-safety: `process-init`**.

```
extern "C" {
    fn rok_runtime_init();
}
```

호출 시점은 프로세스 시작 직후이자 `main()` 이전이며, 효과로는 worker 풀을 생성하고 GC 공간을 mmap 하며 I/O driver 를 초기화합니다. 컴파일러가 생성한 C-ABI `main` 은 이 호출 직후에 `rok.register_types` 를 호출합니다 ([8.2.13](802-scan-table.md#8213-등록-시점)).

런타임은 이 밖에 thread-local 데이터 심볼 `rok_tls_shadow_stack_top` 을 export 합니다 ([8.3.8](803-tls-slots.md#838-컴파일러-emit-의-형식)).

### `rok_runtime_shutdown`

런타임을 종료합니다. **Thread-safety: `process-shutdown`**.

```
extern "C" {
    fn rok_runtime_shutdown();
}
```

호출 시점은 `main()` 이 끝난 직후이며, 효과로는 worker 풀을 종료하고 미해제 자원을 정리합니다.

### `rok_exit`

프로세스를 즉시 종료합니다. **Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_exit(code: i32) -> !;
}
```

[`rok::std::env::exit`](../../refs-standard-libraries/300-std/306-env.md) 가 이 심볼을 부릅니다. `finally` 와 `Drop` 은 실행하지 않습니다 ([4.16.4](../004-functions/416-entry-point.md#4164-종료-코드와-미처리-예외)).

### `rok_runtime_yield`

[Safepoint 프로토콜](804-safepoint.md) 의 `yield` 함수에 해당합니다. **Thread-safety: `caller-thread-only`** (호출자 mutator 가 자기 yield_flag 를 polling 한 결과로 호출).

```
extern "C" {
    fn rok_runtime_yield();
}
```

### `rok_thread_init` / `rok_thread_shutdown`

worker thread 의 lifecycle 을 담당합니다. **Thread-safety: `rok_thread_init` 은 `per-thread-init`, `rok_thread_shutdown` 은 `per-thread-shutdown`**.

```
extern "C" {
    fn rok_thread_init();
    fn rok_thread_shutdown();
}
```

각 worker thread 의 시작 및 종료 시점에 호출되어, [TLS 슬롯](803-tls-slots.md) 의 TLS 를 초기화하거나 정리합니다.

## 8.5.3. Coroutine 영역

### `rok_coro_frame_alloc`

[코루틴](../007-runtime/708-coroutine.md) 의 frame 을 할당합니다. **Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_coro_frame_alloc(size: usize, type_id: u32) -> Ptr<u8>;
}
```

`rok_gc_alloc_slow` 와 거의 동일하지만, frame 전용으로 별도 처리가 가능합니다.

### `rok_coro_park` / `rok_coro_unpark`

**Thread-safety: `rok_coro_park` 는 `caller-thread-only`** (호출자 코루틴 자신을 park), **`rok_coro_unpark` 는 `thread-safe`** (다른 thread 에서 호출 가능).

```
extern "C" {
    fn rok_coro_park(frame: Ptr<u8>, waker: Ptr<u8>);
    fn rok_coro_unpark(frame: Ptr<u8>);
}
```

`rok_coro_park` 는 코루틴을 일시 중단시키고 waker 를 등록합니다. `rok_coro_unpark` 는 코루틴을 재개시키며 task 큐에 push 합니다.

여기서 `waker` 는 깨우는 함수와 컨텍스트의 묶음, 곧 런타임의 `Waker` 객체 ([2.20.2](../../refs-standard-libraries/200-core/220-future.md)) 를 가리키는 **불투명 포인터 (`Ptr<u8>`)** 입니다. C ABI 경계에서는 `Waker` 의 내부 표현을 노출하지 않으므로, [7.8.8](../007-runtime/708-coroutine.md) 을 비롯한 모든 자리에서 `waker` 인자의 타입은 `Ptr<u8>` 로 통일됩니다 (`rok_io_register`, `rok_timer_register` 도 같음). 포인터가 가리키는 객체의 정확한 레이아웃은 코루틴 구현이 결정합니다.

## 8.5.4. Scheduler 영역

### `rok_task_spawn`

[스케줄러](../007-runtime/706-scheduler.md) 의 task spawn 에 해당합니다. **Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_task_spawn(coro_frame: Ptr<u8>) -> Ptr<u8>;   // TaskHandle
}
```

`coro_frame` 은 `async fn` 의 frame 이며, 반환값은 `await` 할 수 있는 객체인 task handle 입니다.

### `rok_task_await`

**Thread-safety: `caller-thread-only`** (호출자 코루틴이 직접 대기 / 일시 중단되는 흐름).

```
extern "C" {
    fn rok_task_await(handle: Ptr<u8>) -> Ptr<u8>;       // 결과 객체
}
```

task 의 결과를 기다립니다. 호출자도 코루틴이므로 결과가 나오지 않으면 호출자가 일시 중단됩니다.

### `rok_task_cancel`

**Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_task_cancel(handle: Ptr<u8>);
}
```

task 를 cancel 합니다. 진행 중이라면 중단시키며, `await` 중인 호출자에게 취소 신호를 전달합니다.

### `rok_thread_spawn`

1:1 OS thread 를 생성합니다. **Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_thread_spawn(closure: Ptr<u8>) -> Ptr<u8>;    // ThreadHandle
}
```

### `rok_thread_join`

**Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_thread_join(handle: Ptr<u8>) -> Ptr<u8>;
}
```

### `rok_par_for` / `rok_par_scope`

병렬 실행을 담당합니다 ([스케줄러](../007-runtime/706-scheduler.md) 참조). **Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_par_for(start: i64, end: i64, step: i64, callback: Ptr<u8>);
    fn rok_par_scope(callback: Ptr<u8>);
}
```

`rok_par_for` 는 `par::for_each` 의 desugar 이며, `rok_par_scope` 는 `par::scope` 의 desugar 입니다.

## 8.5.5. I/O / Timer 영역

### `rok_io_register`

**Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_io_register(fd: i32, interest: u32, waker: Ptr<u8>);
}
```

각 인자의 의미는 다음과 같습니다.

- `fd`: file descriptor 입니다.
- `interest`: 비트 플래그로, READ / WRITE / EDGE 등이 포함됩니다.
- `waker`: fd 가 ready 될 때 깨울 waker 입니다.

### `rok_timer_register`

**Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_timer_register(deadline: u64, waker: Ptr<u8>);
}
```

`deadline` 은 절대 시간으로, 밀리초나 마이크로초 등이며 정확한 단위는 구현이 결정합니다. `waker` 는 시간 도달 시 깨울 waker 입니다.

## 8.5.6. 에러 전달 (심볼 없음)

[에러 제어 흐름](../003-expressions/314-error-control-flow.md) 의 `throw` / `try` / `catch` 에는 런타임 심볼이 없습니다. throwing 함수는 에러를 반환값의 일부로 돌려주고 ([8.6.5](806-calling-convention.md#865-throwing-함수의-반환-값-기반-에러-전달)), 호출 지점이 그 값을 검사해 분기합니다 ([8.9.6](809-lowering-control.md#896-try--catch--finally-값-기반-분기)). 따라서 에러를 던지거나 현재 에러를 꺼내기 위해 런타임을 호출하는 일은 없으며, 에러 처리를 위한 TLS 슬롯이나 jmp_buf 도 두지 않습니다.

## 8.5.7. Diagnostic 영역

### `rok_panic`

**Thread-safety: `thread-safe`** (어떤 thread 에서든 호출 가능, 호출 즉시 프로세스 종료).

```
extern "C" {
    fn rok_panic(msg: ConstPtr<u8>, msg_len: usize);   // 반환 안 함
}
```

회복 불가능한 panic 으로, fatal 입니다. 이 함수는 반환하지 않으며, 메시지를 stderr 에 출력한 뒤 프로세스를 종료 코드 `101` 로 종료시킵니다. `catch` 나 `finally` 는 실행되지 않습니다.

### `rok_panic_str`

**Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_panic_str(msg: str) -> !;          // msg 는 (ptr, len)
}
```

`rok_panic` 과 같으며 메시지를 `str` 로 받습니다. core 의 `panic(msg: str)` 이 이 심볼을 부릅니다.

### `rok_oom`

**Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_oom() -> !;
}
```

`fatal error: out of memory` 를 stderr 에 출력하고 종료 코드 `101` 로 종료합니다. 런타임 안의 할당 실패와 core 의 `oom()` 이 모두 이 심볼을 거칩니다. 런타임은 rok 함수를 이름으로 부를 수 없으므로, 할당 실패 시 `oom_fn` 슬롯의 함수를 부르지 않고 이 심볼을 직접 씁니다.

## 8.5.8. 심볼 명명 규약

심볼의 접두사별 영역은 다음 표와 같이 정리됩니다.

| Prefix          | 영역           |
|-----------------|----------------|
| `rok_gc_*`      | GC 관련        |
| `rok_runtime_*` | 런타임 일반    |
| `rok_thread_*`  | OS thread 관련 |
| `rok_coro_*`    | 코루틴 관련    |
| `rok_task_*`    | task 관련      |
| `rok_par_*`     | 병렬 실행      |
| `rok_io_*`      | I/O            |
| `rok_timer_*`   | 타이머         |
| `rok_panic`     | 진단           |

이러한 접두사 규약을 통해 외부 라이브러리나 사용자 코드와의 심볼 충돌을 피하게 됩니다.

## 8.5.9. 함수 시그니처의 호환성

각 함수의 시그니처는 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 의 일부를 이룹니다. 변경 시의 정책은 다음과 같습니다.

- minor: 호환 가능한 변경만 허용되며, 인자 추가는 함수 wrapper 를 통해 처리합니다.
- major: 호환을 깨는 변경이 가능하며, 이때 ABI 버전 해시가 변환됩니다.

자세한 사항은 [개요](README.md) 의 안정성 정책을 참조합니다.

## 8.5.10. Calling convention

위 모든 함수의 calling convention 은 타깃 플랫폼의 C ABI 를 따릅니다. 예를 들어 System V x86_64, AAPCS aarch64, MS x64 Windows 등이 해당합니다.

## 8.5.11. 사용 예: TLAB fast path 의 slow 호출

```mlir
; LLVM IR (fast path 실패 시)
%cursor = load i8*, i8** @rok_tls_tlab_cursor
%new_cursor = getelementptr i8, i8* %cursor, i64 24
%limit = load i8*, i8** @rok_tls_tlab_limit
%check = icmp ugt i8* %new_cursor, %limit
br i1 %check, label %slow, label %fast

slow:
  %obj = call i8* @rok_gc_alloc_slow(i64 24, i64 8, i32 12345)
  br label %cont

fast:
  store i8* %new_cursor, i8** @rok_tls_tlab_cursor
  br label %cont

cont:
  %obj_phi = phi i8* [%obj_slow, %slow], [%cursor, %fast]
  ; ... 계속
```

위 예시는 fast path 와 slow path 가 결합된 LLVM IR 구조를 보여줍니다.

## 8.5.12. 부트스트랩 / 테스트 보조 심볼 (부록, 비규범)

> [!IMPORTANT]
> 이 절은 **비규범 (non-normative)** 입니다. 아래 심볼은 표준 라이브러리의 `Display` / f-string 출력, `rok::bare::alloc`, `str` 표면이 완성되기 전까지 컴파일러 부트스트랩과 테스트 스위트를 지탱하기 위해 참조 런타임 (`rokr`) 이 내보내는 보조 심볼입니다. ABI 본 표 (8.5.1 ~ 8.5.11) 의 일부가 아니며, 안정성 약속도 없습니다. 다른 구현은 이 심볼을 제공하지 않아도 사양을 준수한 것으로 봅니다.

시그니처는 본문과 같은 표기 (`Ptr<T>` = 가변 raw 포인터, `ConstPtr<T>` = 읽기 전용 raw 포인터) 를 따릅니다. 문자열 인자는 NUL 종료 C 문자열이며, `bool` 은 C ABI 에 안정된 1 비트 표현이 없으므로 `i32` (0 = false, 그 외 = true) 로 전달합니다. 출력 계열 함수는 식 안에서 조합되도록 항상 `0` 을 반환합니다.

```
extern "C" {
    // 출력 — 값 + 개행
    fn rok_print_i32(n: i32) -> i32;
    fn rok_print_i64(n: i64) -> i32;
    fn rok_print_f64(d: f64) -> i32;                       // printf "%g"
    fn rok_print_str(s: ConstPtr<u8>) -> i32;
    fn rok_print_bool(b: i32) -> i32;                      // "true" / "false", 개행 없음

    // 출력 — 개행 없음
    fn rok_write_str(s: ConstPtr<u8>) -> i32;
    fn rok_write_i64(n: i64) -> i32;
    fn rok_write_f64(d: f64) -> i32;
    fn rok_write_hex(n: i64) -> i32;                       // "0x%llx"

    // 메모리
    fn rok_alloc(size: usize, align: usize) -> Ptr<u8>;    // type_id 없는 할당
    fn rok_free(ptr: Ptr<u8>);                             // null 무시
    fn rok_load_i32(ptr: ConstPtr<i32>) -> i32;
    fn rok_load_i64(ptr: ConstPtr<i64>) -> i64;
    fn rok_load_f64(ptr: ConstPtr<f64>) -> f64;
    fn rok_store_i32(ptr: Ptr<i32>, val: i32);
    fn rok_store_i64(ptr: Ptr<i64>, val: i64);
    fn rok_store_f64(ptr: Ptr<f64>, val: f64);

    // C 문자열
    fn rok_str_len(s: ConstPtr<u8>) -> usize;              // null → 0
    fn rok_str_eq(a: ConstPtr<u8>, b: ConstPtr<u8>) -> i32; // 같으면 1, 아니면 0

    // 테스트
    fn rok_assert(cond: i32) -> i32;                       // 0 이면 "assertion failed" 출력 후 종료 코드 101
}
```

다음 규칙이 적용됩니다.

- 컴파일러는 부트스트랩 모드에서 사용자 파일마다 위 심볼의 `extern "C"` 선언을 합성할 수 있으며, 이 경우 사용자 코드는 별도의 선언 없이 호출할 수 있습니다. 이 합성은 구현 편의이며 언어 의미의 일부가 아닙니다.
- 사용자 코드와 표준 라이브러리는 이 심볼에 의존해서는 안 됩니다. 시그니처는 예고 없이 바뀌거나 제거될 수 있습니다.
- `rok_` prefix 를 공유하지만 8.5.8 의 영역 표에는 포함되지 않습니다. 이름 충돌을 피하기 위해 사양의 규범 심볼은 위 이름을 재사용하지 않습니다.
- `rok_panic` 은 이 부록에 속하지 않습니다. 시그니처는 [8.5.7](#857-diagnostic-영역) 의 `rok_panic(msg: ConstPtr<u8>, msg_len: usize)` 가 권위이며, 구현이 이와 다른 시그니처를 내보내는 것은 부적합 (non-conformance) 입니다.
