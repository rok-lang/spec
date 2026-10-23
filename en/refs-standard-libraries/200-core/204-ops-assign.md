# 2.4. Compound assignment traits

The `*Assign` trait family of `rok::core::ops` dispatches compound assignment operators such as `a += b` and `a *= b` as **in-place** updates. The syntax defined in [refs-lang 3.4 Assignment and compound assignment](../../refs-lang/003-expressions/304-assignment.md) desugars to these traits.

> [!NOTE]
> rok does not use a `mut` keyword in ordinary code ([refs-lang 5.1 Variable declarations](../../refs-lang/005-modules/501-variable-declaration.md)).
> Variables are always declared with `let`, and assigning to the same name again is a **reassignment**.
> In-place update of a heap object (List, Dict, etc.) is determined not by a `let mut` declarator but by
> **whether the object exposes mutating methods**. The `*Assign` traits are where those mutating methods live.

## 2.4.1. The key difference - `+=` vs `=` then `+`

```rok
a += b;
```

This expression has two possible meanings. With **in-place dispatch**, it resolves to `<typeof(a) as AddAssign<typeof(b)>>::add_assign(ref a, b)`, which directly modifies the **contents** of the object `a` refers to and creates no new object. With the **fallback** (`Add::add` followed by reassignment), it becomes `a = a + b`, storing the result of the `Add` call back into `a` and creating a new object.

rok chooses between the two modes automatically.

```
Compiler behavior (on a += b):
1. If typeof(a) implements AddAssign<typeof(b)>
     → dispatch AddAssign::add_assign(ref a, b)
2. Otherwise, if typeof(a) implements Add<typeof(b)>
     → desugar to a = Add::add(a, b)
3. If neither, compile error
```

This policy enables in-place optimization of collections. For example, when `xs: List<i32>` performs `xs += y`, it pushes directly without creating a new List.

## 2.4.2. Arithmetic *Assign traits

### `AddAssign` - `+=`

```rok
trait AddAssign<Rhs = Self> {
    #[infix("+=")]
    fn add_assign(ref self, rhs: Rhs);
}
```

```rok
let counter: i32 = 0;
counter += 5;       //→ 5 (counter is updated in place)
counter += 3;       //→ 8

let xs: List<i32> = [1, 2];
xs += [3, 4];       //→ [1, 2, 3, 4]  (List's add_assign - efficient push)
```

### `SubAssign` - `-=`

```rok
trait SubAssign<Rhs = Self> {
    #[infix("-=")]
    fn sub_assign(ref self, rhs: Rhs);
}
```

### `MulAssign` - `*=`

```rok
trait MulAssign<Rhs = Self> {
    #[infix("*=")]
    fn mul_assign(ref self, rhs: Rhs);
}
```

### `DivAssign` - `/=`

```rok
trait DivAssign<Rhs = Self> {
    #[infix("/=")]
    fn div_assign(ref self, rhs: Rhs);
}
```

### `RemAssign` - `%=`

```rok
trait RemAssign<Rhs = Self> {
    #[infix("%=")]
    fn rem_assign(ref self, rhs: Rhs);
}
```

### `ModuloAssign` - `%%=`

```rok
trait ModuloAssign<Rhs = Self> {
    #[infix("%%=")]
    fn modulo_assign(ref self, rhs: Rhs);
}
```

