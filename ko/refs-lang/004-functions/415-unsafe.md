# 4.15. `unsafe`: 함수 수식어와 블록 표현식

`unsafe` 키워드는 컴파일러가 안전성을 보증하지 않는 영역을 눈에 보이게 가둡니다. 쓰는 형태는 **함수 / 메서드 선언의 수식어**와 **블록 표현식** 두 가지이며, 둘은 한 쌍으로 동작합니다. `unsafe fn`으로 선언한 함수는 `unsafe { ... }` 블록 안에서 호출해야 합니다.

```
unsafe_fn        ::= "unsafe" fn_decl
unsafe_block     ::= "unsafe" block_expr
```

이 모델은 Rust의 `unsafe`와 같은 구조입니다. 목적은 위험 영역을 코드에 명시적으로 드러내는 것입니다.

## 4.15.1. `unsafe fn`: 함수 수식어

함수나 메서드 선언 앞에 `unsafe`를 두면 그 함수의 본체가 안전 추상의 보장 밖에 있다는 뜻입니다. 사전 조건(precondition)을 만족할 책임이 호출자에게 있다는 사실을 시그니처가 바로 알려 줍니다.

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

`unsafe fn`의 본체 안에서는 따로 `unsafe { ... }` 블록이 없어도 unsafe 호출을 할 수 있습니다. 함수 시그니처가 이미 unsafe를 선언했으므로 컴파일러는 본체 전체를 unsafe 스코프로 봅니다.

```rok
unsafe fn copy_two<T>(p: Ptr<T>) -> (T, T) {
    let a = p.read();              // 본체가 unsafe scope - 명시 블록 불필요
    let b = p.add(1).read();
    (a, b)
}
```

### 메서드 수신자와 결합

`impl`의 메서드도 같은 형태를 따릅니다.

```rok
impl Buffer {
    unsafe fn raw_data(self) -> Ptr<u8>;
    unsafe fn raw_data_mut(ref self) -> Ptr<u8>;
}
```

`unsafe`는 `ref self` / `self` 같은 receiver 표기와 충돌하지 않습니다. 이 키워드는 함수 헤더 맨 앞에 옵니다.

### `extern "C"` 블록 안

사용자가 선언한 C ABI 함수는 정의가 컴파일러의 통제 밖에 있으므로 모두 묵시적으로 `unsafe`입니다([extern 블록](../005-modules/509-extern-block.md)). `unsafe fn`으로 명시하기를 권장합니다. 생략하면 컴파일러가 자동으로 `unsafe`로 처리합니다.

예외로, toolchain이 선언하는 함수(컴파일러가 주입하거나 런타임이 제공하는 `rok_print_*`, `rok_panic_str` 같은 심볼)는 안전하게 호출할 수 있습니다.

```rok
extern "C" {
    fn malloc(size: usize) -> CPtr<u8>;        // 자동 unsafe 처리
    unsafe fn free(ptr: CPtr<u8>);             // 명시 (권장)
}
```

## 4.15.2. `unsafe { ... }`: 블록 표현식

unsafe 함수 호출, 원시 포인터 역참조 같은 동작은 `unsafe { ... }` 블록 안에서만 할 수 있습니다. 블록의 결과는 일반 [블록 표현식](../003-expressions/309-block-expressions.md)과 똑같이 마지막 식의 값입니다.

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

`unsafe { ... }` 블록 안에서는 다음 동작을 추가로 할 수 있습니다.

- `unsafe fn`으로 선언한 함수 / 메서드 호출
- 원시 포인터(`Ptr<T>`, `ConstPtr<T>`, `ManyPtr<T>`) 역참조(`p.read()`, `p.write(v)`)
- [`rok::bare::*`](../../refs-standard-libraries/100-bare/README.md) 모듈의 위험 함수 직접 호출
- [`rok::bare::mlir::__mlir_op`](../../refs-standard-libraries/100-bare/109-mlir.md) MLIR op 직접 호출
- `union` 필드 읽기(비트 재해석, [union](../002-types/205-union.md)). 필드 쓰기는 `unsafe` 없이 할 수 있습니다.
- 다른 소유자에게 넘긴 값을 그 뒤에 사용([stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync-스레드-경계-마커)). 다른 스레드가 동시에 그 값을 쓰지 않는다는 점은 블록 작성자가 보장합니다.

그 밖의 일반 안전 동작은 unsafe 블록 안에서도 똑같이 동작합니다. `unsafe`는 '기능을 더할' 뿐 '기존 의미를 바꾸지' 않습니다.

### 범위 좁히기

`unsafe` 블록은 되도록 좁게 두기를 권장합니다. 함수 전체를 `unsafe { ... }`로 감싸지 말고 실제로 위험한 호출 한 줄만 감쌉니다. 그러면 나머지 줄이 안전하다는 점이 눈에 분명해집니다.

