# 0.7. 용어집

본 사양을 읽다 보면 같은 개념이 여러 장에 걸쳐 반복적으로 등장하기 때문에, 자주 쓰이는 용어를 한 곳에 모아 두는 것이 독자에게 도움이 됩니다. 이 문서는 rok 사양 전반에서 자주 등장하는 핵심 용어를 가나다순과 알파벳순으로 정리하여, 본문을 읽다가 개념이 모호해질 때 곧바로 참조할 수 있도록 마련한 항목집입니다. 각 항목은 짧은 정의에 더해 rok 의 구체적인 맥락을 함께 담고 있어, 단순한 사전적 풀이를 넘어 사양 내에서 그 용어가 어떤 위치를 차지하는지 짚을 수 있도록 구성하였습니다.

## ABI (Application Binary Interface)

ABI 는 컴파일된 코드가 링크와 실행 시점에 따라야 하는 바이너리 수준의 계약을 가리킵니다. 객체 레이아웃, 호출 규약, 심볼 명명 등이 여기에 포함됩니다. rok 의 ABI 는 [8장 — ABI / MLIR lowering](../008-abi/README.md) 에서 정의합니다.

## AOT (Ahead-of-Time)

AOT 는 소스를 미리 컴파일하여 정적 산출물(`.a` 또는 실행 파일) 을 생성하는 모드를 가리키며, JIT 와 대조됩니다. rok 은 JIT 와 AOT 두 엔진을 모두 지원하며, 두 엔진은 같은 파이프라인을 공유합니다.

## archive

archive 는 컴파일된 패키지의 산출물로서, 표준 ar 형식의 `.a` 파일입니다. concrete 코드의 obj 를 담으며, 라이브러리의 인터페이스 (RHIR) 는 그 옆의 `<name>.rhir` 파일입니다.

## block (블록)

블록은 `{ ... }` 로 둘러싸인 문장과 식의 묶음을 가리킵니다. 그 자체로 하나의 표현식이며, 마지막 식이 블록의 값이 됩니다.

## block (Immix)

Immix 의 block 은 GC 의 메모리 단위로서, 32 KB 의 OS 메모리 페이지에 해당합니다. Immix 알고리듬에서 line 으로 분할되어 mark / sweep 의 단위가 됩니다. 위에서 설명한 코드 블록과는 다른 개념입니다.

## concrete fn / generic fn

호출 지점의 타입이 모두 구체인 함수가 concrete fn 이며, 타입 매개변수가 남아 있는 함수가 generic fn 입니다. AOT 에서 generic fn 은 호출 지점에서 specialize 되어 concrete fn 이 됩니다.

## dispatch (디스패치)

디스패치란 이름, 연산자, 메서드 호출이 어떤 구현으로 연결되는지를 결정하는 과정을 뜻합니다. rok 은 정적 디스패치만 사용하며, 트레잇-as-타입 금지로 인해 vtable 간접 호출이 존재하지 않습니다.

## extern "C" 심볼

extern "C" 심볼은 C ABI 로 노출되거나 호출되는 심볼이며, 컴파일러와 런타임의 인터페이스 역할을 담당합니다.

## FFI (Foreign Function Interface)

FFI 는 다른 언어(주로 C) 의 함수를 호출하거나, 다른 언어로부터 호출되는 인터페이스를 가리킵니다. `extern "C"` 블록으로 정의합니다.

## GC (Garbage Collector)

GC 는 자동 메모리 관리 메커니즘입니다. rok 의 GC 는 Immix mark-region 과 Sticky Mark-Bit generational 을 결합한 형태로 구성됩니다.

## generic / 제네릭

제네릭은 타입을 매개변수로 받는 함수 또는 타입을 가리킵니다. 호출과 인스턴스화 지점에서 단형화됩니다.

## HIR (High-level IR) / MIR (Mid-level IR)

HIR 과 MIR 은 컴파일 파이프라인의 중간 표현입니다. 전체 흐름은 소스 → AST → HIR → MIR → 표준 MLIR 로 이어집니다.

