# 1.2. cast 연산 - 명시 named cast

pointer와 type 사이의 변환은 **named function**으로만 존재합니다. `cast` 모듈은 없습니다. `ptr_cast`, `align_cast`, `align_cast_unchecked`, `const_cast`는 `rok::bare::ptr`에, `reinterpret`와 `transmute_unchecked`는 `rok::bare::mem`에 정의되며, `rok::bare` 루트가 여섯 개 모두를 다시 내보냅니다(`rok::bare::ptr_cast`). C와 Rust의 `as` cast가 위험한 까닭은 변환이 조용히(silent) 일어나기 때문입니다. Rok은 이 함정을 피하려고 모든 cast에 **명시적인 이름**을 요구합니다.

## 1.2.1. cast 함수 4 종

기본 함수 4개는 다음과 같습니다.

| 함수                                     | 의미                                   | 위험도                  |
|------------------------------------------|----------------------------------------|-------------------------|
| `ptr_cast<U>(p: Ptr<T>)`                 | type만 변경 (provenance/address 보존) | alignment는 호출자 책임 |
| `align_cast<T>(p: Ptr<u8>)`              | alignment up-cast                      | 검증 (어긋나면 `none`)  |
| `const_cast<T>(p: ConstPtr<T>)`          | const 제거                             | 매우 높음 (호출자 보장) |
| `reinterpret<A: Pod, B: Pod>(a: A) -> B` | bit-level cast (`Pod`만)              | 낮음 (Pod marker 보장)  |

더 깊은 우회가 필요할 때 쓰는 escape hatch도 하나 있습니다.

| 함수                                  | 의미                                     | 위험도 |
|---------------------------------------|------------------------------------------|--------|
| `transmute_unchecked<A,B>(a: A) -> B` | **Pod 검증 없이** bit cast (호출자 책임) | 최고   |

컴파일러가 pointer에 쓴 `as` 키워드를 거부하므로, 사용자는 named cast를 쓸 수밖에 없습니다.

## 1.2.2. `ptr_cast<U>` - pointer type 변환

```rok
fn ptr_cast<U, T>(p: Ptr<T>) -> Ptr<U>;
```

이 함수는 address와 provenance를 그대로 두고 type tag만 바꿉니다. 따라서 **alignment 책임은 호출자**에게 있습니다. 인자로는 `Ptr`만 받습니다(Rok에는 오버로딩이 없습니다). `ConstPtr`와 `ManyPtr`는 각자의 `cast` 메서드로 같은 변환을 합니다 ([1.1](101-ptr.md)): `p.cast::<U>()`.

```rok
let p: Ptr<u32> = ...;        // 4-byte aligned
let q: Ptr<u8> = rok::bare::ptr_cast::<u8>(p);    // 안전 (down-align)
let r: Ptr<u64> = rok::bare::ptr_cast::<u64>(p);  // 위험: u64 은 8-byte 필요
                                                 // r.read() 시 SIGBUS 가능 (ARM)
```

어떤 빌드에서도 **alignment 검증**을 넣지 않습니다. 정렬이 어긋난 결과로 읽거나 쓰면 UB입니다. 먼저 `align_cast`(1.2.3)나 `is_aligned`(1.1.2)로 확인합니다.

### 안전 cast - type-equivalent

layout이 동일한 두 struct 사이에서는 안전하게 변환할 수 있습니다.

```rok
struct A { x: u32 }
struct B { x: u32 }       // A 와 layout 동일

let pa: Ptr<A> = ...;
let pb: Ptr<B> = rok::bare::ptr_cast::<B>(pa);    // layout OK 면 안전
```

다만 layout이 정말 같은지는 컴파일러가 검증하지 않으므로, 호출자가 이를 보장해야 합니다.

## 1.2.3. `align_cast<T>` - alignment 검증

```rok
fn align_cast<T>(p: Ptr<u8>) -> Ptr<T> | none;
```

