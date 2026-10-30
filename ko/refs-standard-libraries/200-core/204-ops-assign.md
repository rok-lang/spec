# 2.4. 복합 대입 트레잇

`rok::core::ops`의 `*Assign` 트레잇 패밀리는 `a += b`, `a *= b` 같은 복합 대입 연산자를 **제자리(in-place)** 갱신으로 디스패치합니다. [refs-lang 3.4 대입과 복합 대입](../../refs-lang/003-expressions/304-assignment.md)이 정의한 문법은 이 트레잇 호출로 풀립니다.

> [!NOTE]
> Rok은 `mut` 키워드를 일반 코드에서 쓰지 않습니다([refs-lang 5.1 변수 선언](../../refs-lang/005-modules/501-variable-declaration.md)).
> 변수는 항상 `let`으로 선언하며, 같은 이름에 다시 대입하면 **재할당**입니다.
> 힙 객체(List, Dict 등)를 제자리에서 갱신할 수 있는지는 변수 선언자 `let mut`가 아니라
> **객체가 가변 메서드를 노출하는지**로 정해집니다. `*Assign` 트레잇이 그 가변 메서드의 자리입니다.

## 2.4.1. 핵심 차이: `+=` 대 `=` 뒤 `+`

```rok
a += b;
```

이 식은 두 가지로 해석할 수 있습니다. **제자리 디스패치**에서는 식이 `<typeof(a) as AddAssign<typeof(b)>>::add_assign(ref a, b)`로 풀립니다. 이 호출은 `a`가 가리키는 객체의 **내용**을 직접 고치고, 새 객체를 만들지 않습니다. **대체 경로(fallback)**(`Add::add` 후 재대입)에서는 식이 `a = a + b`가 됩니다. `Add` 호출 결과를 `a`에 다시 저장하므로 새 객체가 생깁니다.

Rok은 두 모드 중 하나를 자동으로 고릅니다.

```
컴파일러 동작 (a += b 받았을 때):
1. typeof(a) 가 AddAssign<typeof(b)> 구현했으면
     → AddAssign::add_assign(ref a, b) dispatch
2. 그렇지 않고 typeof(a) 가 Add<typeof(b)> 구현했으면
     → a = Add::add(a, b) 로 desugar
3. 둘 다 없으면 컴파일 에러
```

이 정책 덕분에 컬렉션은 제자리 갱신으로 최적화할 수 있습니다. 예를 들어 `xs: List<i32>`에 `xs += y`를 하면 새 List를 만들지 않고 바로 원소를 넣습니다.

## 2.4.2. 산술 *Assign 트레잇

### `AddAssign`: `+=`

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
xs += [3, 4];       //→ [1, 2, 3, 4]  (List 의 add_assign - push 효율)
```

### `SubAssign`: `-=`

```rok
trait SubAssign<Rhs = Self> {
    #[infix("-=")]
    fn sub_assign(ref self, rhs: Rhs);
}
```

### `MulAssign`: `*=`

```rok
trait MulAssign<Rhs = Self> {
    #[infix("*=")]
    fn mul_assign(ref self, rhs: Rhs);
}
```

### `DivAssign`: `/=`

```rok
trait DivAssign<Rhs = Self> {
    #[infix("/=")]
    fn div_assign(ref self, rhs: Rhs);
}
```

### `RemAssign`: `%=`

```rok
trait RemAssign<Rhs = Self> {
    #[infix("%=")]
    fn rem_assign(ref self, rhs: Rhs);
}
```

### `ModuloAssign`: `%%=`

```rok
trait ModuloAssign<Rhs = Self> {
    #[infix("%%=")]
    fn modulo_assign(ref self, rhs: Rhs);
}
```

`%%=`는 [`Modulo`](203-ops.md#modulo-내림-modulo) (floored modulo, 결과 부호 = 제수 부호)의 복합 대입 형태입니다. `RemAssign`과는 다른 트레잇입니다. 제수가 양수이면 결과가 늘 `[0, rhs)` 범위에 들도록 제자리에서 갱신합니다.

## 2.4.3. 비트 *Assign 트레잇

### `BitAndAssign`: `&=`

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

### `BitOrAssign`: `|=`

```rok
trait BitOrAssign<Rhs = Self> {
    #[infix("|=")]
    fn bitor_assign(ref self, rhs: Rhs);
}
```

### `BitXorAssign`: `^=`

```rok
trait BitXorAssign<Rhs = Self> {
    #[infix("^=")]
    fn bitxor_assign(ref self, rhs: Rhs);
}
```

### `ShlAssign`: `<<=`

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

### `ShrAssign`: `>>=`

```rok
trait ShrAssign<Rhs = Self> {
    #[infix(">>=")]
    fn shr_assign(ref self, rhs: Rhs);
}
```

## 2.4.4. 시그니처 패턴: `ref self`

모든 `*Assign` 메서드는 `ref self`를 받습니다([refs-lang 4.7 ref keyword](../../refs-lang/004-functions/407-ref-keyword.md)). 그래서 호출한 쪽의 변수 바인딩 자체를 참조해 제자리에서 고칩니다.

```rok
fn add_assign(ref self, rhs: Rhs);
       ^^^^^
       caller 의 변수가 가리키는 객체를 직접 수정
