# 4.15. `unsafe` - 함수 modifier 와 블록 표현식

`unsafe` 키워드는 컴파일러가 안전성을 보증하지 않는 영역을 시각적으로 가두는 두 가지 형태로 등장합니다. **함수 / 메서드 선언의 modifier** 와 **블록 표현식** 이 그 두 자리이며, 두 자리는 한 쌍으로 작동합니다. `unsafe fn` 으로 선언된 함수는 호출자가 반드시 `unsafe { ... }` 블록 안에서 호출해야 합니다.

```
unsafe_fn        ::= "unsafe" fn_decl
unsafe_block     ::= "unsafe" block_expr
```

이 모델은 Rust 의 `unsafe` 와 동형이며, 위험 영역을 코드 위에서 명시적으로 드러내는 것이 그 목적입니다.

## 4.15.1. `unsafe fn` - 함수 modifier

함수 또는 메서드 선언 앞에 `unsafe` 를 두면, 그 함수의 본체는 안전 추상의 보장 밖에 있다는 뜻이 됩니다. 호출자가 사전 조건 (precondition) 을 만족시킬 책임을 진다고 시그니처가 곧바로 알려 줍니다.

```rok
impl<T> Ptr<T> {
    unsafe fn read(self) -> T;
    unsafe fn write(self, value: T);
}

extern "C" {
    unsafe fn strlen(s: ConstPtr<c_char>) -> usize;
}

unsafe fn transmute_unchecked<A, B>(a: A) -> B { ... }
```

`unsafe fn` 자체의 본체 안에서는 별도의 `unsafe { ... }` 블록 없이도 unsafe 호출이 가능합니다. 함수 시그니처가 이미 unsafe 를 선언했기 때문에 본체 전체가 unsafe scope 로 간주됩니다.

```rok
unsafe fn copy_two<T>(p: Ptr<T>) -> (T, T) {
    let a = p.read();              // 본체가 unsafe scope: 명시 블록 불필요
    let b = p.add(1).read();
    (a, b)
}
```

### 메서드 receiver 와의 결합

`impl` 의 메서드도 같은 형태를 따릅니다.

```rok
impl Buffer {
    unsafe fn raw_data(self) -> Ptr<u8>;
    unsafe fn raw_data_mut(ref self) -> Ptr<u8>;
}
```

`unsafe` 는 `ref self` / `self` 같은 receiver 표기와 충돌하지 않으며, 함수 헤더 가장 앞자리에 위치합니다.

### `extern "C"` 블록 안

사용자가 선언한 C ABI 함수는 정의가 컴파일러 통제 밖이므로 묵시적으로 모두 `unsafe` 입니다 ([extern 블록](../005-modules/509-extern-block.md)). 명시 표기 (`unsafe fn`) 가 권장되며, 생략하면 자동으로 `unsafe` 처리됩니다.

예외로, toolchain 이 선언하는 함수 (컴파일러가 주입하거나 런타임이 제공하는 `rok_print_*`, `rok_panic_str` 같은 심볼) 는 안전하게 호출할 수 있습니다.

```rok
extern "C" {
    fn malloc(size: usize) -> CPtr<u8>;        // 자동 unsafe 처리
    unsafe fn free(ptr: CPtr<u8>);             // 명시 (권장)
}
```

## 4.15.2. `unsafe { ... }` - 블록 표현식

unsafe 함수를 호출하거나 raw pointer 를 dereference 하는 등의 동작은 `unsafe { ... }` 블록 안에서만 허용됩니다. 블록 자체는 일반 [블록 표현식](../003-expressions/309-block-expressions.md) 과 동일하게 마지막 식의 값을 결과로 갖습니다.

```rok
let n: usize = unsafe { strlen(cs.as_ptr()) };

let v: i32 = unsafe {
    let p = ptr.add(idx);
    p.read()
};

unsafe {
    free(ptr);
}
```

### 블록 안에서 가능한 동작

`unsafe { ... }` 블록 안에서는 다음 동작이 추가로 허용됩니다.

- `unsafe fn` 으로 선언된 함수 / 메서드 호출
- raw pointer (`Ptr<T>`, `ConstPtr<T>`, `ManyPtr<T>`) 의 dereference (`p.read()`, `p.write(v)`)
- [`rok::bare::*`](../../refs-standard-libraries/100-bare/README.md) 모듈의 위험 함수 직접 호출
- [`rok::bare::mlir::__mlir_op`](../../refs-standard-libraries/100-bare/109-mlir.md) MLIR op 직접 호출
- `union` 필드 읽기 (비트 재해석, [union](../002-types/205-union.md)). 필드 쓰기는 `unsafe` 없이 가능합니다.
- 다른 소유자에게 넘긴 뒤의 값 사용 ([stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync---스레드-경계-marker)): 다른 스레드가 동시에 그것을 쓰지 않음을 블록의 작성자가 보장합니다.

이 외의 일반 안전 동작은 unsafe 블록 안에서도 똑같이 작동합니다. `unsafe` 는 *기능을 추가* 하는 것이지 *기존 의미를 바꾸는* 것이 아닙니다.

### scope 의 좁힘

`unsafe` 블록은 가능한 한 좁게 두는 것이 권장됩니다. 함수 전체를 `unsafe { ... }` 로 감싸는 대신, 실제로 위험한 호출 한 줄만 감싸 두면 다른 줄의 안전성이 시각적으로 분명해집니다.

