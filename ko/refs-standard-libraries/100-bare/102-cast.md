# 1.2. cast 연산 — 명시 named cast

`rok::bare::cast` 모듈은 pointer 와 type 사이의 변환을 오직 **named function** 형태로만 노출합니다. C 와 Rust 의 `as` cast 가 가장 위험한 까닭은 변환이 조용히(silent) 일어난다는 점이며, rok 은 이 함정을 피하기 위해 모든 cast 를 **명시적인 이름** 으로 강제합니다.

## 1.2.1. cast 함수 4 종

기본 4 함수는 다음과 같습니다.

| 함수                                     | 의미                                   | 위험도                  |
|------------------------------------------|----------------------------------------|-------------------------|
| `ptr_cast<U>(p: Ptr<T>)`                 | type 만 변경 (provenance/address 보존) | alignment 검증          |
| `align_cast<T>(p: Ptr<u8>)`              | alignment up-cast                      | 검증 (debug)            |
| `const_cast<T>(p: ConstPtr<T>)`          | const 제거                             | 매우 높음 (호출자 보장) |
| `reinterpret<A: Pod, B: Pod>(a: A) -> B` | bit-level cast (`Pod` 만)              | 낮음 (Pod marker 보장)  |

여기에 더해, 더 깊은 우회로가 필요할 때 쓰는 escape hatch 도 하나 있습니다.

| 함수                                  | 의미                                     | 위험도 |
|---------------------------------------|------------------------------------------|--------|
| `transmute_unchecked<A,B>(a: A) -> B` | **Pod 검증 없이** bit cast (호출자 책임) | 최고   |

`as` 키워드는 pointer 자리에서 컴파일러가 거부하므로, 사용자는 자연히 named cast 를 쓸 수밖에 없습니다.

## 1.2.2. `ptr_cast<U>` — pointer type 변환

```rok
fn ptr_cast<U, T>(p: Ptr<T>) -> Ptr<U>;
fn ptr_cast<U, T>(p: ConstPtr<T>) -> ConstPtr<U>;
fn ptr_cast<U, T>(p: ManyPtr<T>) -> ManyPtr<U>;
```

이 함수는 address 와 provenance 를 그대로 보존하면서 type tag 만 바꾼다. 따라서 **alignment 책임은 호출자** 가 집니다.

```rok
let p: Ptr<u32> = ...;        // 4-byte aligned
let q: Ptr<u8> = rok::bare::ptr_cast::<u8>(p);    // 안전 (down-align)
let r: Ptr<u64> = rok::bare::ptr_cast::<u64>(p);  // 위험 — u64 은 8-byte 필요
                                                 // r.read() 시 SIGBUS 가능 (ARM)
```

debug 빌드에서는 **alignment 검증** 이 들어가 panic 으로 이어지지만, release 빌드에서는 검증이 사라져 그대로 UB 가 됩니다.

### 안전 cast — type-equivalent

layout 이 동일한 두 struct 사이에서는 안전하게 변환할 수 있습니다.

```rok
struct A { x: u32 }
struct B { x: u32 }       // A 와 layout 동일

let pa: Ptr<A> = ...;
let pb: Ptr<B> = rok::bare::ptr_cast::<B>(pa);    // layout OK 면 안전
```

다만 layout 이 정말 같은지는 컴파일러가 검증하지 않으므로, 이 보장은 호출자가 직접 떠안아야 합니다.

## 1.2.3. `align_cast<T>` — alignment 검증

```rok
fn align_cast<T>(p: Ptr<u8>) -> Ptr<T> | none;
```

`Ptr<u8>` (1-byte align) 을 더 큰 alignment 의 `Ptr<T>` 로 바꿀 때, 실제 정렬을 **검증한 뒤에야** 변환합니다.

```rok
let bytes: ManyPtr<u8> = ...;
let p: Ptr<u8> = bytes.add(offset).as_one();

if let aligned: Ptr<u32> = rok::bare::align_cast::<u32>(p) {
    let v = unsafe { aligned.read() };       // 안전 (검증됨)
} else {
    panic("not 4-byte aligned");
}
```

`align_cast` 의 반환은 `Result` 가 아니라 `T | none` 입니다. 두 형태 가운데 후자가 더 가볍기 때문입니다.

### unchecked variant

```rok
fn align_cast_unchecked<T>(p: Ptr<u8>) -> Ptr<T>;     // 검증 없이
```

호출 직전에 이미 정렬을 검증해 둔 자리에서만 사용합니다. 이 변종은 debug 빌드에서도 검증을 수행하지 않습니다.

## 1.2.4. `const_cast<T>` — const 제거

```rok
fn const_cast<T>(p: ConstPtr<T>) -> Ptr<T>;
```

말 그대로 const 를 벗겨내는 함수입니다. 매우 위험한 이유는 명확합니다. 원본이 read-only 영역(immortal const) 일 경우, 그 자리에 write 하면 곧장 segfault 가 납니다.

```rok
let s: str = "hello";              // immortal const
let p: ConstPtr<u8> = s.as_ptr();
let m: Ptr<u8> = rok::bare::const_cast(p);
unsafe { m.write(b'H') };          // SEGFAULT — immortal const 쓰기
```

C 의 `const_cast<T*>` 와 정확히 동형입니다. 사용처는 read-only API 를 임시로 mutability 가 필요한 자리에 wrapping 하는 정도로 한정되며, 일반 코드에서는 **사용하지 말 것**.

