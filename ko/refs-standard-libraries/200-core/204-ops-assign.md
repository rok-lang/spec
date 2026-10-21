# 2.4. 복합 대입 트레잇

`rok::core::ops` 의 `*Assign` 트레잇 패밀리는 `a += b`, `a *= b` 같은 복합 대입 연산자를 **in-place** 갱신으로 dispatch 하는 역할을 담당합니다. 즉, [refs-lang 3.4 대입과 복합 대입](../../refs-lang/003-expressions/304-assignment.md) 에서 정의된 syntax 가 본 트레잇으로 desugar 되는 구조입니다.

> [!NOTE]
> rok 은 `mut` 키워드를 일반 코드에서 쓰지 않습니다 ([refs-lang 5.1 변수 선언](../../refs-lang/005-modules/501-variable-declaration.md)).
> 변수는 항상 `let` 으로 선언되고, 같은 이름에 다시 대입하면 **재할당** 으로 처리됩니다.
> heap 객체 (List, Dict 등) 의 in-place 갱신은 변수 선언자 `let mut` 가 아니라
> **객체가 가변 메서드를 노출하는지** 로 결정됩니다. `*Assign` 트레잇이 그 가변 메서드의 자리입니다.

## 2.4.1. 핵심 차이 — `+=` vs `=` then `+`

```rok
a += b;
```

이 식에는 두 가지 의미가 가능합니다. 먼저 **in-place dispatch** 의 경우 `<typeof(a) as AddAssign<typeof(b)>>::add_assign(ref a, b)` 로 풀려서 `a` 가 가리키는 객체의 **내용** 을 직접 수정하므로 새 객체를 만들지 않습니다. 반면 **fallback** (`Add::add` 후 재대입) 의 경우 `a = a + b` 형태가 되어 `Add` 호출 결과를 `a` 에 다시 저장하면서 새 객체를 생성합니다.

rok 은 이 두 모드를 자동으로 선택합니다.

```
컴파일러 동작 (a += b 받았을 때):
1. typeof(a) 가 AddAssign<typeof(b)> 구현했으면
     → AddAssign::add_assign(ref a, b) dispatch
2. 그렇지 않고 typeof(a) 가 Add<typeof(b)> 구현했으면
     → a = Add::add(a, b) 로 desugar
3. 둘 다 없으면 컴파일 에러
```

이러한 정책 덕분에 컬렉션의 in-place 최적화가 가능해집니다. 예를 들어 `xs: List<i32>` 가 `xs += y` 를 수행할 때 새 List 를 만들지 않고 직접 push 됩니다.

## 2.4.2. 산술 *Assign 트레잇

### `AddAssign` — `+=`

```rok
trait AddAssign<Rhs = Self> {
    #[infix("+=")]
    fn add_assign(ref self, rhs: Rhs);
}
```

```rok
let counter: i32 = 0;
counter += 5;       //→ 5 (counter 가 in-place 갱신)
counter += 3;       //→ 8

let xs: List<i32> = [1, 2];
xs += [3, 4];       //→ [1, 2, 3, 4]  (List 의 add_assign — push 효율)
```

### `SubAssign` — `-=`

```rok
trait SubAssign<Rhs = Self> {
    #[infix("-=")]
    fn sub_assign(ref self, rhs: Rhs);
}
```

### `MulAssign` — `*=`

```rok
trait MulAssign<Rhs = Self> {
    #[infix("*=")]
    fn mul_assign(ref self, rhs: Rhs);
}
```

### `DivAssign` — `/=`

```rok
trait DivAssign<Rhs = Self> {
    #[infix("/=")]
    fn div_assign(ref self, rhs: Rhs);
}
```

### `RemAssign` — `%=`

```rok
trait RemAssign<Rhs = Self> {
    #[infix("%=")]
    fn rem_assign(ref self, rhs: Rhs);
}
```

### `ModuloAssign` — `%%=`