```rok
// 권장: 좁은 unsafe scope
fn safe_wrapper(p: Ptr<i32>) throws -> i32 {
    let validated = try check_ptr(p);
    unsafe { validated.read() }    // 위험 동작만 unsafe
}

// 비권장: 넓은 unsafe scope
fn safe_wrapper(p: Ptr<i32>) throws -> i32 {
    unsafe {
        let validated = try check_ptr(p);
        validated.read()
    }
}
```

## 4.15.3. 호출 규칙

`unsafe fn` 의 호출은 다음 규칙을 따릅니다.

| 호출 자리                | `unsafe fn` 호출 가능? |
|--------------------------|------------------------|
| 일반 함수 본체           | ✗ (컴파일 에러)        |
| `unsafe fn` 본체         | ✓ (자동 unsafe scope)  |
| `unsafe { ... }` 블록    | ✓                      |
| `extern "C"` 블록 선언자 | ✓ (자동 unsafe)        |

규칙을 어기면 다음과 같이 컴파일 에러가 발생합니다.

```rok
fn safe_caller(p: Ptr<i32>) -> i32 {
    p.read()    // ERROR: call to unsafe fn 'read' outside unsafe block
}
```

해결은 `unsafe { ... }` 로 감싸거나, 호출자 자신을 `unsafe fn` 으로 표시하는 두 가지 길이 있습니다.

### `unsafe fn` 타입

`unsafe fn` 과 C 함수를 값으로 쓰면 그 타입은 `unsafe fn(..) -> R` 입니다 ([4.4](404-function-type.md)). 이 타입의 값을 호출하려면 `unsafe` 가 필요합니다. 안전한 `fn(..) -> R` 값은 `unsafe fn(..) -> R` 자리에 넣을 수 있지만, 반대는 안 됩니다.

```rok
let f: unsafe fn(usize) -> CPtr<u8> = malloc;   // C 함수 값
let p = unsafe { f(16) };
let g: unsafe fn(i32) -> i32 = |x| x + 1;      // OK: safe → unsafe
let h: fn(usize) -> CPtr<u8> = f;              // ERROR: unsafe → safe
```

## 4.15.4. trait 안에서

`trait` 의 메서드도 `unsafe fn` 을 사용할 수 있습니다. `impl` 측은 같은 시그니처로 구현해야 하며, 호출자는 일반 호출 자리와 동일한 규칙을 따릅니다.

```rok
trait RawAccess {
    unsafe fn raw_get(self, idx: usize) -> u8;
}

impl RawAccess for Buffer {
    unsafe fn raw_get(self, idx: usize) -> u8 {
        self.ptr.add(idx).read()
    }
}

let buf: Buffer = ...;
let b = unsafe { buf.raw_get(0) };
```

### `unsafe impl`

`unsafe impl` 은 자동 marker 트레잇 `Send` / `Sync` 를 손으로 구현할 때만 씁니다. 그 타입이 스레드 경계를 넘어도 안전하다는 것을 작성자가 보증한다는 표시입니다. 다른 트레잇에 `unsafe impl` 을 쓰면 컴파일 에러입니다 ([2.9.1](../002-types/209-impl.md#291-두-형태)).

```rok
unsafe impl Send for Handle { }
```

## 4.15.5. 책임 모델

`unsafe` 는 컴파일러가 안전 검증을 *포기* 한다는 뜻이 아닙니다. 다음 두 책임이 호출자에게 이전된다는 뜻입니다.

1. **메모리 안전성**: dangling pointer 나 잘못된 정렬 접근, 데이터 경합 등의 미정의 동작을 일으키지 않을 책임.
2. **사전 조건**: `unsafe fn` 의 문서가 약속한 사용 조건 (예: pointer 가 valid 한 객체를 가리킬 것, 길이가 buffer 안일 것 등) 을 만족시킬 책임.

이 두 책임을 호출자가 받아들인다는 시각적 신호가 곧 `unsafe { ... }` 입니다. 그래서 코드 리뷰 시점에 위험 영역이 한눈에 들어오게 됩니다.

## 4.15.6. 자동 묵시 unsafe 와의 관계

다음 자리는 `unsafe` 명시 없이도 자동으로 unsafe scope 로 간주됩니다.

- [`extern "C"` 블록](../005-modules/509-extern-block.md) 의 함수 선언 본체
- `unsafe fn` 의 본체

이 자동 처리 덕분에 자주 등장하는 위험 영역에서 `unsafe { ... }` 가 잡음처럼 반복되는 일이 줄어듭니다.

## 4.15.7. 다른 언어와의 비교

| 언어    | unsafe 표기 모델                                       |
|---------|--------------------------------------------------------|
| Rust    | `unsafe fn` + `unsafe { ... }` (동형)                  |
| C / C++ | 별도 표기 없음, 모든 코드가 잠재적 unsafe              |
| Swift   | `@unsafe` 어노테이션 / `withUnsafePointer` 함수형 패턴 |
| Go      | `unsafe` 패키지 (별도 import), 블록 표기 없음          |
| Zig     | `@ptrCast` 등 명시 builtin, 블록 표기 없음             |
| **Rok** | **Rust 식 `unsafe fn` + `unsafe { ... }` 채택**        |

Rok은 Rust 의 모델을 그대로 따릅니다. 위험 영역의 시각적 분리가 단일 키워드로 일관되게 표시되며, 이는 코드 리뷰와 정적 분석 모두에 유리합니다.
