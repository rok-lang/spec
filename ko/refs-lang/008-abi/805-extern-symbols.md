# 8.5. Extern "C" 심볼

`libnokr.a` 가 외부에 노출하는 모든 [extern "C" 심볼](../000-meta/007-glossary.md#extern-"c"-심볼) 의 시그니처를 정의한다. 컴파일러
생성 코드가 이 심볼들을 호출한다.

> [!IMPORTANT]
> 모든 심볼은 `nok_` prefix 로 시작한다. 외부 라이브러리 / 사용자 코드와의 심볼 충돌
> 회피.

## 8.5.1. GC 영역

### `nok_gc_alloc_slow`

[할당](../007-runtime/702-allocation.md) 의 [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) slow path.

```
extern "C" {
    fn nok_gc_alloc_slow(size: usize, align: usize, type_id: u32) -> Ptr<u8>;
}
```

- `size`: 객체의 총 크기 (16 + payload, 정렬 후).
- `align`: 객체의 정렬 요구 (보통 8, `#[repr(align(N))]` 시 더 큼).
- `type_id`: 객체의 타입 ID.
- 반환: 헤더 자리부터 시작하는 포인터. type_id, gc_flags 는 함수가 채워둔다.

null 반환은 out of memory (drastic). 실제로는 fatal error 던짐 (구현 결정).

### `nok_gc_wb_slow`

[배리어와 루트 추적](../007-runtime/704-barriers-and-roots.md) 의 write barrier slow path.

```
extern "C" {
    fn nok_gc_wb_slow(obj: Ptr<u8>);
}
```

- `obj`: 변경된 슬롯이 속한 객체의 포인터.
- 반환: 없음.
- 효과: remembered set 에 obj 추가, log 가 차면 flush.

### `nok_gc_register_type`

[ScanTable](802-scan-table.md) 의 타입 등록.

```
extern "C" {
    fn nok_gc_register_type(type_id: u32, scan_table: ConstPtr<u8>);
}
```

- 호출 시점: 프로세스 시작.

### `nok_gc_pin` / `nok_gc_unpin`

[FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 객체 pinning ([객체 레이아웃과 메모리 공간](../007-runtime/701-object-layout.md) / [`extern` 블록 (FFI 선언)](../005-modules/509-extern-block.md)).

```
extern "C" {
    fn nok_gc_pin(obj: Ptr<u8>) -> Ptr<u8>;       // 새 위치 (Non-Moving 공간) 반환
    fn nok_gc_unpin(obj: Ptr<u8>);
}
```

- pin: 객체를 Non-Moving 공간으로 옮긴다. 외부 코드가 들고 있는 동안 안전.
- unpin: 다시 일반 [Immix](../000-meta/007-glossary.md#immix) 공간으로 돌릴 수 있다 (또는 그대로).

### `nok_gc_collect`

명시적 [GC](../000-meta/007-glossary.md#gc-garbage-collector) 트리거 (드물게).

```
extern "C" {
    fn nok_gc_collect();
}
```

- 즉시 GC 사이클 시작. 동시 수집이라 호출자도 mutator 로 계속 작동한다.

## 8.5.2. Runtime 영역

### `nok_runtime_init`

런타임 초기화.

```
extern "C" {
    fn nok_runtime_init();
}
```

- 호출 시점: 프로세스 시작 직후, `main()` 전.
- 효과: worker 풀 생성, GC 공간 mmap, I/O driver 초기화.

### `nok_runtime_shutdown`

런타임 종료.

```
extern "C" {
    fn nok_runtime_shutdown();
}
```

- 호출 시점: `main()` 끝난 후.
- 효과: worker 풀 종료, 미해제 자원 정리.

### `nok_runtime_yield`

[Safepoint 프로토콜](804-safepoint.md) 의 `yield` 함수.

```
extern "C" {
    fn nok_runtime_yield();
}
```

### `nok_thread_init` / `nok_thread_shutdown`

worker thread 의 lifecycle.

```
extern "C" {
    fn nok_thread_init();
    fn nok_thread_shutdown();
}
```

각 worker thread 시작 / 종료 시 호출. [TLS 슬롯](803-tls-slots.md) 의 TLS 초기화 / 정리.

## 8.5.3. Coroutine 영역

### `nok_coro_frame_alloc`

[코루틴](../007-runtime/708-coroutine.md) 의 frame 할당.

```
extern "C" {
    fn nok_coro_frame_alloc(size: usize, type_id: u32) -> Ptr<u8>;
}
```

- `nok_gc_alloc_slow` 와 거의 동일하지만, frame 전용으로 별도 처리 가능.

### `nok_coro_park` / `nok_coro_unpark`

```
extern "C" {
    fn nok_coro_park(frame: Ptr<u8>, waker: Ptr<u8>);
    fn nok_coro_unpark(frame: Ptr<u8>);
}
```

- `nok_coro_park`: 코루틴 일시 중단, waker 등록.
- `nok_coro_unpark`: 코루틴 재개. task 큐에 push.

`waker` 는 깨우는 함수와 컨텍스트의 묶음. 정확한 표현은 코루틴 구현의 결정.

## 8.5.4. Scheduler 영역

### `nok_task_spawn`

[스케줄러](../007-runtime/706-scheduler.md) 의 task spawn.

```
extern "C" {
    fn nok_task_spawn(coro_frame: Ptr<u8>) -> Ptr<u8>;   // TaskHandle
}
```

- `coro_frame`: `async fn` 의 frame.
- 반환: task handle (`await` 할 수 있는 객체).

### `nok_task_await`

```
extern "C" {
    fn nok_task_await(handle: Ptr<u8>) -> Ptr<u8>;       // 결과 객체
}
```

- task 결과를 기다린다.
- 호출자도 코루틴이라 결과가 안 나오면 호출자가 일시 중단된다.

### `nok_task_cancel`

```
extern "C" {
    fn nok_task_cancel(handle: Ptr<u8>);
}
```

- task 를 cancel. 진행 중이면 중단, `await` 중인 호출자에게 취소 신호.

### `nok_thread_spawn`

1:1 OS thread.

```
extern "C" {
    fn nok_thread_spawn(closure: Ptr<u8>) -> Ptr<u8>;    // ThreadHandle
}
```

### `nok_thread_join`

```
extern "C" {
    fn nok_thread_join(handle: Ptr<u8>) -> Ptr<u8>;
}
```

### `nok_par_for` / `nok_par_scope`

병렬 실행 ([스케줄러](../007-runtime/706-scheduler.md)).

```
extern "C" {
    fn nok_par_for(start: i64, end: i64, step: i64, callback: Ptr<u8>);
    fn nok_par_scope(callback: Ptr<u8>);
}
```

- `nok_par_for`: `par::for_each` 의 desugar.
- `nok_par_scope`: `par::scope` 의 desugar.

## 8.5.5. I/O / Timer 영역

### `nok_io_register`

```
extern "C" {
    fn nok_io_register(fd: i32, interest: u32, waker: Ptr<u8>);
}
```

- `fd`: file descriptor.
- `interest`: 비트 플래그 (READ / WRITE / EDGE 등).
- `waker`: fd ready 시 깨울 waker.

### `nok_timer_register`

```
extern "C" {
    fn nok_timer_register(deadline: u64, waker: Ptr<u8>);
}
```

- `deadline`: 절대 시간 (밀리초 / 마이크로초 등. 정확한 단위는 구현 결정).
- `waker`: 시간 도달 시 깨울 waker.

## 8.5.6. 예외 영역

### `nok_throw`

[에러 제어 흐름](../003-expressions/314-error-control-flow.md) 의 예외 발생을 런타임에 구현하는 심볼.

```
extern "C" {
    fn nok_throw(exc: Ptr<u8>);   // 반환 안 함 — longjmp
}
```

- `exc`: 던져진 예외 객체.
- 반환 안 함. 가장 가까운 jmp_buf 로 longjmp.
- 호출자는 `llvm.unreachable` 또는 동등한 표시로 control flow 종료.

### `nok_exception_current`

```
extern "C" {
    fn nok_exception_current() -> Ptr<u8>;
}
```

- `catch` 블록 안에서 현재 던져진 예외 값을 가져온다.

## 8.5.7. Diagnostic 영역

### `nok_panic`

```
extern "C" {
    fn nok_panic(msg: ConstPtr<u8>, msg_len: usize);   // 반환 안 함
}
```

- 회복 불가능한 panic. fatal.
- 반환 안 함. 메시지를 stderr 에 출력하고 프로세스 종료.

## 8.5.8. 심볼 명명 규약

| Prefix                          | 영역           |
| ------------------------------- | -------------- |
| `nok_gc_*`                      | GC 관련        |
| `nok_runtime_*`                 | 런타임 일반    |
| `nok_thread_*`                  | OS thread 관련 |
| `nok_coro_*`                    | 코루틴 관련    |
| `nok_task_*`                    | task 관련      |
| `nok_par_*`                     | 병렬 실행      |
| `nok_io_*`                      | I/O            |
| `nok_timer_*`                   | 타이머         |
| `nok_throw` / `nok_exception_*` | 예외           |
| `nok_panic`                     | 진단           |

이 prefix 규약으로 외부 라이브러리 / 사용자 코드와의 심볼 충돌을 피한다.

## 8.5.9. 함수 시그니처의 호환성

각 함수의 시그니처는 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 의 일부. 변경 시:

- minor: 호환 변경만 가능 (인자 추가는 함수 wrapper 로).
- major: 호환을 깨는 변경 가능, ABI 버전 해시가 바뀐다.

[개요](README.md) 의 안정성 정책 참조.

## 8.5.10. Calling convention

위 모든 함수의 calling convention 은 타깃 플랫폼의 C ABI (예: System V x86_64, AAPCS
aarch64, MS x64 Windows).

## 8.5.11. 사용 예: TLAB fast path 의 slow 호출

```mlir
; LLVM IR (fast path 실패 시)
%cursor = load i8*, i8** @nok_tls_tlab_cursor
%new_cursor = getelementptr i8, i8* %cursor, i64 24
%limit = load i8*, i8** @nok_tls_tlab_limit
%check = icmp ugt i8* %new_cursor, %limit
br i1 %check, label %slow, label %fast

slow:
  %obj = call i8* @nok_gc_alloc_slow(i64 24, i64 8, i32 12345)
  br label %cont

fast:
  store i8* %new_cursor, i8** @nok_tls_tlab_cursor
  br label %cont

cont:
  %obj_phi = phi i8* [%obj_slow, %slow], [%cursor, %fast]
  ; ... 계속
```

위 예가 fast path + slow path 의 LLVM IR 구조다.