`%%=` is the compound assignment form of [`Modulo`](203-ops.md#modulo---floored-modulo-) (floored modulo, sign of result = sign of divisor). It is a separate trait from `RemAssign`, and updates in place so that, for a positive divisor, the result always lies in `[0, rhs)`.

## 2.4.3. Bitwise *Assign traits

### `BitAndAssign` - `&=`

```rok
trait BitAndAssign<Rhs = Self> {
    #[infix("&=")]
    fn bitand_assign(ref self, rhs: Rhs);
}
```

```rok
let flags: u32 = 0xFF;
flags &= 0x0F;     //→ 0x0F
```

### `BitOrAssign` - `|=`

```rok
trait BitOrAssign<Rhs = Self> {
    #[infix("|=")]
    fn bitor_assign(ref self, rhs: Rhs);
}
```

### `BitXorAssign` - `^=`

```rok
trait BitXorAssign<Rhs = Self> {
    #[infix("^=")]
    fn bitxor_assign(ref self, rhs: Rhs);
}
```

### `ShlAssign` - `<<=`

```rok
trait ShlAssign<Rhs = Self> {
    #[infix("<<=")]
    fn shl_assign(ref self, rhs: Rhs);
}
```

```rok
let n: i32 = 1;
n <<= 4;       //→ 16
```

### `ShrAssign` - `>>=`

```rok
trait ShrAssign<Rhs = Self> {
    #[infix(">>=")]
    fn shr_assign(ref self, rhs: Rhs);
}
```

## 2.4.4. Signature pattern - `ref self`

Every `*Assign` method takes `ref self` ([refs-lang 4.7 ref keyword](../../refs-lang/004-functions/407-ref-keyword.md)). It refers to the caller's variable binding itself, so the modification happens in place.

```rok
fn add_assign(ref self, rhs: Rhs);
       ^^^^^
       directly modifies the object the caller's variable refers to
```

This signature is the core of in-place behavior. If it took only `self` (by value), it would modify a copy, and the caller would not see the change.

## 2.4.5. Precedence / associativity - prec=-1, right

Every `*Assign` token is fixed at the same level in the [refs-lang 3.2](../../refs-lang/003-expressions/302-precedence-and-associativity.md) table. Its precedence is `prec = -1`, lower than ordinary arithmetic (+=7, *=8) and lower than `..` (=0). It is the lowest level. Chained assignment is not allowed ([refs-lang 3.4.4](../../refs-lang/003-expressions/304-assignment.md)).

## 2.4.6. Trait implementation matrix

Every `*Assign` impl for primitive types carries `#[intrinsic("rok.OP_assign.TYPE")]`.

| Type         | Traits                                      | Behavior                                        |
|--------------|---------------------------------------------|-------------------------------------------------|
| `i*`, `u*`   | 6 arithmetic + 5 bitwise (11 traits)        | `rok.add_assign.i32`, `rok.shl_assign.u32`, ... |
| `f32`, `f64` | 6 arithmetic only                           | `rok.add_assign.f64`, ...                       |
| `bool`       | `BitAndAssign`/`BitOrAssign`/`BitXorAssign` | `rok.and_assign.bool`, ...                      |
| `int`        | Arithmetic + bitwise                        | auto-promote logic - replaced by a new value (immutable) |
| `float`      | Arithmetic                                  | (same as above - auto-promote)                  |
| `List<T>`    | `AddAssign<List<T>>`, `AddAssign<T>`        | push                                            |
| `str`        | `AddAssign<str>`, `AddAssign<char>`         | `s = s + t` - replaced by a new str (str is immutable) |
| `Dict<K,V>`  | `AddAssign<Dict<K,V>>`                      | merge - overwrites existing keys                |

`a op= b` calls `XxxAssign::xxx_assign` (`AddAssign::add_assign`, …) when the left type implements it; otherwise it is `a = a op b`.

## 2.4.7. *Assign on collections - the main use

For mutable containers such as `List<T>` / `Dict<K,V>`, `*Assign` performs a **true in-place** operation.

```rok
let xs: List<i32> = [1, 2, 3];
let alias = xs;             // shares the same object (Python-style, refs-lang 4.7.x)

xs += [4, 5];               // push onto xs (no new object)
alias;                       //→ [1, 2, 3, 4, 5]  (alias sees the change too)

// compare: xs = xs + [4, 5] creates a new List, and alias does not see the change
```

This meaning matches rok's automatic sharing model for heap objects ([refs-lang 4.7](../../refs-lang/004-functions/407-ref-keyword.md)) exactly.

## 2.4.8. User-defined *Assign

If a user type implements only `Add` and not `AddAssign`, the compiler uses the fallback.

```rok
struct Vec3 { x: f64; y: f64; z: f64; }

impl Add for Vec3 {
    type Output = Vec3;
    fn add(self, rhs: Vec3) -> Vec3 { ... }
}
// AddAssign not defined

let v = Vec3 { x: 1.0, y: 2.0, z: 3.0 };
v += Vec3 { x: 0.5, y: 0.5, z: 0.5 };
// → compiler: no AddAssign, Add exists → desugar 'v = v + (...)'
// Vec3 is immutable, so a new object is created and reassigned to v
```

Where performance matters, the user is advised to implement `AddAssign` explicitly.

```rok
impl AddAssign for Vec3 {
    fn add_assign(ref self, rhs: Vec3) {
        self.x += rhs.x;
        self.y += rhs.y;
        self.z += rhs.z;
    }
}
```

## 2.4.9. Supertrait relationship of *Assign - none

rok does **not** make `Add` a supertrait of `AddAssign`. The two traits are independent.

The reason is that some types, such as collections, have no meaningful `Add` (`xs + ys` is expensive because it creates a new collection) but do have a meaningful `AddAssign`. A supertrait relationship would make it impossible for such types to implement only `AddAssign`.

## 2.4.10. Related - broadcast `*Assign`?

No compound assignment variants are defined for the 19 tokens of [2.6 broadcast](206-broadcast.md) (`.+`, `.*`, `.%%`, `.&&`, ...). That is, a token such as `a .+= b` does not exist. The background is that broadcast results usually create a new container, which makes in-place meaning ambiguous.

Where needed, the user calls a method such as `dot_add_assign` explicitly.

```rok
xs.dot_add_assign(ys);    // method call (broadcast in place)
```

```rok
impl<T> List<T> {
    fn dot_add_assign(self, rhs: List<T>) where T: Add<Output = T>;
    // likewise dot_OP_assign for OP in add, sub, mul, div, rem, modulo,
    // bitand, bitor, bitxor, shl, shr - each requiring T: OP<Output = T>
}
```

The lengths must match; otherwise it panics. There is no `Slice` form.