## 1.2.5. `reinterpret<A: Pod, B: Pod>` — bit cast (안전)

```rok
trait Pod { /* marker — Plain Old Data */ }

fn reinterpret<A: Pod, B: Pod>(a: A) -> B
where
    sizeof::<A>() == sizeof::<B>();
```

`Pod` (Plain Old Data) marker 가 있는 타입에 한해서는 **safe transmute** 가 보장됩니다. 그 근거는 다음과 같습니다.

- `Pod` 타입은 모든 bit pattern 이 유효하므로 uninit 이 노출될 일이 없습니다.
- 변환은 같은 size 끼리만 허용됩니다.
- 값 변환이라 alignment 가 자연히 보장됩니다.

```rok
let f: f32 = 3.14;
let bits: u32 = rok::bare::reinterpret(f);     // OK — 둘 다 Pod, 4 byte
let f2: f32 = rok::bare::reinterpret(bits);    // round-trip OK
```

### `Pod` 자동 derive

표준 라이브러리가 자동으로 `Pod` 을 붙여 주는 타입은 다음과 같습니다.

```rok
impl Pod for i8, i16, i32, i64, u8, u16, u32, u64, usize, isize { }
impl Pod for f32, f64 { }
impl Pod for bool { }
impl Pod for char, c_char { }
impl<T: Pod, const N: usize> Pod for [T; N] { }
impl<T: Pod, U: Pod> Pod for (T, U) { }
// Tuple 더 큰 arity 는 macro / 자동
```

반면 다음은 자동으로 붙지 않습니다 (uninit 노출 가능성 때문입니다).

- `bool` 은 0/1 만 valid 이지만, 실제로는 niche 가 invalid bit 를 잡아 주므로 자동 적용됩니다.
- `Ptr<T>` 는 provenance 정보가 따로 있어 bit 만으로는 표현되지 않습니다.
- 사용자 struct 는 `#[derive(Pod)]` 로 명시해야 합니다.

```rok
#[derive(Pod)]
#[repr(C)]
struct Vec3 { x: f32; y: f32; z: f32; }

let v: Vec3 = Vec3 { x: 1.0, y: 2.0, z: 3.0 };
let bytes: [u8; 12] = rok::bare::reinterpret(v);
let v2: Vec3 = rok::bare::reinterpret(bytes);
```

이 설계는 [Project Safe Transmute (Rust RFC 2835)](https://rust-lang.github.io/rfcs/2835-project-safe-transmute.html) 모델과 [bytemuck crate](https://docs.rs/bytemuck) 패턴을 그대로 따릅니다.

## 1.2.6. `transmute_unchecked<A, B>` — escape hatch

```rok
fn transmute_unchecked<A, B>(a: A) -> B;       // size check 도 안 함
```

이 함수는 `Pod` 검증조차 **없이** bit 를 그대로 옮깁니다. 이름 그대로 가장 위험한 cast 로, uninit 노출, size 불일치, alignment 문제 모두 호출자 책임입니다.

쓰임새는 표준 라이브러리 내부의 가장 **낮은 layer** 에 한정됩니다. 예컨대 `Pod` impl 자체를 작성할 때처럼 다른 도구로는 도저히 표현할 수 없는 경우만 해당합니다. 일반 사용자 코드에서는 절대 쓰지 말 것.

```rok
// 표준 라이브러리 내부 예
impl Pod for SomeNewtype {
    // empty marker
}

let s: SomeNewtype = ...;
let bits: u64 = rok::bare::transmute_unchecked(s);   // SomeNewtype 이 size 8 보장한 코드
```

## 1.2.7. `as` cast 의 적용 자리

`as` 는 오직 수치 변환 자리에서만 허용되며, 그 안에서도 lossless 이거나 saturate / wrap 이 명시된 경우로 한정됩니다.

| 표현               | 결과                                                   |
|--------------------|--------------------------------------------------------|
| `i32 as i64`       | OK (lossless widening)                                 |
| `i64 as i32`       | **컴파일 에러** — `try_into` / `saturating_into`       |
| `f32 as f64`       | OK (lossless)                                          |
| `i32 as f32`       | **컴파일 에러** — `as f32` 정밀도 손실 가능            |
| `u32 as i32`       | **컴파일 에러** — bit reinterpret (`reinterpret` 사용) |
| `Ptr<T> as Ptr<U>` | **컴파일 에러** — `rok::bare::ptr_cast::<U>`            |
| `Ptr<T> as usize`  | **컴파일 에러** — `.addr()` 또는 `expose_provenance`   |
| `usize as Ptr<T>`  | **컴파일 에러** — `from_exposed_provenance`            |

자세한 변환 매트릭스는 [2.11 conversion](../200-core/211-conversion.md) 을 참고하라.

## 1.2.8. 정책 요약

본 장의 cast 정책은 다음 네 가지로 정리됩니다.

- `as` cast 는 **수치 lossless** 만 허용하고, pointer 와 bit reinterpret 은 거부합니다.
- `rok::bare::ptr_cast` / `align_cast` / `const_cast` / `reinterpret` 네 함수가 명시 cast 의 통로입니다.
- `Pod` marker trait 이 bit reinterpret 의 안전 조건을 제공합니다.
- `transmute_unchecked` 는 escape hatch 로만 노출됩니다.

이 정책 덕분에 rok 은 C 의 번거로운 형 변환과 Rust 의 `as` 함정을 동시에 피해 갈 수 있습니다.
