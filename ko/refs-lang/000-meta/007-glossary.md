# 0.7. 용어집

본 사양에서 자주 쓰는 용어의 정의. 항목은 가나다, 알파벳 순.

## ABI (Application Binary Interface)

컴파일된 코드가 링크와 실행 시점에 따라야 하는 바이너리 수준 계약. 객체 레이아웃, 호출
규약, 심볼 명명 등을 포함한다. nok 의 ABI 는 [8장 — ABI / MLIR lowering](../008-abi/README.md)
에서 정의한다.

## AOT (Ahead-of-Time)

소스를 미리 컴파일하여 정적 산출물(`.a` 또는 실행 파일)을 생성하는 모드. JIT 와 대조된다.
nok 은 JIT 와 AOT 두 엔진을 지원하며, 둘 다 같은 파이프라인을 공유한다.

## archive

컴파일된 패키지의 산출물. 표준 ar 형식의 `.a` 파일이다. nok 의 obj, NHIR bytecode, 메타가
하나의 archive 에 들어간다.

## block (블록)

`{ ... }` 로 둘러싸인 문장과 식의 묶음. 자체로 표현식이며 마지막 식이 블록의 값이 된다.

## block (Immix)

GC 의 메모리 단위. 32 KB 의 OS 메모리 페이지다. Immix 알고리듬에서 line 으로 분할되어
mark / sweep 단위가 된다. 위 블록(코드)과는 다른 개념.

## concrete fn / generic fn

호출 지점의 타입이 모두 구체인 함수가 concrete fn. 타입 매개변수가 남아 있는 함수가 generic
fn. AOT 에서 generic fn 은 호출 지점에서 specialize 되어 concrete fn 이 된다.

## dispatch (디스패치)

이름, 연산자, 메서드 호출이 어떤 구현으로 연결되는지의 결정. nok 은 정적 디스패치만 사용하며,
트레잇-as-타입 금지로 vtable 간접 호출이 없다.

## extern "C" 심볼

C ABI 로 노출되거나 호출되는 심볼. 컴파일러와 런타임의 인터페이스다.

## FFI (Foreign Function Interface)

다른 언어(주로 C)의 함수를 호출하거나, 다른 언어로부터 호출되는 인터페이스. `extern "C"`
블록으로 정의한다.

## GC (Garbage Collector)

자동 메모리 관리. nok 의 GC 는 Immix mark-region 과 Sticky Mark-Bit generational 의 결합이다.

## generic / 제네릭

타입을 매개변수로 받는 함수 또는 타입. 호출과 인스턴스화 지점에서 단형화된다.

## HIR (High-level IR) / MIR (Mid-level IR)

컴파일 파이프라인의 중간 표현. 소스 → AST → HIR → MIR → 표준 MLIR.

## Immix

GC 알고리듬 (Blackburn & McKinley, 2008). 32 KB block 과 128 B line 단위의 mark-region 이며,
opportunistic evacuation 으로 단편화를 처리한다.

## JIT (Just-in-Time)

코드를 런타임에 즉시 컴파일하는 모드. REPL 과 Jupyter 가 사용한다. AOT 와 대조된다.

## lang slot / 매직

언어가 라이브러리의 특정 항목을 식별하는 메커니즘. `#[lang(slot_name)]` 로 stdlib 항목을
컴파일러 슬롯에 연결한다.

## LOS (Large Object Space)

GC 의 큰 객체 전용 공간. 임계(8 KB / 2 KB) 이상의 객체는 LOS 에 이동 없이 둔다.

## monomorphization (단형화)

generic 함수를 호출 지점의 구체 타입별로 별도 컴파일하는 과정. 간접 호출 비용이 0 이며,
컴파일 시간과 바이너리 크기를 비용으로 치른다.

## NHIR (nok HIR bytecode)

nok HIR 을 직렬화한 형식. archive 의 `.nok.hir` 섹션에 포함되며, 다운스트림 패키지가
generic fn 을 specialize 할 수 있게 한다.

## niche optimization

합 타입이나 enum 의 사용하지 않는 비트 패턴을 활용하여 태그 자리를 없애는 최적화. 대표 예는
`Ptr | none` 의 `none = null pointer`.

## scan_table

GC 가 객체의 어떤 슬롯이 GC 포인터인지 알아내기 위한 메타 테이블. `type_id → scan_table`
로 매핑되며, 컴파일러가 빌드 시 등록한다.

## shadow stack

GC 가 살아 있는 객체를 추적하기 위한 보조 스택. 함수 진입과 종료 시 GC 객체 변수를 push,
pop 한다. 일반 stack 위와 별도로 thread-local 영역에 둔다.

## specialize

generic 함수의 단형화. 구체 타입 튜플로 컴파일 결과를 생성하는 작업.

## sticky mark-bit

generational GC 의 세대 추적 비트. 처음 mark 되면 켜져서 "오래된 객체" 가 된다. young GC 는
sticky 객체를 건너뛰어 비용을 줄인다.

## sum type / 합 타입

`A | B` 형태로 둘 중 하나를 담는 타입. tagged union 의 한 형태다. 닫힌 슬롯이며, 모든 변형이
명시되어 패턴 매칭의 완전성 검사가 가능하다.

## TLAB (Thread-Local Allocation Buffer)

각 worker thread 가 가진 thread-local 메모리 버퍼. fast-path 할당이 bump pointer 만으로
이루어지며 동기화 비용이 0 이다.

## work-stealing

스케줄러의 load balancing 메커니즘. 각 worker 가 지역 deque 를 가지며, 일이 떨어진 worker
가 다른 worker 의 deque 끝에서 task 를 훔친다.
