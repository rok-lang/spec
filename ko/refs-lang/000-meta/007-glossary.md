# 0.7. 용어집

이 사양에는 같은 개념이 여러 장에 걸쳐 되풀이해 나옵니다. 그래서 자주 쓰는 용어를 한곳에 모아 둡니다. 본문을 읽다가 개념이 헷갈리면 이 문서를 바로 찾아보면 됩니다. 각 항목에는 짧은 정의와 함께 Rok에서 쓰는 구체적인 맥락이 있습니다. 사전적 뜻뿐 아니라 그 용어가 사양 안에서 차지하는 자리도 알 수 있습니다.

## ABI (Application Binary Interface)

ABI는 컴파일된 코드가 링크하고 실행할 때 따라야 하는 바이너리 수준의 계약입니다. 객체 레이아웃, 호출 규약, 심볼 명명 등이 여기에 들어갑니다. Rok의 ABI는 [8장: ABI / MLIR lowering](../008-abi/README.md)에서 정의합니다.

## AOT (Ahead-of-Time)

AOT는 소스를 미리 컴파일해서 정적 산출물(`.a` 또는 실행 파일)을 만드는 모드입니다. JIT와 대비됩니다. Rok은 JIT와 AOT 두 엔진을 모두 지원합니다. 두 엔진은 rok-mir까지 같은 파이프라인을 공유합니다.

## archive

아카이브(archive)는 컴파일된 패키지의 산출물이며, 표준 ar 형식의 `.a` 파일입니다. 아카이브에는 concrete 코드의 오브젝트가 들어 있습니다. 라이브러리의 인터페이스(ROKHIR)는 그 옆의 `<name>.rokhir` 파일입니다.

## block (블록)

블록은 `{ ... }`로 둘러싼 문장과 식의 묶음입니다. 블록 자체가 하나의 표현식입니다. 마지막 식이 블록의 값이 됩니다.

## block (Immix)

Immix의 block은 GC의 메모리 단위입니다. 32 KB의 OS 메모리 페이지에 해당합니다. Immix 알고리듬에서 block은 line으로 나뉘고, mark / sweep의 단위가 됩니다. 위의 코드 블록과는 다른 개념입니다.

## concrete fn / generic fn

호출 지점의 타입이 모두 구체 타입인 함수가 concrete fn이고, 타입 매개변수가 남아 있는 함수가 generic fn입니다. AOT에서는 generic fn을 호출 지점에서 특수화(specialize)해 concrete fn으로 만듭니다.

## dispatch (디스패치)

디스패치는 이름, 연산자, 메서드 호출을 어떤 구현에 연결할지 정하는 과정입니다. Rok은 정적 디스패치만 씁니다. 트레잇을 타입으로 쓸 수 없으므로 vtable 간접 호출이 없습니다.

## extern "C" 심볼

extern "C" 심볼은 C ABI로 노출하거나 호출하는 심볼입니다. 컴파일러와 런타임은 이런 심볼로 서로 연결됩니다.

## FFI (Foreign Function Interface)

FFI는 다른 언어(주로 C)의 함수를 호출하거나 다른 언어에서 호출받는 인터페이스입니다. `extern "C"` 블록으로 정의합니다.

## GC (Garbage Collector)

GC는 자동 메모리 관리 메커니즘입니다. Rok의 GC는 Immix mark-region과 Sticky Mark-Bit generational을 결합한 형태입니다.

## generic / 제네릭

제네릭은 타입을 매개변수로 받는 함수 또는 타입입니다. 호출 지점과 인스턴스화 지점에서 단형화됩니다.

## HIR (High-level IR) / MIR (Mid-level IR)

HIR과 MIR은 컴파일 파이프라인의 중간 표현입니다. 전체 흐름은 소스 → AST → HIR → MIR → 기계어(네이티브 백엔드) 또는 표준 MLIR → LLVM IR(MLIR → LLVM 백엔드)로 이어집니다.

## Immix

Immix는 Blackburn과 McKinley가 2008년에 발표한 GC 알고리듬입니다. 32 KB block과 128 B line 단위의 mark-region을 씁니다. 단편화는 opportunistic evacuation으로 해결합니다.

## JIT (Just-in-Time)

