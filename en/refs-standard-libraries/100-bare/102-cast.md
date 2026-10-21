# 1.2. cast operations — explicitly named casts

The `rok::bare::cast` module exposes conversions between pointers and types only as **named functions**. The reason `as` casts in C and Rust are so dangerous is that the conversion happens silently; to avoid that pitfall, rok requires every cast to carry an **explicit name**.

## 1.2.1. The 4 cast functions

The four basic functions are:

| Function                                 | Meaning                                    | Risk                    |
|------------------------------------------|--------------------------------------------|-------------------------|
| `ptr_cast<U>(p: Ptr<T>)`                 | Changes the type only (provenance/address kept) | alignment check    |
| `align_cast<T>(p: Ptr<u8>)`              | Alignment up-cast                          | checked (debug)         |
| `const_cast<T>(p: ConstPtr<T>)`          | Removes const                              | very high (caller guarantees) |
| `reinterpret<A: Pod, B: Pod>(a: A) -> B` | bit-level cast (`Pod` only)                | low (guaranteed by the Pod marker) |

In addition, there is one escape hatch for when a deeper bypass is needed.

| Function                              | Meaning                                         | Risk    |
|---------------------------------------|-------------------------------------------------|---------|
| `transmute_unchecked<A,B>(a: A) -> B` | bit cast **without Pod checking** (caller's responsibility) | highest |

The compiler rejects the `as` keyword on pointers, so users necessarily use named casts.

## 1.2.2. `ptr_cast<U>` — pointer type conversion

```rok
fn ptr_cast<U, T>(p: Ptr<T>) -> Ptr<U>;
fn ptr_cast<U, T>(p: ConstPtr<T>) -> ConstPtr<U>;
fn ptr_cast<U, T>(p: ManyPtr<T>) -> ManyPtr<U>;
```

This function keeps address and provenance intact and changes only the type tag. **Alignment is therefore the caller's responsibility.**

```rok
let p: Ptr<u32> = ...;        // 4-byte aligned
let q: Ptr<u8> = rok::bare::ptr_cast::<u8>(p);    // safe (down-align)
let r: Ptr<u64> = rok::bare::ptr_cast::<u64>(p);  // dangerous — u64 needs 8-byte
                                                 // r.read() may SIGBUS (ARM)
```

In debug builds an **alignment check** is inserted and leads to a panic; in release builds the check is gone and it is UB.

### Safe cast — type-equivalent

Conversion between two structs with identical layout is safe.

```rok
struct A { x: u32 }
struct B { x: u32 }       // same layout as A

let pa: Ptr<A> = ...;
let pb: Ptr<B> = rok::bare::ptr_cast::<B>(pa);    // safe if layout matches
```

The compiler does not verify that the layouts really match, however; the caller must bear that guarantee.

## 1.2.3. `align_cast<T>` — alignment check

```rok
fn align_cast<T>(p: Ptr<u8>) -> Ptr<T> | none;
```

When turning a `Ptr<u8>` (1-byte align) into a `Ptr<T>` of larger alignment, the conversion happens **only after the actual alignment is verified**.

```rok
let bytes: ManyPtr<u8> = ...;
let p: Ptr<u8> = bytes.add(offset).as_one();

if let aligned: Ptr<u32> = rok::bare::align_cast::<u32>(p) {
    let v = unsafe { aligned.read() };       // safe (verified)
} else {
    panic("not 4-byte aligned");
}
```

`align_cast` returns `T | none`, not `Result`, because the latter of the two forms is lighter.

### unchecked variant

```rok
fn align_cast_unchecked<T>(p: Ptr<u8>) -> Ptr<T>;     // without checking
```

Use it only where alignment has already been verified just before the call. This variant performs no check even in debug builds.

## 1.2.4. `const_cast<T>` — const removal

```rok
fn const_cast<T>(p: ConstPtr<T>) -> Ptr<T>;
```

This function strips const. The danger is obvious: if the original lives in a read-only region (immortal const), writing there segfaults immediately.

```rok
let s: str = "hello";              // immortal const
let p: ConstPtr<u8> = s.as_ptr();
let m: Ptr<u8> = rok::bare::const_cast(p);
unsafe { m.write(c'H') };          // SEGFAULT — write to immortal const
```

It is exactly isomorphic to C's `const_cast<T*>`. Its use is limited to temporarily wrapping a read-only API where mutability is required; ordinary code **must not use it**.

## 1.2.5. `reinterpret<A: Pod, B: Pod>` — bit cast (safe)

```rok
trait Pod { /* marker — Plain Old Data */ }

fn reinterpret<A: Pod, B: Pod>(a: A) -> B;
```

For types with the `Pod` (Plain Old Data) marker, **safe transmute** is guaranteed, because:

- Every bit pattern of a `Pod` type is valid, so uninit can never be exposed.
- Conversion is allowed only between equal sizes. A `where size_of` bound has no compile-time form, so the size condition is checked when `reinterpret` runs (a mismatch panics).
- It is a value conversion, so alignment is naturally guaranteed.

```rok
let f: f32 = 3.14;
let bits: u32 = rok::bare::reinterpret(f);     // OK — both Pod, 4 bytes
let f2: f32 = rok::bare::reinterpret(bits);    // round-trip OK
```

### Automatic `Pod` derive

The standard library automatically gives `Pod` to these types:

```rok
impl Pod for i8, i16, i32, i64, u8, u16, u32, u64, usize, isize { }
impl Pod for f32, f64 { }
impl Pod for bool { }
impl Pod for char, c_char { }
impl<T: Pod, const N: usize> Pod for [T; N] { }
impl<T: Pod, U: Pod> Pod for (T, U) { }
// larger tuple arities via macro / automatic
```

The following are not given it automatically (because uninit could be exposed):

- `bool` is valid only as 0/1, but in practice the niche catches invalid bits, so it is applied automatically.
- `Ptr<T>` carries separate provenance information and cannot be represented by bits alone.
- User structs must opt in explicitly with `#[derive(Pod)]`.

```rok
#[derive(Pod)]
#[repr(C)]
struct Vec3 { x: f32; y: f32; z: f32; }

let v: Vec3 = Vec3 { x: 1.0, y: 2.0, z: 3.0 };
let bytes: [u8; 12] = rok::bare::reinterpret(v);
let v2: Vec3 = rok::bare::reinterpret(bytes);
```

This design follows the [Project Safe Transmute (Rust RFC 2835)](https://rust-lang.github.io/rfcs/2835-project-safe-transmute.html) model and the [bytemuck crate](https://docs.rs/bytemuck) pattern.

## 1.2.6. `transmute_unchecked<A, B>` — escape hatch

```rok
fn transmute_unchecked<A, B>(a: A) -> B;       // no size check either
```

This function moves bits as is, **without** even `Pod` checking. As the name says, it is the most dangerous cast: uninit exposure, size mismatch and alignment problems are all the caller's responsibility.

Its use is limited to the **lowest layer** inside the standard library, only for cases that cannot be expressed with any other tool, such as writing a `Pod` impl itself. Ordinary user code must never use it.

```rok
// example inside the standard library
impl Pod for SomeNewtype {
    // empty marker
}

let s: SomeNewtype = ...;
let bits: u64 = rok::bare::transmute_unchecked(s);   // code where SomeNewtype guarantees size 8
```

## 1.2.7. Where `as` casts apply

`as` is allowed only for numeric conversions, and among those only when lossless or when saturate / wrap is explicit.

| Expression         | Result                                                 |
|--------------------|--------------------------------------------------------|
| `i32 as i64`       | OK (lossless widening)                                 |
| `i64 as i32`       | **Compile error** — `try_into` / `saturating_into`     |
| `f32 as f64`       | OK (lossless)                                          |
| `i32 as f32`       | **Compile error** — `as f32` may lose precision        |
| `u32 as i32`       | **Compile error** — bit reinterpret (use `reinterpret`) |
| `Ptr<T> as Ptr<U>` | **Compile error** — `rok::bare::ptr_cast::<U>`          |
| `Ptr<T> as usize`  | **Compile error** — `.addr()` or `expose_provenance`   |
| `usize as Ptr<T>`  | **Compile error** — `from_exposed_provenance`          |

See [2.11 conversion](../200-core/211-conversion.md) for the full conversion matrix.

## 1.2.8. Policy summary

The cast policy of this chapter comes down to four points.

- `as` casts allow only **lossless numeric** conversions; pointers and bit reinterpretation are rejected.
- The four functions `rok::bare::ptr_cast` / `align_cast` / `const_cast` / `reinterpret` are the channels for explicit casts.
- The `Pod` marker trait provides the safety condition for bit reinterpretation.
- `transmute_unchecked` is exposed only as an escape hatch.

With this policy rok avoids both C's cumbersome type conversions and Rust's `as` pitfalls.