```

제자리 갱신의 열쇠가 이 시그니처입니다. `self`(값)만 받는다면 복사본만 고치므로 호출한 쪽에서는 바뀐 결과를 볼 수 없습니다.

## 2.4.5. 우선순위 / 결합성: prec=-1, right

모든 `*Assign` 토큰은 [refs-lang 3.2](../../refs-lang/003-expressions/302-precedence-and-associativity.md) 표에서 같은 등급으로 고정됩니다. 우선순위는 `prec = -1`이므로 일반 산술(+=7, *=8)보다 낮고 `..`(=0)보다도 낮습니다. 가장 낮은 등급입니다. 연쇄 대입은 쓸 수 없습니다([refs-lang 3.4.4](../../refs-lang/003-expressions/304-assignment.md)).

## 2.4.6. 트레잇 구현 매트릭스

원시 타입의 `*Assign` 구현에는 모두 `#[intrinsic("rok.OP_assign.TYPE")]`이 붙습니다.

| 타입         | 트레잇                                      | 동작                                            |
|--------------|---------------------------------------------|-------------------------------------------------|
| `i*`, `u*`   | 산술 6개 + 비트 5개 (11 트레잇)           | `rok.add_assign.i32`, `rok.shl_assign.u32`, ... |
| `f32`, `f64` | 산술 6개만                                  | `rok.add_assign.f64`, ...                       |
| `bool`       | `BitAndAssign`/`BitOrAssign`/`BitXorAssign` | `rok.and_assign.bool`, ...                      |
| `int`        | 산술 + 비트                                 | 자동 승격 로직, 새 값으로 교체(불변)            |
| `float`      | 산술                                        | (위와 같음: 자동 승격)                          |
| `List<T>`    | `AddAssign<List<T>>`, `AddAssign<T>`        | 넣음                                            |
| `str`        | `AddAssign<str>`, `AddAssign<char>`         | `s = s + t`: 새 str로 교체(str은 불변)        |
| `Dict<K,V>`  | `AddAssign<Dict<K,V>>`                      | 병합, 기존 키는 덮어씀                          |

`a op= b`는 왼쪽 타입이 `XxxAssign`을 구현하면 `XxxAssign::xxx_assign`(`AddAssign::add_assign` 등)을 부릅니다. 구현하지 않으면 `a = a op b`입니다.

## 2.4.7. 컬렉션의 *Assign: 핵심 사용처

`List<T>` / `Dict<K,V>` 같은 가변 컨테이너에서는 `*Assign`이 **실제로 제자리에서** 동작합니다.

```rok
let xs: List<i32> = [1, 2, 3];
let alias = xs;             // 같은 객체 공유 (Python 식, refs-lang 4.7.x)

xs += [4, 5];               // xs 에 push (새 객체 안 만듦)
alias;                       //→ [1, 2, 3, 4, 5]  (alias 도 변화 봄)

// 비교: xs = xs + [4, 5] 는 새 List 생성, alias 는 변화 못 봄
```

이 의미는 Rok의 힙 객체 자동 공유 모델([refs-lang 4.7](../../refs-lang/004-functions/407-ref-keyword.md))과 정확히 일치합니다.

## 2.4.8. 사용자 정의 *Assign

사용자 타입이 `Add`만 구현하고 `AddAssign`은 구현하지 않았다면 컴파일러는 대체 경로를 씁니다.

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

성능이 중요하다면 `AddAssign`을 명시적으로 구현하기를 권장합니다.

```rok
impl AddAssign for Vec3 {
    fn add_assign(ref self, rhs: Vec3) {
        self.x += rhs.x;
        self.y += rhs.y;
        self.z += rhs.z;
    }
}
```

## 2.4.9. *Assign의 슈퍼트레잇 관계: 없음

Rok은 `Add`를 `AddAssign`의 상위 트레잇(supertrait)으로 묶지 **않습니다**. 두 트레잇은 서로 독립입니다.

컬렉션처럼 `Add`는 의미가 없고 `AddAssign`만 의미가 있는 타입이 있기 때문입니다. `xs + ys`는 새 컬렉션을 만드므로 비용이 큽니다. 상위 트레잇으로 묶으면 그런 타입은 `AddAssign`만 구현할 수 없습니다.

## 2.4.10. 관련: broadcast `*Assign` 여부

[2.6 broadcast](206-broadcast.md)의 토큰 19개(`.+`, `.*`, `.%%`, `.&&` 등)에는 복합 대입 변형이 없습니다. `a .+= b` 같은 토큰은 없습니다. 브로드캐스트 결과는 보통 새 컨테이너이므로 제자리 갱신의 의미가 모호하기 때문입니다.

필요하면 `dot_add_assign` 같은 메서드를 명시적으로 호출합니다.

```rok
xs.dot_add_assign(ys);    // 메서드 호출 (broadcast in-place)
```

```rok
impl<T> List<T> {
    fn dot_add_assign(self, rhs: List<T>) where T: Add<Output = T>;
    // dot_OP_assign 도 같음 - OP 는 add, sub, mul, div, rem, modulo,
    // bitand, bitor, bitxor, shl, shr 이며 각각 T: OP<Output = T> 필요
}
```

두 길이는 같아야 합니다. 다르면 panic이 일어납니다. `Slice` 형태는 없습니다.