JIT는 코드를 런타임에 네이티브 백엔드로 즉시 컴파일하는 모드입니다. REPL과 Jupyter가 이 모드를 씁니다. AOT와 대비됩니다.

## lang slot / 매직

lang slot은 언어가 라이브러리의 특정 항목을 식별하는 메커니즘입니다. `#[lang(slot_name)]`으로 표준 라이브러리 항목을 컴파일러 슬롯에 연결합니다.

## LOS (Large Object Space)

LOS는 GC에서 큰 객체 전용 공간입니다. 임계값(8 KB / 2 KB) 이상의 객체는 LOS에 두고 옮기지 않습니다.

## monomorphization (단형화)

단형화는 제네릭 함수를 호출 지점의 구체 타입마다 따로 컴파일하는 과정입니다. 간접 호출 비용이 0이 되는 대신 컴파일 시간과 바이너리 크기가 늘어납니다.

## ROKHIR (Rok HIR bytecode)

ROKHIR은 미리 빌드한 라이브러리 패키지의 인터페이스 파일(`<name>.rokhir`)입니다. 이 파일은 아카이브 옆에 놓입니다. 다운스트림 패키지는 이 파일에서 선언을 읽고 generic fn을 특수화합니다. ROKHIR의 본문은 구체 함수의 본문을 잘라 낸 패키지 소스입니다([9.4](../009-tools/904-rokhir-bytecode.md)).

## niche optimization

niche optimization은 합 타입이나 enum에서 쓰지 않는 비트 패턴을 이용해 태그 자리를 없애는 최적화입니다. 대표적인 예가 `Ptr | none`의 `none = null pointer`입니다.

## scan_table

scan_table은 객체의 어느 슬롯이 GC 포인터인지 GC가 알아내는 데 쓰는 메타 테이블입니다. 매핑은 `type_id → scan_table`이고, 컴파일러가 빌드 시점에 등록합니다.

## shadow stack

shadow stack은 GC가 살아 있는 객체를 추적하는 데 쓰는 보조 스택입니다. 컴파일러는 함수 진입과 종료 시점에 프레임을 체인에 잇고 떼는 코드를 직접 내보냅니다. 프레임에는 GC 객체 변수의 슬롯이 들어 있습니다. 프레임은 기계 스택 위에 있고, 체인의 머리는 스레드마다 TLS에 둡니다.

## specialize