`Ptr<u8>`(1-byte align)을 더 큰 alignment의 `Ptr<T>`로 바꿀 때는 실제 정렬을 **검증한 뒤에만** 변환합니다.

```rok
let bytes: ManyPtr<u8> = ...;
let p: Ptr<u8> = bytes.add(offset).as_one();

if let aligned: Ptr<u32> = rok::bare::align_cast::<u32>(p) {
    let v = unsafe { aligned.read() };       // 안전 (검증됨)
} else {
    panic("not 4-byte aligned");
}
```

`align_cast`는 `Result`가 아니라 `T | none`을 반환합니다. 둘 가운데 후자가 더 가볍기 때문입니다.

### unchecked variant

```rok
fn align_cast_unchecked<T>(p: Ptr<u8>) -> Ptr<T>;     // 검증 없이
```

호출 직전에 정렬을 이미 검증한 곳에서만 사용합니다. 이 변종은 debug 빌드에서도 검증하지 않습니다.

## 1.2.4. `const_cast<T>` - const 제거

```rok
fn const_cast<T>(p: ConstPtr<T>) -> Ptr<T>;
```

const를 벗겨내는 함수입니다. 위험한 이유는 분명합니다. 원본이 read-only 영역(immortal const)에 있으면, 거기에 쓰는 순간 segfault가 납니다.

```rok
use rok::bare::ptr::{Ptr, ConstPtr, const_cast};

let s: str = "hello";              // immortal const
let p: ConstPtr<u8> = s.as_ptr();
let m: Ptr<u8> = const_cast(p);
unsafe { m.write(72u8) };          // 'H': SEGFAULT (immortal const 쓰기)
```

C의 `const_cast<T*>`와 정확히 동형입니다. read-only API를 mutability가 필요한 곳에 임시로 wrapping하는 용도로만 쓰며, 일반 코드에서는 **사용하면 안 됩니다**.

## 1.2.5. `reinterpret<A: Pod, B: Pod>` - bit cast (안전)

```rok
trait Pod { /* marker: Plain Old Data */ }

fn reinterpret<A: Pod, B: Pod>(a: A) -> B;
```

`Pod`(Plain Old Data) marker가 있는 타입에서는 **safe transmute**가 보장됩니다. 근거는 다음과 같습니다.

- `Pod` 타입은 모든 bit pattern이 유효하므로 uninit이 노출될 일이 없습니다.
- 변환은 size가 같을 때만 허용됩니다. `where size_of` bound는 컴파일 타임 형태가 없으므로, size 조건은 `reinterpret`가 실행될 때 검사합니다(다르면 panic).
- 값 변환이라 alignment가 자연히 보장됩니다.

```rok
let f: f32 = 3.14;
let bits: u32 = rok::bare::reinterpret(f);     // OK: 둘 다 Pod, 4 byte
let f2: f32 = rok::bare::reinterpret(bits);    // round-trip OK
```

### `Pod` 자동 derive

표준 라이브러리가 `Pod` marker를 자동으로 붙이는 타입은 다음과 같습니다.

```rok
impl Pod for i8, i16, i32, i64, u8, u16, u32, u64, usize, isize { }
impl Pod for f32, f64 { }
impl Pod for bool { }
impl Pod for char, c_char { }
impl<T: Pod, const N: usize> Pod for [T; N] { }
impl<A: Pod> Pod for (A,) { }
impl<A: Pod, B: Pod> Pod for (A, B) { }
impl<A: Pod, B: Pod, C: Pod> Pod for (A, B, C) { }
impl<A: Pod, B: Pod, C: Pod, D: Pod> Pod for (A, B, C, D) { }
```

`bool`과 `char`는 모든 bit pattern이 유효한 값이 아닌데도(`bool`은 0 또는 1, `char`는 Unicode scalar) 이 목록에 있습니다. `reinterpret`는 bit를 검사하지 않으므로, 다른 bit를 `bool`이나 `char`로 reinterpret하면 유효하지 않은 값이 되고, 이는 UB입니다. 호출자가 먼저 검사합니다.