## Immix

Immix 는 Blackburn 과 McKinley 가 2008 년에 발표한 GC 알고리듬입니다. 32 KB block 과 128 B line 단위의 mark-region 을 사용하며, opportunistic evacuation 으로 단편화를 처리합니다.

## JIT (Just-in-Time)

JIT 는 코드를 런타임에 즉시 컴파일하는 모드입니다. REPL 과 Jupyter 가 이 모드를 사용하며, AOT 와 대조됩니다.

## lang slot / 매직

lang slot 은 언어가 라이브러리의 특정 항목을 식별하는 메커니즘입니다. `#[lang(slot_name)]` 으로 표준 라이브러리 항목을 컴파일러 슬롯에 연결합니다.

## LOS (Large Object Space)

LOS 는 GC 의 큰 객체 전용 공간입니다. 임계(8 KB / 2 KB) 이상의 객체는 LOS 에 이동 없이 그대로 둡니다.

## monomorphization (단형화)

단형화는 generic 함수를 호출 지점의 구체 타입별로 별도로 컴파일하는 과정을 뜻합니다. 간접 호출 비용이 0 이라는 이점을 얻는 대신, 컴파일 시간과 바이너리 크기를 비용으로 치릅니다.

## RHIR (rok HIR bytecode)

RHIR 은 미리 빌드한 라이브러리 패키지의 인터페이스 파일 (`<name>.rhir`) 입니다. archive 옆에 놓이며, 다운스트림 패키지가 선언을 읽고 generic fn 을 specialize 할 수 있게 합니다. body 는 구체 함수 본문을 잘라 낸 패키지 소스입니다 ([9.4](../009-tools/904-rhir-bytecode.md)).

## niche optimization

niche optimization 은 합 타입이나 enum 의 사용하지 않는 비트 패턴을 활용하여 태그 자리를 없애는 최적화입니다. 대표적인 예는 `Ptr | none` 의 `none = null pointer` 입니다.

## scan_table

scan_table 은 GC 가 객체의 어떤 슬롯이 GC 포인터인지 알아내기 위한 메타 테이블입니다. `type_id → scan_table` 로 매핑되며, 컴파일러가 빌드 시점에 등록합니다.

## shadow stack

shadow stack 은 GC 가 살아 있는 객체를 추적하기 위한 보조 스택입니다. 컴파일러가 함수 진입과 종료 시점에 GC 객체 변수 슬롯을 담은 frame 을 link, unlink 하는 코드를 직접 emit 합니다. frame 은 기계 스택 위에 있고, 체인 머리는 thread 마다 TLS 에 둡니다.

## specialize