specialize는 [monomorphization (단형화)](#monomorphization-단형화)의 다른 이름입니다. AOT / JIT 파이프라인의 한 단계를 가리킬 때 씁니다.

## stone

레지스트리에 올린 Rok 패키지입니다. 이름은 `[a-z0-9_-]+` 형태이고 글자로 시작합니다. 코드에서는 `-`를 `_`로 바꿔 씁니다. 레지스트리에서 `-` / `_`와 대소문자는 같은 이름으로 봅니다. `rok.rokon`에서 `Dep::version("req")`나 문자열 `"req"`로 요구합니다([9.10.4a](../009-tools/910-rok-frontend.md#9104a-stone과-rokstonesio)).

## sticky mark-bit

sticky mark-bit는 generational GC에서 세대를 추적하는 비트입니다. 객체가 처음 표시(mark)될 때 켜집니다. 그때부터 그 객체는 "오래된 객체"입니다. young GC는 sticky 객체를 건너뛰어 비용을 줄입니다.

## sum type / 합 타입

합 타입은 `A | B` 형태로 둘 중 하나를 담는 타입입니다. tagged union의 한 형태입니다. 닫힌 슬롯이어서 모든 변형이 드러나 있습니다. 그래서 컴파일러가 패턴 매칭의 완전성을 검사할 수 있습니다.

## TLAB (Thread-Local Allocation Buffer)

TLAB은 워커 스레드마다 하나씩 있는 스레드 로컬 메모리 버퍼입니다. fast-path 할당은 bump pointer만으로 끝나므로 동기화 비용이 0입니다.

## work-stealing

work-stealing은 스케줄러의 부하 분산(load balancing) 메커니즘입니다. 워커마다 지역 deque가 있습니다. 일이 떨어진 워커는 다른 워커의 deque 끝에서 task를 훔쳐 옵니다.

## unsafe block / unsafe fn

`unsafe { ... }` 블록 표현식과 `unsafe fn` 함수 수식어는 Rok의 유일한 격리 메커니즘입니다. 안전 추상의 보장 밖에 있는 영역을 눈에 보이게 가둡니다. 원시 포인터 역참조, `unsafe fn` 호출, MLIR op 직접 호출 등은 unsafe 범위 안에서만 할 수 있습니다. 자세한 내용은 [4.15 unsafe](../004-functions/415-unsafe.md)에 있습니다.

## MLIR plugin

MLIR 플러그인은 외부에서 빌드한 MLIR 다이얼렉트를 Rok 컴파일러에 내장된 MLIR에 동적으로 등록하는 메커니즘입니다. `--load-mlir-plugin=<path>` CLI 옵션이나 패키지 매니페스트 선언으로 불러옵니다. 불러온 op은 `rok::bare::mlir::__mlir_op`으로 호출할 수 있습니다. 도메인 특화 컴파일러(암호화, 신호 처리, 새 가속기 ISA 등) 영역에서 씁니다. 자세한 내용은 [refs-standard-libraries 1.10](../../refs-standard-libraries/100-bare/110-mlir-plugin.md)에 있습니다.

## rokc / rok / rokls / rokr

toolchain을 이루는 실행 파일과 런타임입니다. `rokc`는 LLVM, MLIR, lld를 내장한 컴파일러(rustc에 해당, [9.1](../009-tools/901-rokc-cli.md))입니다. `rok`은 cargo 같은 프런트엔드([9.10](../009-tools/910-rok-frontend.md))입니다. 매니페스트, 의존성, 환경, toolchain 고정을 맡고 `rokc`를 호출합니다. `rokls`는 에디터용 LSP 서버 실행 파일(`rok lsp`)이고, `rokr`은 런타임(`librokr.a`)입니다. `$ROK_HOME/bin/`의 실행 파일은 활성 toolchain(`$ROK_HOME/toolchains/<version>-<triple>/`)으로 가는 프록시입니다. `rokc`는 표준 라이브러리와 런타임을 자기 실행 파일 기준의 sysroot에서 찾습니다([9.2.19](../009-tools/902-aot-pipeline.md#9219-설치-레이아웃)).

## ROKON

Rok Object Notation. Rok의 리터럴 문법 부분집합으로 값을 적는 데이터 형식입니다. 확장자는 `.rokon`입니다. 매니페스트 `rok.rokon`과 lockfile `rok.lock`이 이 형식입니다([stdlib 3.9](../../refs-standard-libraries/300-std/309-rokon.md)).

## rokstones.io

Rok의 기본 stone 레지스트리입니다. index는 별도 git 저장소 `rokstones.io-index`입니다([9.10.4a](../009-tools/910-rok-frontend.md#9104a-stone과-rokstonesio)).

## rok.rokon / rok.lock

패키지 루트의 매니페스트와 lockfile입니다. 이름은 이 철자 그대로(모두 소문자) 씁니다. 둘 다 [ROKON](#rokon) 형식입니다. 의존성이 없는 패키지에는 필요하지 않습니다([0.3.4](003-source-units.md#034-매니페스트rokrokon는-선택-사항)). `rok.rokon`은 `Package { .. }` 값 하나로 의존성과 toolchain 고정을 선언하고, `rok.lock`은 `rok update`가 해소한 결과를 고정합니다.

## 환경(environment)

`rok`이 의존성 집합을 두는 단위입니다. `rok.rokon`이 있는 모든 디렉터리가 환경입니다. 활성화 단계는 없습니다. 프로젝트 밖에서는 `$ROK_HOME/envs/`의 이름 있는 환경(기본 `default`)을 씁니다([9.10.5](../009-tools/910-rok-frontend.md#9105-환경)).

이 항목은 단어 뜻만 옮긴 것이 아닙니다. 컴파일 파이프라인, 메모리 관리, 타입 체계, 동시성 모델 등 Rok이 딛고 선 토대를 간추린 색인이기도 합니다. 본문을 읽다가 개념이 흐려지면 이 장으로 돌아와 각 용어가 사양 전체에서 차지하는 자리를 확인하세요.