반면 다음은 자동으로 붙지 않습니다.

- `Ptr<T>`는 provenance 정보가 따로 있어 bit만으로는 표현할 수 없습니다.
- 사용자 struct는 `#[derive(Pod)]`로 명시해야 합니다.

```rok
#[derive(Pod)]
#[repr(C)]
struct Vec3 { x: f32; y: f32; z: f32; }

let v: Vec3 = Vec3 { x: 1.0, y: 2.0, z: 3.0 };
let bytes: [u8; 12] = rok::bare::reinterpret(v);
let v2: Vec3 = rok::bare::reinterpret(bytes);
```

이 설계는 [Project Safe Transmute (Rust RFC 2835)](https://rust-lang.github.io/rfcs/2835-project-safe-transmute.html) 모델과 [bytemuck crate](https://docs.rs/bytemuck) 패턴을 따릅니다.

## 1.2.6. `transmute_unchecked<A, B>` - escape hatch

```rok
unsafe fn transmute_unchecked<A, B>(a: A) -> B;   // size check 도 안 함
```

이 함수는 `Pod` 검증조차 **없이** bit를 그대로 옮깁니다. 이름대로 가장 위험한 cast이므로 `unsafe fn`입니다. uninit 노출, size 불일치, alignment 문제, 안에 든 GC 참조는 모두 호출자 책임입니다. size 불일치는 거부하지 않습니다. 값은 두 타입 중 큰 쪽 크기의, 0으로 채운 scratch block을 거치므로 원본 크기를 넘는 byte는 0으로 읽힙니다.

표준 라이브러리 내부의 가장 **낮은 layer**에서만 씁니다. `Pod` impl 자체를 작성할 때처럼 다른 도구로는 표현할 수 없는 경우뿐입니다. 일반 사용자 코드에서는 절대 쓰면 안 됩니다.

```rok
// 표준 라이브러리 내부 예
impl Pod for SomeNewtype {
    // empty marker
}

let s: SomeNewtype = ...;
let bits: u64 = unsafe { rok::bare::transmute_unchecked(s) };   // SomeNewtype 이 size 8 보장한 코드
```

## 1.2.7. `as` cast 의 적용 자리

`as`는 수치 변환에만 허용되며, 그중에서도 lossless이거나 saturate / wrap이 명시된 경우뿐입니다.

| 표현               | 결과                                                   |
|--------------------|--------------------------------------------------------|
| `i32 as i64`       | OK (lossless widening)                                 |
| `i64 as i32`       | **컴파일 에러**: `try_into` / `saturating_into`        |
| `f32 as f64`       | OK (lossless)                                          |
| `i32 as f32`       | **컴파일 에러**: `as f32` 정밀도 손실 가능             |
| `u32 as i32`       | **컴파일 에러**: bit reinterpret (`reinterpret` 사용)  |
| `Ptr<T> as Ptr<U>` | **컴파일 에러**: `rok::bare::ptr_cast::<U>`             |
| `Ptr<T> as usize`  | **컴파일 에러**: `.addr()` 또는 `expose_provenance`    |
| `usize as Ptr<T>`  | **컴파일 에러**: `from_exposed_provenance`             |

전체 변환 매트릭스는 [2.11 conversion](../200-core/211-conversion.md)을 참고하세요.

## 1.2.8. 정책 요약

이 장의 cast 정책은 네 가지로 정리됩니다.

- `as` cast는 **lossless 수치 변환**만 허용하고, pointer와 bit reinterpret는 거부합니다.
- `rok::bare::ptr_cast` / `align_cast` / `const_cast` / `reinterpret` 네 함수가 명시 cast의 통로입니다.
- `Pod` marker trait이 bit reinterpret의 안전 조건입니다.
- `transmute_unchecked`는 escape hatch로만 노출됩니다.

이 정책으로 Rok은 C의 번거로운 형 변환과 Rust의 `as` 함정을 함께 피합니다.
