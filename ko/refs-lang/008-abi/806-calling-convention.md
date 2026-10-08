# 8.6. Calling Convention

함수 호출의 레지스터·스택 사용 규약을 정의한다. 본 문서는 플랫폼 C [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 를 따르되 변형이
필요한 부분만 명시한다.

## 8.6.1. 기본: 플랫폼 C ABI

[Extern "C" 심볼](805-extern-symbols.md) 의 모든 함수와 일반 nok 함수의 기본 호출 규약은 각 플랫폼의
C ABI.

| 플랫폼                   | ABI                |
| ------------------------ | ------------------ |
| Linux x86_64             | System V AMD64     |
| Linux aarch64            | AAPCS64            |
| Linux i686               | System V i386      |
| Linux armv7              | AAPCS              |
| macOS (x86_64 / aarch64) | System V / AAPCS64 |
| Windows x86_64           | MS x64             |
| Windows aarch64          | AAPCS64 (MS 변형)  |
| WASM                     | WebAssembly ABI    |

이 규약은 LLVM 백엔드가 자동 처리한다. 본 사양은 변형 / 추가 사항만 정의한다.

## 8.6.2. 레지스터 인자 / 반환

각 ABI 의 기본 인자 / 반환 레지스터를 그대로 사용한다.

| 플랫폼         | 인자 레지스터 (정수)       | 반환 (정수) |
| -------------- | -------------------------- | ----------- |
| Linux x86_64   | rdi, rsi, rdx, rcx, r8, r9 | rax, rdx    |
| Windows x86_64 | rcx, rdx, r8, r9           | rax         |
| aarch64        | x0 ~ x7                    | x0, x1      |
| i686           | (스택만)                   | eax, edx    |
| armv7          | r0 ~ r3                    | r0, r1      |

부동소수 / 벡터는 각 ABI 의 표준 (xmm0-7 / vN / sN 등).

## 8.6.3. `struct` 인자 / 반환: small `struct` rule

작은 `struct` 는 레지스터에 직접 패킹된다.

| 플랫폼          | small `struct` 한계    |
| --------------- | ---------------------- |
| System V x86_64 | 16 바이트 (2 레지스터) |
| Windows x86_64  | 8 바이트 (1 레지스터)  |
| aarch64         | 16 바이트 (2 레지스터) |

이 한계 안의 `struct` 는 레지스터 전달. 초과는 호출자가 임시 메모리에 저장 후 포인터 전달.

이는 LLVM 의 byval / sret 자동 처리.

## 8.6.4. GC 객체 포인터: 일반 포인터로

GC 객체 포인터는 플랫폼의 일반 포인터와 동일 ABI 로 전달한다. 특별 레지스터 / tagged pointer
사용 안 함.

다만 호출 전후에 [배리어와 루트 추적](../007-runtime/704-barriers-and-roots.md) 의 [shadow stack](../000-meta/007-glossary.md#shadow-stack) 에 등록되어 있어야 한다.

이 등록은 `llvm.gcroot` intrinsic 으로 자동 처리된다.

## 8.6.5. Setjmp / Longjmp: 예외 처리

[에러 제어 흐름](../003-expressions/314-error-control-flow.md) 의 `try` / `catch` / `throw` 는 `setjmp` / `longjmp` 또는 LLVM 의 EH
intrinsic 으로 구현한다.

### 옵션 A: setjmp / longjmp

```
try { body } catch e: T { handler }
```

→ lowering:

```mlir
; jmp_buf 할당
%jbuf = alloca jmp_buf

; setjmp
%r = call i32 @setjmp(jmp_buf* %jbuf)
%is_normal = icmp eq i32 %r, 0
br i1 %is_normal, label %try_body, label %catch

try_body:
  ; ... body ...
  br label %end

catch:
  ; e 를 받아 handler 실행
  ; ...
  br label %end

end:
```

`throw expr` →

```mlir
call void @nok_throw(i8* %exception)
; 가장 가까운 jmp_buf 로 longjmp
```

자세한 메커니즘은 `nokr` (facade) 가 제공하는 `nok_throw` 구현.

### 옵션 B: LLVM EH intrinsic

LLVM 의 `invoke` / `landingpad` / `personality function` 사용. C++ EH 방식.

장점은 zero-cost happy path (정상 흐름엔 비용 없음).
단점은 DWARF / 플랫폼 EH 인프라 필요.

### nok 의 선택

ClangIR (`cir.eh.setjmp` 패턴) 을 따라 setjmp / longjmp 우선. 이유:

- 모든 플랫폼에서 작동 (특히 WASM).
- Shadow stack 과 자연스럽게 공존 (longjmp 시 shadow stack 도 unwind).
- 평균 비용은 정상 흐름에서 setjmp 한 번 정도.

LLVM EH 는 향후 지원 가능 (옵션).

## 8.6.6. Calling convention 의 nok 변형: 없음

nok 자체는 C ABI 를 그대로 따른다. 별도 변형 없음.

이 결정은:

- [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 호출이 자유로움. 사용자 / FFI 함수가 동일 ABI 라 wrapper 비용 없음.
- 디버거 / 도구 친화성. 일반 C 디버거가 nok 함수 stack 도 그대로 본다.

## 8.6.7. tail call

LLVM 의 `tail call` 최적화는 일반 함수에서 자동 적용. 코루틴에는 적용 안 함 (state machine
구조).

## 8.6.8. inline 함수의 ABI

`#[inline]` ([내장 속성](../006-meta-system/602-built-in-attributes.md)) 의 효과는 호출 자체가 사라질 수 있다는 점. 이 경우 calling convention
은 의미 없음.

`#[inline(never)]` 는 항상 호출. 일반 ABI.

## 8.6.9. 매개변수 전달의 평가 순서

[매개변수](../004-functions/402-parameters.md) 의 좌→우 평가 순서는 calling convention 과 직교다.

평가 순서가 좌→우 라는 말은, 각 인자 표현식을 좌측부터 평가한다는 뜻이다. 그 결과를 어떤
레지스터나 스택 자리에 두는지는 ABI 가 정한다.

C / C++ 의 일부 ABI 는 우→좌 평가를 권장한다 (스택 push 순서). nok 은 좌→우 의미 보장. 컴파일
러가 임시 변수 / 레지스터를 적절히 사용해 둘을 일치시킨다.

## 8.6.10. 가변 인자 (variadic) 함수

[매개변수](../004-functions/402-parameters.md) 의 `...T` variadic 은 언어 레벨 variadic (slice 로 묶어 전달).
플랫폼의 C variadic (`...` 가변 인자) 은 별개.

C 의 variadic 함수 호출 (예: `printf`) 은 `extern "C"` 로 직접 호출 가능. ABI 는 플랫폼 C 가변
인자 규약.

```
extern "C" {
    fn printf(fmt: ConstPtr<c_char>, ...);
}
```

가변 인자 함수의 호출 측은 플랫폼별로 약간 다르다. LLVM 이 자동 처리.

## 8.6.11. 함수 포인터 ABI

[함수 타입](../004-functions/404-function-type.md) 의 `fn(...) -> ...` 은 간접 호출 시에도 같은 calling convention.

```mlir
%fn_ptr = ; ... 함수 포인터 ...
%result = call i32 %fn_ptr(...)
```

인자 / 반환 ABI 가 동일하므로 직접 호출과 간접 호출의 인터페이스가 같다.

## 8.6.12. 클로저 호출

[클로저](../004-functions/405-closure.md) 의 클로저는 환경과 코드 포인터의 묶음.

호출 시:

```
환경 포인터를 첫 인자로, 그 뒤 일반 인자들.
```

이 환경 인자가 클로저 본체에서 캡처된 변수에 접근하는 통로다.

LLVM 이 일반 함수처럼 컴파일한다 (환경이 그냥 첫 인자).

## 8.6.13. 비동기 함수 호출

[비동기](../004-functions/412-async.md) 의 `async fn` 은 호출 즉시 Future ptr 반환. 일반 함수 호출 ABI.

```
let f = async_fn(args);     // 즉시 반환 (Future ptr)
let r = await f;             // 코루틴 메커니즘
```

`await` 자체는 `nok_coro_park` ([Extern "C" 심볼](805-extern-symbols.md)) 호출 + 재개. 일반 ABI 위에 작동한다.

## 8.6.14. 안정성

본 calling convention 은 플랫폼 C ABI 를 따르므로 별도 안정성 결정이 없다. 플랫폼이 호환을
보장하면 nok 도 호환.

setjmp / longjmp 메커니즘은 런타임 ABI 의 일부. 변경 시 [개요](README.md) 의 안정성 정책.
