# 7. 런타임 개요

본 블록은 rok 런타임 시스템 ([GC](../000-meta/007-glossary.md#gc-garbage-collector), 작업 스케줄러, 코루틴) 의 구현 사양을 정의합니다. 이 시스템은 [실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md) 의 두 엔진 ([JIT](../000-meta/007-glossary.md#jit-just-in-time) / [AOT](../000-meta/007-glossary.md#aot-ahead-of-time)) 양쪽에서 동일하게 작동합니다.

## 목차

본 장의 세부 절은 다음과 같이 구성됩니다.

- [7.1. 객체 레이아웃과 메모리 공간](701-object-layout.md)
- [7.2. 할당](702-allocation.md)
- [7.3. GC 알고리듬](703-gc-algorithm.md)
- [7.4. 배리어와 루트 추적](704-barriers-and-roots.md)
- [7.5. 동시 mark](705-concurrent-collection.md)
- [7.6. 스케줄러](706-scheduler.md)
- [7.7. I/O 드라이버와 타이머](707-io-and-timer.md)
- [7.8. 코루틴](708-coroutine.md)

## 핵심 결정

런타임의 핵심 구성 요소에 대한 결정은 다음과 같이 요약됩니다.

```
GC          자체 Immix + Sticky Mark-Bit 세대
Scheduler   M:N work-stealing
Coroutine   컴파일러가 만든 상태 기계 (stackless)
```

세 부분 모두 자체 구현이며, 외부 라이브러리에 대한 의존은 0 입니다.

## 의존성 0 원칙

런타임은 외부 라이브러리에 의존하지 않습니다. 구체적으로 다음과 같은 외부 구성 요소를 사용하지 않습니다.

- 외부 GC 라이브러리 (Boehm GC, MMTk 등) 를 사용하지 않습니다.
- 외부 비동기 런타임을 사용하지 않습니다.
- 외부 코루틴 라이브러리를 사용하지 않습니다.

다만 플랫폼 시스템 인터페이스 (`libc`, OS 헤더 등) 는 의존성으로 치지 않습니다. 이는 OS 인터페이스의 필수 부분이기 때문입니다.

## M:N 모델

스케줄러는 M:N 하이브리드 방식을 따릅니다.

```
OS Thread (worker) x N  ──── rok task / coroutine x M
                                   (typically M >> N)
```

이 모델의 운용 방식은 다음과 같습니다. 기본 N 은 CPU 코어 수이며, 런타임 설정으로 조정할 수 있습니다. 또한 한 OS thread 가 여러 역할을 동시에 수행하는데, 그 역할은 Scheduler Worker, Mutator, I/O Driver, GC Worker 입니다. 다만 I/O driver 는 현재 별도의 reactor thread 하나가 맡습니다 ([7.7](707-io-and-timer.md)).

이러한 통합으로 추가 thread 분리 비용 없이 모든 런타임 활동이 한 스레드 풀에서 처리됩니다.

## 단일 .a 산출물

런타임은 단일 staticlib `librokr.a` 한 개로 배포됩니다. 사용자 패키지를 빌드하면 `librokr.a` 가 링크되어 추가 외부 의존 없이 실행 가능한 바이너리가 나옵니다.

## 런타임 오류: 스택 오버플로

스레드의 스택이 넘치면 런타임은 stdout 을 (가능하면) flush 하고 stderr 에 다음을 출력한 뒤 abort (SIGABRT) 합니다. panic 과 달리 unwind 나 `finally` 실행은 없습니다.

```
thread '<name>' has overflowed its stack
```

`<name>` 은 그 스레드의 이름입니다. `thread::Builder::name` ([stdlib 3.4](../../refs-standard-libraries/300-std/304-thread.md)) 으로 준 이름이며, 그 이름은 OS 스레드 이름으로도 설정됩니다. main 스레드는 `main`, 이름이 없는 스레드는 `<unnamed>` 입니다. main 스레드, `thread::spawn` 스레드, executor worker (깊은 `await` 사슬 포함) 가 모두 대상입니다. 검출 방식 (guard 범위와 `sigaltstack`) 과 TLS 슬롯은 [8.5.7](../008-abi/805-extern-symbols.md#스택-오버플로) 에 있습니다.

## 컴파일러와의 인터페이스

런타임은 컴파일러가 생성한 코드와 정해진 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 로 통신합니다. 인터페이스는 다음 세 가지 요소로 구성됩니다.

- 객체 레이아웃 (16 바이트 헤더 + payload) 입니다.
- `extern "C"` 함수로는 `rok_gc_alloc_slow`, `rok_runtime_yield`, `rok_thread_spawn` 등이 있습니다.
- 코루틴은 컴파일러가 만든 상태 기계이며 `llvm.coro.*` 를 쓰지 않습니다 ([7.8](708-coroutine.md)). GC 루트는 intrinsic 없이 컴파일러가 shadow stack frame 을 직접 emit 합니다 ([7.4.1](704-barriers-and-roots.md#741-루트-추적-컴파일러가-직접-emit-하는-shadow-stack)).

이 ABI 는 [8. 런타임 ABI / MLIR lowering](../008-abi/README.md) 에서 자세히 정의되며, 본 7 장은 내부 동작만을 다룹니다.