```rok
// 권장 - 좁은 unsafe scope
fn safe_wrapper(p: Ptr<i32>) throws -> i32 {
    let validated = try check_ptr(p);
    unsafe { validated.read() }    // 위험 동작만 unsafe
}

// 비권장 - 넓은 unsafe scope
fn safe_wrapper(p: Ptr<i32>) throws -> i32 {
    unsafe {
        let validated = try check_ptr(p);
        validated.read()
    }
}
```

## 4.15.3. 호출 규칙

`unsafe fn`의 호출은 다음 규칙을 따릅니다.

| 호출 자리                | `unsafe fn` 호출       |
|--------------------------|------------------------|
| 일반 함수 본체           | ✗ (컴파일 에러)        |
| `unsafe fn` 본체         | ✓ (자동 unsafe 스코프) |
| `unsafe { ... }` 블록    | ✓                      |
| `extern "C"` 블록 선언자 | ✓ (자동 unsafe)        |

규칙을 어기면 다음과 같이 컴파일 에러입니다.

```rok
fn safe_caller(p: Ptr<i32>) -> i32 {
    p.read()    // ERROR - call to unsafe fn 'read' outside unsafe block
}
```

해결 방법은 두 가지입니다. `unsafe { ... }`로 감싸거나, 호출자 자신을 `unsafe fn`으로 표시합니다.

### `unsafe fn` 타입

`unsafe fn`과 C 함수를 값으로 쓰면 그 타입은 `unsafe fn(..) -> R`입니다([4.4](404-function-type.md)). 이 타입의 값을 호출하려면 `unsafe`가 필요합니다. 안전한 `fn(..) -> R` 값은 `unsafe fn(..) -> R` 자리에 넣을 수 있지만, 반대로는 넣을 수 없습니다.

```rok
let f: unsafe fn(usize) -> CPtr<u8> = malloc;   // C 함수 값
let p = unsafe { f(16) };
let g: unsafe fn(i32) -> i32 = |x| x + 1;      // OK - safe → unsafe
let h: fn(usize) -> CPtr<u8> = f;              // ERROR - unsafe → safe
```

## 4.15.4. 트레잇 안에서

`trait`의 메서드도 `unsafe fn`일 수 있습니다. `impl` 쪽은 같은 시그니처로 구현해야 합니다. 호출자는 일반 호출 자리와 같은 규칙을 따릅니다.

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

`unsafe impl`은 자동 마커 트레잇 `Send` / `Sync`를 직접 구현할 때만 씁니다. 그 타입이 스레드 경계를 넘어도 안전하다고 작성자가 보증한다는 표시입니다. 다른 트레잇에 `unsafe impl`을 쓰면 컴파일 에러입니다([2.9.1](../002-types/209-impl.md#291-두-형태)).

```rok
unsafe impl Send for Handle { }
```

## 4.15.5. 책임 모델

`unsafe`는 컴파일러가 안전 검사를 '포기'한다는 뜻이 아닙니다. 다음 두 책임이 호출자에게 넘어간다는 뜻입니다.

1. **메모리 안전성**: 댕글링 포인터, 잘못된 정렬 접근, 데이터 경합 같은 미정의 동작을 일으키지 않을 책임
2. **사전 조건**: `unsafe fn`의 문서가 약속한 사용 조건을 만족할 책임(예: 포인터가 유효한 객체를 가리킬 것, 길이가 버퍼 안일 것)

`unsafe { ... }`는 호출자가 이 두 책임을 받아들인다는 눈에 보이는 표시입니다. 그래서 코드 리뷰 때 위험 영역이 한눈에 들어옵니다.

## 4.15.6. 묵시적 unsafe와 관계

컴파일러는 다음 자리를 `unsafe`를 적지 않아도 자동으로 unsafe 스코프로 봅니다.

- [`extern "C"` 블록](../005-modules/509-extern-block.md)의 함수 선언 본체
- `unsafe fn`의 본체

이 자동 처리 덕분에 위험 영역이 자주 나오는 곳에서 `unsafe { ... }`를 거듭 적지 않아도 됩니다.

## 4.15.7. 다른 언어와 비교

| 언어    | unsafe 표기 모델                                       |
|---------|--------------------------------------------------------|
| Rust    | `unsafe fn` + `unsafe { ... }` (동형)                  |
| C / C++ | 별도 표기 없음, 모든 코드가 잠재적으로 unsafe          |
| Swift   | `@unsafe` 어노테이션 / `withUnsafePointer` 함수형 패턴 |
| Go      | `unsafe` 패키지 (별도 import), 블록 표기 없음          |
| Zig     | `@ptrCast` 등 명시 builtin, 블록 표기 없음             |
| **Rok** | **Rust 식 `unsafe fn` + `unsafe { ... }` 채택**        |

Rok은 Rust의 모델을 그대로 따릅니다. 키워드 하나로 위험 영역을 일관되게 구분해 보여 주므로 코드 리뷰와 정적 분석 모두에 유리합니다.
