# 8.6. Calling Convention

이 문서에서는 함수 호출 시 사용되는 레지스터와 스택의 사용 규약을 정의합니다. 본 문서는 기본적으로 플랫폼 C [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 를 따르며, 변형이 필요한 부분에 한해서만 별도로 명시합니다.

## 8.6.1. 기본: 플랫폼 C ABI

[Extern "C" 심볼](805-extern-symbols.md) 로 선언된 모든 함수는 각 플랫폼의 C ABI 를 씁니다. rok 함수 사이의 호출은 같은 레지스터를 쓰지만 집계 값은 다르게 넘깁니다 (8.6.6).

| 플랫폼                   | ABI                |
|--------------------------|--------------------|
| Linux x86_64             | System V AMD64     |
| Linux aarch64            | AAPCS64            |
| Linux i686               | System V i386      |
| Linux armv7              | AAPCS              |
| macOS (x86_64 / aarch64) | System V / AAPCS64 |
| Windows x86_64           | MS x64             |
| Windows aarch64          | AAPCS64 (MS 변형)  |
| WASM                     | clang 의 WebAssembly C ABI |

LLVM 백엔드는 스칼라 인자의 레지스터 배치를 처리하지만, 집계 (struct / 튜플 / 배열) 를 C 경계에서 어떻게 넘길지 분류하는 일은 하지 않습니다. 그 분류는 **프론트엔드 (rokc) 의 일** 이며, rokc 는 clang 과 같은 규칙으로 분류한 결과를 LLVM 에 넘깁니다 (8.6.3). 본 사양은 그 밖에 변형되거나 추가되는 사항만 정의합니다. WebAssembly C ABI 에서 스칼라 하나로 된 집계는 그 스칼라로 넘기고, 그 밖의 집계는 호출자 사본의 포인터로 넘기며 (결과는 `sret`), 런타임 `str` 인자는 참조로 넘깁니다.

## 8.6.2. 레지스터 인자 / 반환

각 ABI 가 정의하는 기본 인자 및 반환 레지스터를 그대로 사용합니다.

| 플랫폼         | 인자 레지스터 (정수)       | 반환 (정수) |
|----------------|----------------------------|-------------|
| Linux x86_64   | rdi, rsi, rdx, rcx, r8, r9 | rax, rdx    |
| Windows x86_64 | rcx, rdx, r8, r9           | rax         |
| aarch64        | x0 ~ x7                    | x0, x1      |
| i686           | (스택만)                   | eax, edx    |
| armv7          | r0 ~ r3                    | r0, r1      |

부동소수점이나 벡터 값의 경우에도 각 ABI 가 정한 표준 레지스터 (xmm0-7, vN, sN 등) 를 그대로 따르고 있습니다.

## 8.6.3. `struct` 인자 / 반환: small `struct` rule

C 경계 (`extern "C"` 선언과 정의) 에서 집계를 값으로 넘길 때, rokc 는 clang 과 같은 규칙으로 타깃 ABI 에 따라 분류해 그 집계를 LLVM 이 레지스터로 옮길 수 있는 형태로 바꿉니다 (coercion). HFA 는 최대 4 개의 float / double 레지스터에 들어갑니다.

| 타깃                                      | 분류                                                                                      |
|-------------------------------------------|-------------------------------------------------------------------------------------------|
| AAPCS64 (Linux / Apple / Windows arm64)   | HFA (원소 1 ~ 4 개의 같은 부동소수) 는 `[n x f32]` / `[n x f64]`. 그 밖의 16 바이트 이하 집계는 `i64` / `[2 x i64]` (8 바이트 이하 반환값은 `iN`). 더 크면 인자는 호출자 사본의 포인터, 반환은 `sret` |
| System V x86_64                           | eightbyte 마다 INTEGER / SSE 로 분류해 정수 / SSE 레지스터에. 레지스터가 모자라면 집계 전체를 `byval`. 16 바이트 초과는 인자 `byval`, 반환 `sret` |
| Windows x64                               | 크기 1 / 2 / 4 / 8 바이트 집계는 `iN`. 그 밖은 인자는 참조 (호출자 사본의 포인터), 반환은 `sret` |
| 그 밖의 타깃                              | C 경계를 넘는 집계 값 전달은 컴파일 에러                                                  |

- 좁은 정수 매개변수와 반환값 (`i8` / `i16` / `u8` / `u16` / `bool` / `c_char`) 의 `signext` / `zeroext` 는 플랫폼 C ABI 를 따릅니다. x86_64 SysV 와 Apple arm64 는 `i8` / `i16` / `bool` 을 확장하고, Linux AAPCS64 는 확장하지 않으며, Windows x64 는 `bool` 만 확장합니다.
- 모든 함수는 inline stack probe 를 가지므로, 큰 frame 도 guard page 를 건너뛰지 않고 건드립니다.
- `extern "C" fn` **정의** 에도 같은 분류를 적용합니다. 정의는 C 가 부르는 C-ABI 진입점을 따로 가지며, rok 쪽 호출은 그대로 rok 규약을 씁니다.
- System V i386 에서 `i64` / `u64` / `f64` 의 집계 안 정렬은 4 바이트입니다 (C 와 같음).
- rok 함수끼리의 호출에는 이 분류를 적용하지 않습니다 (8.6.6). 분류는 C 와 맞물리는 자리에만 필요하며, `str` (16 바이트 구조체, Windows x64 에서는 참조) 을 받거나 돌려주는 런타임 함수도 여기에 듭니다 ([8.5](805-extern-symbols.md)).

## 8.6.4. GC 객체 포인터: 일반 포인터로

GC 객체 포인터는 플랫폼이 사용하는 일반 포인터와 동일한 ABI 로 전달됩니다. 별도의 특별 레지스터나 tagged pointer 는 사용하지 않습니다.

다만 호출 전후로 [배리어와 루트 추적](../007-runtime/704-barriers-and-roots.md) 의 [shadow stack](../000-meta/007-glossary.md#shadow-stack) 에 반드시 등록되어 있어야 합니다.

이 등록은 컴파일러가 emit 하는 shadow stack frame 의 link / unlink 로 이루어집니다 ([8.9.12](809-lowering-control.md#8912-shadow-stack-frame)).

## 8.6.5. throwing 함수의 반환: 값 기반 에러 전달

[`throws` 시그니처](../004-functions/414-throws.md) 를 가진 함수는 에러를 반환값의 일부로 돌려줍니다. 별도의 unwinding 메커니즘이나 런타임 심볼은 사용하지 않습니다.

```
fn f(args) throws (E) -> T
```

위 함수의 ABI 상 반환 타입은 다음 두 필드의 `struct` 입니다.

```mlir
!Throws_T_E = !llvm.struct<(i1, !Payload_T_E)>
; 필드 0: is_err - 0 이면 성공, 1 이면 에러
; 필드 1: payload - union(T, E). is_err 가 0 이면 T, 1 이면 E 로 해석
```

payload 는 `T` 와 `E` 가운데 큰 쪽의 크기와 정렬을 갖는 union 입니다. `T` 가 unit 이면 payload 는 `E` 만 담습니다. 에러 채널은 시그니처로 구분되므로 `T` 와 `E` 가 같은 타입이어도 문제가 없습니다. untyped `throws` (`throws (any Error)`) 의 `E` 는 [`any`](../002-types/202-any-slot.md) 와 같은 tag + payload 표현의 `any Error` 값입니다 ([8.8](808-lowering-types.md)).

필드 0 은 `i1` 보다 넓게 저장될 수 있습니다 (payload 가 GC 참조를 담으면 수집기가 합의 tag 로 읽도록 `i32`). 의미 있는 것은 bit 0 뿐이며, payload 가 4 이상으로 정렬되면 offset, 크기, 레지스터는 `{i1, payload}` 의 것과 같습니다. throwing 함수의 `ref` 매개변수 ([4.7](../004-functions/407-ref-keyword.md)) 의 값은 반환값의 구현 정의 부분으로서 carrier 옆에 함께 전달됩니다.

이 `struct` 는 다른 반환값과 똑같이 플랫폼 C ABI 로 전달됩니다. 8.6.3 의 한계 이내이면 레지스터로, 초과하면 sret 으로 돌려줍니다. 호출 지점의 `try` 는 필드 0 을 검사해 분기하며, 자세한 lowering 은 [8.9.6](809-lowering-control.md#896-try--catch--finally-값-기반-분기) 에서 정의합니다.

이 방식은 다음과 같은 이유로 채택되었습니다.

- 모든 플랫폼에서 같은 방식으로 동작합니다 (WASM 을 포함하며, DWARF 나 플랫폼 EH 인프라가 필요 없습니다).
- [shadow stack](../000-meta/007-glossary.md#shadow-stack) 과 충돌하지 않습니다. 에러도 일반 반환 경로로 frame 을 빠져나가므로, frame 마다 shadow stack 을 정상적으로 pop 합니다.
- 에러가 나지 않는 경로의 비용은 반환 후 분기 하나뿐이며, `try` 나 `catch` 에 진입하는 비용은 없습니다.

## 8.6.6. rok 사이의 호출 규약

rok 함수 사이의 호출은 스칼라를 8.6.2 의 레지스터로 넘깁니다. 64 바이트를 넘는 집계 값 (구조체, 튜플, 배열, `str`) 은 주소로 전달하고, 맨 앞의 결과 포인터를 통해 돌려받습니다. 피호출자는 주소로 받은 인자를 진입 시 복사하므로 호출자의 값은 바뀌지 않습니다. 피호출자는 주소로 받은 인자를 복사한 뒤, 반환하기 전 어느 시점에든 결과 포인터를 통해 결과를 쓸 수 있습니다. 그래서 결과 포인터는 주소로 받은 인자를 가리킬 수는 있지만, 피호출자가 호출 중에 읽는 그 밖의 메모리는 가리킬 수 없습니다. 이것은 미리 빌드한 코드, export 된 함수, 함수 값의 호출을 포함한 모든 rok 사이의 호출에 적용됩니다. 직접 호출은 호출자의 `rok_tls` 블록 주소 ([8.3.8](803-tls-slots.md#838-컴파일러-emit-의-형식)) 도 선언된 인자 뒤의 숨은 마지막 포인터 인자로 넘깁니다. 함수 값, `#[export_name]` rok 함수, `any` dispatcher 는 받지 않습니다 (8.6.11). 그보다 작은 집계는 first-class 집계로 넘기고 돌려주며, 대상에서 LLVM SelectionDAG 가 낮추는 방식 그대로 낮춥니다. 예외가 하나 있습니다: 32-bit Arm 에서 원소가 1 또는 2 바이트인 배열은 64 바이트를 넘는 집계처럼 주소로 넘기고 돌려받습니다. 그런 스택 인자에 대한 LLVM 자신의 lowering 이 호출자와 피호출자 사이에서 다르기 때문입니다. 두 backend 가 정확히 이 방식을 따르므로 어느 쪽 코드든 서로 호출할 수 있습니다. 8.6.3 의 C ABI 분류와는 다릅니다. 아래 규칙은 64 바이트 이하의 집계에 적용됩니다.

> [!NOTE]
> 주소로 넘기는 값은 메모리에서 (`memcpy`) 옮기며, first-class 집계로 옮기지 않습니다.

**인자**

- 집계는 스칼라 leaf 로 나뉘어 각자 자기 종류의 다음 레지스터를 차지하며, 레지스터가 다하면 나머지 leaf 는 스택으로 갑니다. 그래서 `str` 이 마지막 인자 레지스터와 스택에 나뉠 수 있습니다.
- AArch64: `i64` / `ptr` / `f32` / `f64` 배열은 자기 종류의 연속된 레지스터를 차지합니다. 남은 레지스터에 다 들어가지 않으면 통째로 스택에 놓이고, 그 종류의 나머지 레지스터는 쓴 것으로 칩니다. `i8` / `i16` / `i32` 배열은 구조체처럼 나뉩니다.
- 스택 슬롯: Apple 대상에서는 최상위 스칼라 인자가 자연 크기와 정렬을 쓰고, 집계의 `i1` / `i8` / `i16` leaf 는 4 바이트 슬롯을 씁니다. 그 밖의 대상에서는 모든 스택 슬롯이 8 바이트입니다.

**결과**

- AArch64: 정수 레지스터 8 개 (`x0`–`x7`) 와 부동소수점 레지스터 8 개 (`v0`–`v7`) 까지. 더 큰 결과는 `x8` 로 받는 sret 포인터로 반환합니다 (포인터는 돌려주지 않음).
- x86-64: 정수 레지스터 셋 (`rax`, `rdx`, `rcx`) 까지, 부동소수점 결과는 `xmm0` / `xmm1` 다음 x87 `st0` / `st1`. 네 번째 정수 레지스터나 다섯 번째 부동소수점 레지스터가 필요한 결과는 `rdi` 로 받고 `rax` 로 돌려주는 sret 포인터로 반환합니다.

**Windows**

- Windows x64: 정수와 부동소수점 인자는 네 개의 레지스터 자리 (`rcx` `rdx` `r8` `r9` / `xmm0`–`xmm3`) 를 함께 씁니다. 각 인자는 다음 자리를 차지하되 자기 종류의 레지스터를 씁니다. 스택 슬롯은 호출자의 32 바이트 home area 뒤에 놓이며 8 바이트입니다. sret 포인터는 `rcx` 로 넘기고, 그 밖의 결과는 x86-64 와 같습니다.
- Windows arm64: 위의 AAPCS64 와 같습니다. C variadic 호출은 모든 부동소수점 인자를 정수 레지스터로 넘깁니다.

**좁은 정수**

피호출자는 좁은 정수와 `bool` 인자 및 결과의 위쪽 비트에 기대지 않습니다. 호출자는 확장해도 됩니다. native backend 는 들어온 `bool` 을 bit 0 으로 mask 하고 좁은 값을 다시 확장합니다.

다른 모듈이 닿을 수 없는 함수 (내부용이며 값으로 쓰이지 않음) 는 구현이 정하는 규약을 쓸 수 있습니다.

`extern "C"` 와 `extern "rok"` 런타임 함수의 호출은 8.6.1–8.6.5 의 C ABI 규칙을 씁니다.

## 8.6.7. tail call

LLVM 의 `tail call` 최적화는 일반 함수에서 자동으로 적용됩니다. 다만 코루틴에는 state machine 구조의 특성상 적용하지 않습니다.

## 8.6.8. inline 함수의 ABI

`#[inline]` ([내장 속성](../006-meta-system/602-built-in-attributes.md)) 의 효과는 호출 자체가 사라질 수 있다는 점에 있습니다. 이 경우에는 calling convention 자체가 의미를 가지지 않습니다.

`#[inline(never)]` 는 항상 호출되며, 일반 ABI 를 따릅니다.

## 8.6.9. 매개변수 전달의 평가 순서

[매개변수](../004-functions/402-parameters.md) 의 좌→우 평가 순서는 calling convention 과 직교적인 관계에 있습니다.

평가 순서가 좌→우 라는 말은 각 인자 표현식을 좌측부터 차례로 평가한다는 뜻입니다. 그 결과를 어떤 레지스터나 스택 자리에 두는지는 ABI 가 결정하는 별개의 문제입니다.

C 와 C++ 의 일부 ABI 는 스택 push 순서를 이유로 우→좌 평가를 권장하기도 합니다. 그러나 rok 은 좌→우 의미를 보장하며, 컴파일러가 임시 변수와 레지스터를 적절히 활용하여 두 요건을 일치시키도록 처리합니다.

## 8.6.10. 가변 인자 (variadic) 함수

[매개변수](../004-functions/402-parameters.md) 의 `...T` variadic 은 언어 레벨의 variadic 으로서 인자들을 하나의 슬라이스 view (`Slice<T>`) 로 묶어 전달합니다 (보통 struct 인자와 같은 ABI). 이는 플랫폼의 C variadic (`...` 가변 인자) 과는 별개의 개념입니다.

C 의 variadic 함수 호출 (예를 들어 `printf`) 은 `extern "C"` 를 통해 직접 호출이 가능하며, 이때의 ABI 는 플랫폼이 정의한 C 가변 인자 규약을 따릅니다.

```
extern "C" {
    fn printf(fmt: ConstPtr<c_char>, ...);
}
```

- `...` 는 `extern` 블록의 선언에서만, 마지막 매개변수로만 쓸 수 있습니다. rok 함수 정의는 C variadic 을 가질 수 없습니다.
- 가변 자리의 인자에는 C 의 default argument promotion 을 적용합니다. `bool`, `i8`, `i16`, `u8`, `u16`, `c_char` 는 `c_int` (`i32`) 로, `f32` 는 `f64` 로, `str` 은 `char*` (NUL 로 끝나는 바이트 포인터) 로 넘어갑니다.
- 집계 (struct / 튜플 / 배열) 는 가변 인자로 넘길 수 없습니다 (컴파일 에러).

## 8.6.11. 함수 포인터 ABI

[함수 타입](../004-functions/404-function-type.md) 의 `fn(...) -> ...` 은 간접 호출 시에도 동일한 calling convention 을 사용합니다.

```mlir
%fn_ptr = ; ... 함수 포인터 ...
%result = call i32 %fn_ptr(...)
```

인자와 반환의 ABI 가 동일하므로, 직접 호출과 간접 호출의 인터페이스가 서로 같습니다. 다만 8.6.6 의 숨은 `rok_tls` 주소는 예외로, 함수 값은 받지 않습니다. 함수 값의 코드 포인터는 주소를 직접 읽어 함수에 넘겨 호출하는 thunk 입니다.

## 8.6.12. 클로저 호출

[클로저](../004-functions/405-closure.md) 에서 다루는 클로저는 환경과 코드 포인터를 묶은 하나의 단위입니다.

호출 시에는 다음과 같이 동작합니다.

```
환경 포인터를 첫 인자로, 그 뒤 일반 인자들.
```

이 환경 인자가 클로저 본체에서 캡처된 변수에 접근하는 통로 역할을 합니다.

LLVM 은 이를 일반 함수처럼 컴파일합니다 (환경이 단지 첫 번째 인자가 될 뿐입니다).

## 8.6.13. 비동기 함수 호출

[비동기](../004-functions/412-async.md) 의 `async fn` 은 호출 즉시 Future ptr 을 반환하며, 이는 일반 함수 호출 ABI 를 그대로 따릅니다.

```
let f = async_fn(args);     // 즉시 반환 (Future ptr)
let r = await f;             // 코루틴 메커니즘
```

`await` 는 상태 기계의 일반 호출 (대상 future 의 poll) 과 반환으로 이루어지며 ([8.10](810-lowering-coroutine-generic.md)), 일반 ABI 위에서 작동합니다.

## 8.6.14. 안정성

본 calling convention 은 플랫폼 C ABI 를 따르고 있으므로, 별도의 안정성 결정을 두지 않습니다. 플랫폼이 호환을 보장하는 범위 안에서 rok 또한 호환을 유지합니다.

throwing 함수의 반환값 레이아웃 (8.6.5) 은 런타임 ABI 의 일부이며, 변경이 필요한 경우 [개요](README.md) 의 안정성 정책을 따릅니다.