specialize 는 [monomorphization (단형화)](#monomorphization-단형화) 의 다른 이름입니다. AOT / JIT 파이프라인의 한 단계 이름으로 사용합니다.

## stone

레지스트리에 올린 rok 패키지입니다. `rok.rokon` 에서 `Dep::version("req")` 나 문자열 `"req"` 로 요구합니다 ([9.10.4a](../009-tools/910-rok-frontend.md#9104a-stone-과-rokstonesio)).

## sticky mark-bit

sticky mark-bit 은 generational GC 의 세대 추적 비트입니다. 처음 mark 되면 켜져서 "오래된 객체" 가 됩니다. young GC 는 sticky 객체를 건너뛰어 비용을 줄입니다.

## sum type / 합 타입

합 타입은 `A | B` 형태로 둘 중 하나를 담는 타입이며, tagged union 의 한 형태입니다. 닫힌 슬롯이며, 모든 변형이 명시되어 패턴 매칭의 완전성 검사가 가능합니다.

## TLAB (Thread-Local Allocation Buffer)

TLAB 은 각 worker thread 가 가진 thread-local 메모리 버퍼입니다. fast-path 할당이 bump pointer 만으로 이루어지며, 동기화 비용은 0 입니다.

## work-stealing

work-stealing 은 스케줄러의 load balancing 메커니즘입니다. 각 worker 가 지역 deque 를 가지며, 일이 떨어진 worker 가 다른 worker 의 deque 끝에서 task 를 훔쳐 옵니다.

## unsafe block / unsafe fn

`unsafe { ... }` 블록 표현식과 `unsafe fn` 함수 modifier 는 안전 추상의 보장 밖에 있는 영역을 시각적으로 가두는 rok 의 단일 메커니즘입니다. raw pointer dereference, `unsafe fn` 호출, MLIR op 직접 호출 등이 unsafe scope 안에서만 허용됩니다. 자세한 내용은 [4.15 unsafe](../004-functions/415-unsafe.md) 를 참조하시면 됩니다.

## MLIR plugin

MLIR plugin 은 rok 컴파일러에 외부에서 빌드된 MLIR 다이얼렉트를 동적으로 등록하기 위한 메커니즘입니다. `--load-mlir-plugin=<path>` CLI 옵션이나 패키지 매니페스트 선언으로 로드되며, 로드된 op 은 `rok::bare::mlir::__mlir_op` 으로 호출 가능해집니다. 도메인 특화 컴파일러 (암호화, 신호 처리, 새 가속기 ISA 등) 영역에서 활용합니다. 자세한 내용은 [refs-standard-libraries 1.10](../../refs-standard-libraries/100-bare/110-mlir-plugin.md) 을 참조하시면 됩니다.

## rokc / rok / rokls / rokr

toolchain 을 이루는 실행 파일과 런타임입니다. `rokc` 는 컴파일러 (rustc 에 해당, [9.1](../009-tools/901-rokc-cli.md)), `rok` 은 매니페스트와 의존성, 환경, toolchain 고정을 맡고 `rokc` 를 부르는 cargo 식 front-end ([9.10](../009-tools/910-rok-frontend.md)), `rokls` 는 편집기용 LSP 서버 실행 파일 (`rok lsp`), `rokr` 은 런타임 (`librokr.a`) 입니다.

## RokON

Rok Object Notation. rok 의 리터럴 문법 부분집합으로 값을 적는 데이터 형식이며, 확장자는 `.rokon` 입니다. 매니페스트 `rok.rokon` 과 lockfile `rok.lock` 이 이 형식입니다 ([stdlib 3.9](../../refs-standard-libraries/300-std/309-rokon.md)).

## rokstones.io

rok 의 기본 stone 레지스트리입니다. index 는 별도 git 저장소 `rokstones.io-index` 입니다 ([9.10.4a](../009-tools/910-rok-frontend.md#9104a-stone-과-rokstonesio)).

## rok.rokon / rok.lock

패키지 루트의 매니페스트와 lockfile 입니다. 이름은 이 철자 그대로 (모두 소문자) 이며, 둘 다 [RokON](#rokon) 형식입니다. 의존성이 없는 패키지에는 필요하지 않습니다 ([0.3.4](003-source-units.md#034-매니페스트-rokrokon-는-선택)). `rok.rokon` 은 `Package { .. }` 값 하나로 의존성과 toolchain 고정을 선언하고, `rok.lock` 은 `rok update` 가 해소한 결과를 고정합니다.

## 환경 (environment)

`rok` 이 의존성 집합을 두는 단위입니다. `rok.rokon` 이 있는 모든 디렉토리가 환경이며 활성화 단계는 없습니다. 프로젝트 밖에서는 `$ROK_HOME/envs/` 의 이름 있는 환경 (기본 `default`) 을 씁니다 ([9.10.5](../009-tools/910-rok-frontend.md#9105-환경)).

여기에 정리한 항목들은 단지 단어의 뜻을 옮긴 것이 아니라, 컴파일 파이프라인, 메모리 관리, 타입 체계, 동시성 모델 등 rok 이 어떤 토대 위에 서 있는지를 압축적으로 보여 주는 색인이기도 합니다. 본문을 읽다가 개념이 흐릿해질 때마다 이 장으로 돌아오면, 각 용어가 사양 전체에서 어떤 자리를 차지하고 있는지를 더욱 또렷하게 가늠할 수 있을 것입니다.