```rok
trait ModuloAssign<Rhs = Self> {
    #[infix("%%=")]
    fn modulo_assign(ref self, rhs: Rhs);
}
```

`%%=` 는 [`Modulo`](203-ops.md#modulo--floored-modulo-) (floored modulo, 결과 부호 = 제수 부호) 의 복합 대입 형태입니다. `RemAssign` 과 다른 트레잇으로 분리되어 있으며, 양수 제수에 대해 결과가 항상 `[0, rhs)` 범위에 있도록 in-place 갱신합니다.

## 2.4.3. 비트 *Assign 트레잇

### `BitAndAssign` — `&=`

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

### `BitOrAssign` — `|=`

```rok
trait BitOrAssign<Rhs = Self> {
    #[infix("|=")]
    fn bitor_assign(ref self, rhs: Rhs);
}
```

### `BitXorAssign` — `^=`

```rok
trait BitXorAssign<Rhs = Self> {
    #[infix("^=")]
    fn bitxor_assign(ref self, rhs: Rhs);
}
```

### `ShlAssign` — `<<=`

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

### `ShrAssign` — `>>=`

```rok
trait ShrAssign<Rhs = Self> {
    #[infix(">>=")]
    fn shr_assign(ref self, rhs: Rhs);
}
```

## 2.4.4. 시그니처 패턴 — `ref self`

모든 `*Assign` 메서드는 `ref self` 를 받습니다 ([refs-lang 4.7 ref keyword](../../refs-lang/004-functions/407-ref-keyword.md)). 이렇게 함으로써 caller 의 변수 binding 자체를 참조해 in-place 수정이 이루어집니다.

```rok
fn add_assign(ref self, rhs: Rhs);
       ^^^^^
       caller 의 변수가 가리키는 객체를 직접 수정
```

이 시그니처가 in-place 의 핵심에 해당합니다. 만일 `self` (값) 만 받게 된다면 복사본을 수정하는 데 그쳐서 caller 측에서는 변경 결과를 볼 수 없게 됩니다.

## 2.4.5. 우선순위 / 결합성 — prec=-1, right

모든 `*Assign` 토큰은 [refs-lang 3.2](../../refs-lang/003-expressions/302-precedence-and-associativity.md) 표에서 동일한 등급으로 고정됩니다. 우선순위 측면에서 `prec = -1` 을 가지므로 일반 산술 (+=7, *=8) 보다 낮고, `..` (=0) 보다도 낮은 위치에 자리합니다. 가장 낮은 등급입니다. 연쇄 대입은 허용하지 않습니다 ([refs-lang 3.4.4](../../refs-lang/003-expressions/304-assignment.md)).

## 2.4.6. 트레잇 구현 매트릭스

primitive 타입의 `*Assign` impl 에는 모두 `#[intrinsic("rok.OP_assign.TYPE")]` 가 부착됩니다.

| 타입         | 트레잇                                      | 동작                                            |
|--------------|---------------------------------------------|-------------------------------------------------|
| `i*`, `u*`   | 산술 6 개 + 비트 5 개 (11 트레잇)           | `rok.add_assign.i32`, `rok.shl_assign.u32`, ... |
| `f32`, `f64` | 산술 6 개만                                 | `rok.add_assign.f64`, ...                       |
| `bool`       | `BitAndAssign`/`BitOrAssign`/`BitXorAssign` | `rok.and_assign.bool`, ...                      |
| `int`        | 산술 + 비트                                 | auto-promote 로직 — 새 값으로 교체 (불변)       |
| `float`      | 산술                                        | (위와 동일 — auto-promote)                      |
| `List<T>`    | `AddAssign<List<T>>`, `AddAssign<T>`        | push                                            |
| `str`        | `AddAssign<str>`, `AddAssign<char>`         | `s = s + t` — 새 str 로 교체 (str 은 불변)      |
| `Dict<K,V>`  | `AddAssign<Dict<K,V>>`                      | merge — 기존 키 덮어씀                          |

`a op= b` 는 왼쪽 타입이 `XxxAssign` 을 구현하면 `XxxAssign::xxx_assign` (`AddAssign::add_assign` 등) 을 부르고, 아니면 `a = a op b` 입니다.

## 2.4.7. 컬렉션의 *Assign — 핵심 사용처

`List<T>` / `Dict<K,V>` 같은 가변 컨테이너에서는 `*Assign` 가 **진짜 in-place** 동작을 수행합니다.

```rok
let xs: List<i32> = [1, 2, 3];
let alias = xs;             // 같은 객체 공유 (Python 식, refs-lang 4.7.x)

xs += [4, 5];               // xs 에 push (새 객체 안 만듦)
alias;                       //→ [1, 2, 3, 4, 5]  (alias 도 변화 봄)

// 비교: xs = xs + [4, 5] 는 새 List 생성, alias 는 변화 못 봄
```

이 의미는 rok 의 heap 객체 자동 공유 모델 ([refs-lang 4.7](../../refs-lang/004-functions/407-ref-keyword.md)) 과 정확히 일치합니다.

## 2.4.8. 사용자 정의 *Assign

사용자 타입이 `Add` 만 구현하고 `AddAssign` 을 구현하지 않은 경우 컴파일러가 fallback 을 사용합니다.

```rok
struct Vec3 { x: f64; y: f64; z: f64; }

impl Add for Vec3 {
    type Output = Vec3;
    fn add(self, rhs: Vec3) -> Vec3 { ... }
}
// AddAssign 정의 안 됨

let v = Vec3 { x: 1.0, y: 2.0, z: 3.0 };
v += Vec3 { x: 0.5, y: 0.5, z: 0.5 };
// → 컴파일러: AddAssign 없음, Add 있음 → desugar 'v = v + (...)'
// Vec3 가 immutable 이라 새 객체 생성 후 v 재대입
```

성능이 중요한 경우라면 사용자가 `AddAssign` 을 명시적으로 구현하는 편이 권장됩니다.

```rok
impl AddAssign for Vec3 {
    fn add_assign(ref self, rhs: Vec3) {
        self.x += rhs.x;
        self.y += rhs.y;
        self.z += rhs.z;
    }
}
```

## 2.4.9. *Assign 의 supertrait 관계 — 없음

rok 은 `AddAssign` 을 `Add` 의 supertrait 로 묶지 **않습니다**. 따라서 두 트레잇은 서로 독립적으로 다루어집니다.

그 이유는 컬렉션처럼 `Add` 가 의미가 없고 (`xs + ys` 가 새 컬렉션을 만드는 비용이 큼) `AddAssign` 만 의미가 있는 경우가 존재하기 때문입니다. 만약 supertrait 로 묶는다면 그러한 타입이 `AddAssign` 만 구현하는 일이 불가능해집니다.

## 2.4.10. 관련 — broadcast `*Assign`?

[2.6 broadcast](206-broadcast.md) 의 19 개 토큰 (`.+`, `.*`, `.%%`, `.&&`, ...) 에 대응하는 복합 대입 변형은 정의되지 않습니다. 즉 `a .+= b` 같은 토큰은 존재하지 않습니다. broadcast 의 결과는 보통 새 컨테이너를 생성하기 때문에 in-place 의미가 모호해진다는 점이 그 배경입니다.

필요한 경우 사용자가 `dot_add_assign` 같은 메서드를 명시적으로 호출하면 됩니다.

```rok
xs.dot_add_assign(ys);    // 메서드 호출 (broadcast in-place)
```

```rok
impl<T> List<T> {
    fn dot_add_assign(self, rhs: List<T>) where T: Add<Output = T>;
    // dot_OP_assign 도 같음 — OP 는 add, sub, mul, div, rem, modulo,
    // bitand, bitor, bitxor, shl, shr 이며 각각 T: OP<Output = T> 필요
}
```

길이가 같아야 하며, 다르면 panic 입니다. `Slice` 형태는 없습니다.
