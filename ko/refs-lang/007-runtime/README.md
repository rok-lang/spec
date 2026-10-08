# 7. 런타임 개요

본 블록은 nok 런타임 시스템 ([GC](../000-meta/007-glossary.md#gc-garbage-collector), 작업 스케줄러, 코루틴) 의 구현 사양을 정의한다. 이 시스템은 [실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md) 의 두 엔진 ([JIT](../000-meta/007-glossary.md#jit-just-in-time) / [AOT](../000-meta/007-glossary.md#aot-ahead-of-time)) 양쪽에서 동일하게 작동한다.

## 목차

- [7.1. 객체 레이아웃과 메모리 공간](701-object-layout.md)
- [7.2. 할당](702-allocation.md)
- [7.3. GC 알고리듬](703-gc-algorithm.md)
- [7.4. 배리어와 루트 추적](704-barriers-and-roots.md)
- [7.5. 동시 수집](705-concurrent-collection.md)
- [7.6. 스케줄러](706-scheduler.md)
- [7.7. I/O 드라이버와 타이머](707-io-and-timer.md)
- [7.8. 코루틴](708-coroutine.md)

## 핵심 결정

```
GC          자체 Immix + Sticky Mark-Bit 세대
Scheduler   M:N work-stealing
Coroutine   LLVM coroutine intrinsic (stackless)
```

세 부분 모두 자체 구현이다. 외부 라이브러리 의존 0.

## 의존성 0 원칙

런타임은 외부 라이브러리에 의존하지 않는다.

- 외부 GC 라이브러리 (Boehm GC, MMTk 등) 사용 안 함.
- 외부 비동기 런타임 사용 안 함.
- 외부 코루틴 라이브러리 사용 안 함.

플랫폼 시스템 인터페이스 (`libc`, OS 헤더 등) 는 의존성으로 치지 않는다. OS 인터페이스의 필수 부분이다.

## M:N 모델

스케줄러는 M:N 하이브리드.

```
OS Thread (worker) N 개  ──── nok task / coroutine M 개
                                   (M >> N 일반)
```

- 기본 N = CPU 코어 수, 런타임 설정으로 조정 가능.
- 한 OS thread 가 여러 역할을 동시에 수행한다. Scheduler Worker, Mutator, I/O Driver, GC Worker.

이 통합으로 추가 thread 분리 비용 없이 모든 런타임 활동이 한 스레드 풀에서 처리된다.

## 단일 .a 산출물

런타임은 단일 staticlib `libnokr.a` 한 개로 배포된다. 사용자 패키지를 빌드하면 `libnokr.a` 가 링크되어 추가 외부 의존 없이 실행 가능한 바이너리가 나온다.

## 컴파일러와의 인터페이스

런타임은 컴파일러가 생성한 코드와 정해진 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 로 통신한다.

- 객체 레이아웃 (16 바이트 헤더 + payload).
- `extern "C"` 함수: `nok_gc_alloc_slow`, `nok_runtime_yield`, `nok_task_spawn` 등.
- LLVM intrinsic: `llvm.gcroot`, `llvm.coro.*`.

이 ABI 는 [8. 런타임 ABI / MLIR lowering](../008-abi/README.md) 에서 자세히 정의한다. 본 7 장은 내부 동작만 다룬다.
