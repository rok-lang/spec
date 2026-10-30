# 7. 런타임 개요

이 장은 Rok 런타임 시스템의 구현 사양을 정의합니다. 런타임은 [GC](../000-meta/007-glossary.md#gc-garbage-collector), 작업 스케줄러, 코루틴으로 이루어집니다. 런타임은 [실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md)에 나오는 두 엔진([JIT](../000-meta/007-glossary.md#jit-just-in-time) / [AOT](../000-meta/007-glossary.md#aot-ahead-of-time))에서 똑같이 동작합니다.

## 목차

이 장은 다음 절로 이루어집니다.

- [7.1. 객체 레이아웃과 메모리 공간](701-object-layout.md)
- [7.2. 할당](702-allocation.md)
- [7.3. GC 알고리듬](703-gc-algorithm.md)
- [7.4. 배리어와 루트 추적](704-barriers-and-roots.md)
- [7.5. 동시 mark](705-concurrent-collection.md)
- [7.6. 스케줄러](706-scheduler.md)
- [7.7. I/O 드라이버와 타이머](707-io-and-timer.md)
- [7.8. 코루틴](708-coroutine.md)

## 핵심 결정

런타임 주요 구성 요소의 결정을 요약하면 다음과 같습니다.

```
GC          자체 Immix + Sticky Mark-Bit 세대
Scheduler   M:N work-stealing
Coroutine   컴파일러가 만든 상태 기계 (stackless)
```

세 부분 모두 직접 구현하고, 외부 라이브러리에 하나도 의존하지 않습니다.

## 의존성 0 원칙

런타임은 외부 라이브러리에 의존하지 않습니다. 구체적으로 다음을 쓰지 않습니다.

- 외부 GC 라이브러리(Boehm GC, MMTk 등)
- 외부 비동기 런타임
- 외부 코루틴 라이브러리

플랫폼 시스템 인터페이스(`libc`, OS 헤더 등)는 OS 인터페이스의 필수 부분이므로 의존성으로 치지 않습니다.

## M:N 모델

스케줄러는 M:N 하이브리드 방식을 따릅니다.

```
OS Thread (worker) x N  ──── rok task / coroutine x M
                                   (typically M >> N)
```

N의 기본값은 CPU 코어 수이고, 런타임 설정으로 바꿀 수 있습니다. OS 스레드 하나가 Scheduler Worker, Mutator, I/O Driver, GC Worker 역할을 함께 맡습니다. 다만 지금은 별도의 reactor 스레드 하나가 I/O 드라이버를 맡습니다([7.7](707-io-and-timer.md)).

역할을 이렇게 합쳐 두었으므로 런타임은 스레드를 따로 나누는 비용 없이 모든 일을 한 스레드 풀에서 처리합니다.

## 단일 .a 산출물

런타임은 정적 라이브러리 `librokr.a` 하나로 배포됩니다. 사용자 패키지를 빌드하면 `librokr.a`를 링크합니다. 그 결과 다른 외부 의존성 없이 실행할 수 있는 바이너리가 나옵니다.

## 런타임 에러: 스택 오버플로

스레드의 스택이 넘치면 런타임은 할 수 있는 경우 stdout을 비웁니다. 그리고 stderr에 다음을 출력하고 프로세스를 중단(SIGABRT)합니다. panic과 달리 스택을 되감지 않고, `finally`도 실행하지 않습니다.

```
thread '<name>' has overflowed its stack
```

`<name>`은 그 스레드의 이름입니다. `thread::Builder::name`([stdlib 3.4](../../refs-standard-libraries/300-std/304-thread.md))으로 준 이름이고, OS 스레드 이름으로도 설정됩니다. 메인 스레드는 `main`, 이름이 없는 스레드는 `<unnamed>`입니다. 메인 스레드, `thread::spawn` 스레드, executor 워커가 모두 대상입니다. 깊은 `await` 사슬도 포함합니다. 검출 방식(guard 범위와 `sigaltstack`)과 TLS 슬롯은 [8.5.7](../008-abi/805-extern-symbols.md#스택-오버플로)에 있습니다. wasm에서는 모듈이 trap을 잡을 수 없고 호스트가 보고합니다. 함께 배포하는 JS 호스트는 위 메시지를 출력합니다.

## 컴파일러 인터페이스

런타임은 컴파일러가 만든 코드와 정해진 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface)로 통신합니다. 인터페이스는 다음 세 요소로 이루어집니다.

- 객체 레이아웃(16바이트 헤더 + 페이로드)
- `rok_gc_alloc_slow`, `rok_runtime_yield`, `rok_thread_spawn` 같은 `extern "C"` 함수
- 코루틴은 컴파일러가 만든 상태 기계이고, `llvm.coro.*` 계열은 쓰지 않습니다([7.8](708-coroutine.md)). GC 루트는 컴파일러가 intrinsic 없이 shadow stack 프레임으로 직접 내보냅니다([7.4.1](704-barriers-and-roots.md#741-루트-추적-컴파일러가-직접-내보내는-shadow-stack)).

이 ABI는 [8. 런타임 ABI / MLIR lowering](../008-abi/README.md)에서 자세히 정의합니다. 7장은 내부 동작만 다룹니다.
