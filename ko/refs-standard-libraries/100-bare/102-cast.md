# 1.2. 캐스트 연산: 이름 있는 명시적 캐스트

`rok::bare::cast` 모듈은 포인터와 타입 사이의 변환을 **이름 붙은 함수**로만 노출합니다. C와 Rust의 `as` 캐스트가 위험한 까닭은 변환이 조용히 일어나기 때문입니다. Rok은 이 함정을 피하려고 모든 캐스트에 **명시적인 이름**을 요구합니다.

## 1.2.1. 캐스트 함수 4종

기본 함수 4개는 다음과 같습니다.

| 함수                                     | 의미                                   | 위험도                  |
|------------------------------------------|----------------------------------------|-------------------------|
| `ptr_cast<U>(p: Ptr<T>)`                 | 타입만 변경 (provenance/주소 보존)    | 정렬 검사               |
| `align_cast<T>(p: Ptr<u8>)`              | 정렬 상향 캐스트                       | 검사 (debug)            |
| `const_cast<T>(p: ConstPtr<T>)`          | const 제거                             | 매우 높음 (호출자 보장) |
| `reinterpret<A: Pod, B: Pod>(a: A) -> B` | 비트 수준 캐스트 (`Pod`만)            | 낮음 (Pod marker 보장)  |

더 깊이 우회해야 할 때 쓰는 탈출구(escape hatch)도 하나 있습니다.

| 함수                                  | 의미                                     | 위험도 |
|---------------------------------------|------------------------------------------|--------|
| `transmute_unchecked<A,B>(a: A) -> B` | **Pod 검증 없이** 비트 캐스트 (호출자 책임) | 최고   |

컴파일러는 포인터에 쓴 `as` 키워드를 거부합니다. 그래서 사용자는 이름 붙은 캐스트를 쓸 수밖에 없습니다.

## 1.2.2. `ptr_cast<U>`: 포인터 타입 변환

```rok
fn ptr_cast<U, T>(p: Ptr<T>) -> Ptr<U>;
fn ptr_cast<U, T>(p: ConstPtr<T>) -> ConstPtr<U>;
fn ptr_cast<U, T>(p: ManyPtr<T>) -> ManyPtr<U>;
```

이 함수는 주소와 provenance를 그대로 두고 타입 태그만 바꿉니다. 따라서 **정렬 책임은 호출자**에게 있습니다.

```rok
let p: Ptr<u32> = ...;        // 4-byte aligned
let q: Ptr<u8> = rok::bare::ptr_cast::<u8>(p);    // 안전 (down-align)
let r: Ptr<u64> = rok::bare::ptr_cast::<u64>(p);  // 위험 - u64 은 8-byte 필요
                                                 // r.read() 시 SIGBUS 가능 (ARM)
```

debug 빌드에서는 **정렬 검사**가 들어가 panic으로 이어집니다. release 빌드에서는 검사가 빠지므로 그대로 UB입니다.

### 안전한 캐스트: 타입 동등

레이아웃이 같은 두 struct 사이에서는 안전하게 변환할 수 있습니다.

```rok
struct A { x: u32 }
struct B { x: u32 }       // A 와 layout 동일

let pa: Ptr<A> = ...;
let pb: Ptr<B> = rok::bare::ptr_cast::<B>(pa);    // layout OK 면 안전
```

다만 컴파일러는 레이아웃이 정말 같은지 검증하지 않습니다. 호출자가 보장해야 합니다.

## 1.2.3. `align_cast<T>`: 정렬 검증

```rok
fn align_cast<T>(p: Ptr<u8>) -> Ptr<T> | none;
```

`Ptr<u8>`(1바이트 정렬)을 정렬이 더 큰 `Ptr<T>`로 바꿀 때는 실제 정렬을 **검증한 뒤에만** 변환합니다.

```rok
let bytes: ManyPtr<u8> = ...;
let p: Ptr<u8> = bytes.add(offset).as_one();

if let aligned: Ptr<u32> = rok::bare::align_cast::<u32>(p) {
    let v = unsafe { aligned.read() };       // 안전 (검증됨)
} else {
    panic("not 4-byte aligned");
}
```

`align_cast`는 `Result`가 아니라 `T | none`을 반환합니다. 둘 가운데 뒤쪽이 더 가볍기 때문입니다.

### unchecked 변형

```rok
fn align_cast_unchecked<T>(p: Ptr<u8>) -> Ptr<T>;     // 검증 없이
```

호출 직전에 정렬을 이미 검증한 곳에서만 씁니다. 이 변종은 디버그 빌드에서도 검증하지 않습니다.

## 1.2.4. `const_cast<T>`: const 제거

```rok
fn const_cast<T>(p: ConstPtr<T>) -> Ptr<T>;
```

const를 벗겨 내는 함수입니다. 위험한 이유는 분명합니다. 원본이 읽기 전용 영역(immortal const)에 있으면 거기에 쓰는 순간 segfault가 납니다.

```rok
let s: str = "hello";              // immortal const
let p: ConstPtr<u8> = s.as_ptr();
let m: Ptr<u8> = rok::bare::const_cast(p);
unsafe { m.write(c'H') };          // SEGFAULT - immortal const 쓰기
```

C의 `const_cast<T*>`와 정확히 같은 모델입니다. 읽기 전용 API를 가변성이 필요한 곳에 임시로 감싸는 용도로만 씁니다. 일반 코드에서는 **쓰면 안 됩니다**.

## 1.2.5. `reinterpret<A: Pod, B: Pod>`: 비트 캐스트(안전)

```rok
trait Pod { /* marker - Plain Old Data */ }

fn reinterpret<A: Pod, B: Pod>(a: A) -> B;
```

`Pod`(Plain Old Data) marker가 있는 타입은 **안전한 transmute**가 보장됩니다. 근거는 다음과 같습니다.

- `Pod` 타입은 모든 비트 패턴이 유효하므로 초기화되지 않은 메모리가 노출될 일이 없습니다.
- 변환은 크기가 같을 때만 할 수 있습니다. `where size_of` 바운드는 컴파일 타임 형태가 없으므로 크기 조건은 `reinterpret`가 실행될 때 검사합니다. 크기가 다르면 panic이 일어납니다.
- 값 변환이라 정렬이 자연히 보장됩니다.

```rok
let f: f32 = 3.14;
let bits: u32 = rok::bare::reinterpret(f);     // OK - 둘 다 Pod, 4 byte
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
impl<T: Pod, U: Pod> Pod for (T, U) { }
// Tuple 더 큰 arity 는 macro / 자동
```

반면 다음은 자동으로 붙지 않습니다. 초기화되지 않은 메모리가 노출될 수 있기 때문입니다.

- `bool`은 0/1만 유효하지만, 실제로는 niche가 유효하지 않은 비트를 잡아 주므로 자동으로 적용됩니다.
- `Ptr<T>`는 provenance 정보가 따로 있어 비트만으로는 표현할 수 없습니다.
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

## 1.2.6. `transmute_unchecked<A, B>`: escape hatch

```rok
fn transmute_unchecked<A, B>(a: A) -> B;       // size check 도 안 함
```

이 함수는 `Pod` 검증조차 **없이** 비트를 그대로 옮깁니다. 이름대로 가장 위험한 캐스트입니다. 초기화되지 않은 메모리 노출, 크기 불일치, 정렬 문제는 모두 호출자 책임입니다.

표준 라이브러리 내부의 가장 **낮은 층**에서만 씁니다. `Pod` impl 자체를 작성할 때처럼 다른 도구로는 표현할 수 없는 경우뿐입니다. 일반 사용자 코드에서는 어떤 경우에도 쓰면 안 됩니다.

```rok
// 표준 라이브러리 내부 예
impl Pod for SomeNewtype {
    // empty marker
}

let s: SomeNewtype = ...;
let bits: u64 = rok::bare::transmute_unchecked(s);   // SomeNewtype 이 size 8 보장한 코드
```

## 1.2.7. `as` 캐스트의 적용 자리

`as`는 수치 변환에만 쓸 수 있습니다. 그중에서도 손실이 없거나 포화(saturate) / 래핑(wrap)을 명시한 경우뿐입니다.

| 표현               | 결과                                                   |
|--------------------|--------------------------------------------------------|
| `i32 as i64`       | OK (손실 없는 확장)                                    |
| `i64 as i32`       | **컴파일 에러**: `try_into` / `saturating_into`        |
| `f32 as f64`       | OK (손실 없음)                                         |
| `i32 as f32`       | **컴파일 에러**: `as f32` 정밀도 손실 가능             |
| `u32 as i32`       | **컴파일 에러**: 비트 재해석 (`reinterpret` 사용)      |
| `Ptr<T> as Ptr<U>` | **컴파일 에러**: `rok::bare::ptr_cast::<U>`             |
| `Ptr<T> as usize`  | **컴파일 에러**: `.addr()` 또는 `expose_provenance`    |
| `usize as Ptr<T>`  | **컴파일 에러**: `from_exposed_provenance`             |

전체 변환 매트릭스는 [2.11 conversion](../200-core/211-conversion.md)에 있습니다.

## 1.2.8. 정책 요약

이 장의 캐스트 정책은 네 가지로 정리됩니다.

- `as` 캐스트는 **손실 없는 수치 변환**만 허용합니다. 포인터와 비트 재해석은 거부합니다.
- `rok::bare::ptr_cast` / `align_cast` / `const_cast` / `reinterpret` 네 함수가 명시 캐스트의 통로입니다.
- `Pod` marker 트레잇이 비트 재해석의 안전 조건입니다.
- `transmute_unchecked`는 탈출구로만 노출합니다.

이 정책으로 Rok은 C의 번거로운 형 변환과 Rust의 `as` 함정을 함께 피합니다.
