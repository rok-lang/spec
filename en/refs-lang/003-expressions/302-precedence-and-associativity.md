# 3.2. Precedence and associativity

Operators have two properties: precedence and associativity. When several operators appear in one expression, these two properties determine the shape of the resulting tree.

## 3.2.1. Precedence

```
a + b * c       →   a + (b * c)              // * binds tighter than +
a * b + c       →   (a * b) + c              // likewise
a < b + c       →   a < (b + c)              // + binds tighter than <
```

Precedence values are integers; an operator with a larger value binds first.

## 3.2.2. Associativity

```
a - b - c       →   (a - b) - c             // left-associative
a ?? b ?? c     →   a ?? (b ?? c)           // right-associative (example)
a ∈ s == t      →   ?                        // non-associative: an error
```

Associativity is one of three kinds.

| Notation | Meaning                                              |
|----------|------------------------------------------------------|
| `left`   | Groups from the left. `a OP b OP c` = `(a OP b) OP c` |
| `right`  | Groups from the right. `a OP b OP c` = `a OP (b OP c)` |
| `none`   | No grouping. `a OP b OP c` is a compile error, except the comparison chains of 3.2.5 |

## 3.2.3. Precedence and associativity table

The language fixes the precedence and associativity of every built-in operator token; the table below is the single authority. The standard library's [`#[infix]` / `#[prefix]` / `#[postfix]`](../006-meta-system/607-operator-magic.md) only connect tokens to trait methods; they do not set or change precedence or associativity.

### Precedence levels

```
Level Operators                                                 Assoc
14   postfix: member access (.), ?., method call, indexing, ?   left
13   ** .** (power)                                             right
13   prefix unary: - ! ~ √ await                                -
12   as is (postfix conversion / type test)                     left
11   ∘ (function composition)                                   left
10   * / % %% × ⋅ ÷ ∩                                           left
 9   + - ∪                                                      left
 8   << >>                                                      left
 7   &                                                          left
 6   ^                                                          left
 5   |                                                          left
 4   == != < > <= >= ≡ ≠ ≤ ≥ ∈ ∉ in !in ⊂ ⊆                     none (chains: 3.2.5)
 3   &&                                                         left
 2   ||                                                         left
 1   ?? (null coalescing)                                       right
 0   .. ..= (range)                                             none
-1   = += -= *= **= /= %= %%= &= |= ^= <<= >>= (assignment)     right  (statement only, 3.4.3)
```

The Unicode operators in the table (`× ⋅ ÷ ∩ ∪ ∘ √ ≡ ≠ ≤ ≥ ∈ ∉ ⊂ ⊆`) are fixed tokens of [1.10.3](../001-lexical/110-punctuation-and-operator-tokens.md#1103-fixed-token-table). Extension tokens not in the 3.2 table ([1.10.4](../001-lexical/110-punctuation-and-operator-tokens.md#1104-extension-tokens)) do not create new levels; they take the level and associativity of the existing token named at registration, as in `like = "+"` ([6.7.2](../006-meta-system/607-operator-magic.md#672-arguments)).

> [!IMPORTANT]
> **Relative precedence of `??` and `..`**: `??` (prec=1) binds **tighter** than the range operators `..` / `..=` (prec=0). So `a ?? 0..3` groups as `(a ?? 0)..3`, and the range operators are weaker than every binary operator except assignment (`a || b..c` is `(a || b)..c`). Neither operator dispatches through a trait; the language handles both directly: `??` is a language-level desugaring over sum types ([3.8.4](308-optional-chaining-and-coalescing.md#384-null-coalescing-)), and ranges are syntax whose six forms the language recognizes as distinct expressions ([3.7](307-range-expressions.md)). The range operators are non-associative, so `a..b..c` is a compile error.

> [!IMPORTANT]
> **Binding of `try` with `catch` / `finally`**: `try` / `try?` / `try!` and the `catch` / `finally` clauses of [error control flow](314-error-control-flow.md) are also language syntax with no trait connection. `try` is a prefix form, but unlike the prefix unary operators in the table (prec=13) it covers the entire expression to its right. Its operand extends up to a `catch` / `finally` clause, the statement terminator `;`, a closing bracket, or a comma, so `try a + b()` is `try (a + b())`. The operand of `try?` / `try!` also ends before `??`: `try? f() ?? d` is `(try? f()) ?? d`, while `try f() ?? d` is `try (f() ?? d)`. The `catch` / `finally` clauses bind tighter than assignment (prec=-1) and weaker than every other operator, attaching to the whole expression on their left. Thus `try parse(s) catch _ { 0 }` is `(try parse(s)) catch _ { 0 }`, and `x = try f() catch _ { 0 }` is `x = ((try f()) catch _ { 0 })`.

> [!IMPORTANT]
> **`in` / `!in` and ranges**: `in` and `!in` are the ASCII spellings of `∈` / `∉` ([1.10.3](../001-lexical/110-punctuation-and-operator-tokens.md#1103-fixed-token-table)). A range on the right of a membership operator needs no parentheses: after the right operand, `..` / `..=` continue it as a range whose end binds like the right operand itself (prec=5). So `i in 0..n` is `i ∈ (0..n)`, `i in 0..n + 1` is `i ∈ (0..(n + 1))`, and `i in 0..n && ok` is `(i ∈ (0..n)) && ok`.

> [!IMPORTANT]
> **`**` and prefix operators**: `**` (power, trait `Pow`, [stdlib 2.3.1](../../refs-standard-libraries/200-core/203-ops.md#231-arithmetic-traits)) sits on level 13 with the prefix operators but binds its left operand tighter than they do, as in mathematics, Python and Fortran: `-x**2` is `-(x**2)`, while `(-x)**2` squares `-x`. It is right-associative, `2**3**2` is `2**(3**2)` = 512, and its right operand may itself start with a prefix operator: `2.0 ** -1` is `0.5`, `2 ** -x ** 2` is `2 ** (-(x ** 2))`. Postfix forms bind tighter still (`a.b**2` is `(a.b)**2`), and `as` binds looser (`n ** 2 as f64` is `(n ** 2) as f64`). `.**` is the element-wise form at the same level.

> [!IMPORTANT]
> **`await`**: `await` is a prefix unary operator at prec=13, like `-` and `!`, so its operand is a unary expression: `await f() == x` is `(await f()) == x` ([4.12.2](../004-functions/412-async.md#4122-the-await-operator)).

> [!IMPORTANT]
> **`as` and `is`**: these two postfix operators are language syntax with no trait connection and share one level (prec=12). They are weaker than prefix unary and stronger than arithmetic, so `-x as i64` is `(-x) as i64` and `a + b is i32` is `a + (b is i32)`. The result of `e is T` is `bool` ([2.2.5](../002-types/202-any-slot.md#225-checking-the-slots-current-type)).

> [!NOTE]
> **Precedence of dot-prefixed (broadcast) operators**: dot-prefixed forms such as `.+`, `.*`, `.&&`, `.%%` have the same precedence and associativity as their undotted forms. They are not listed as separate rows; every dot-prefixed token takes the level of its corresponding plain token. For example, `.+` is prec=9 left like `+`, and `.&&` is prec=3 left like `&&`.

> [!NOTE]
> The binary levels are Rust's: arithmetic binds tighter than shifts, shifts tighter than `&`, then `^`, then `|`, and all of them tighter than comparisons. So `1 + 2 << 3` groups as `(1 + 2) << 3 = 24`, `a ^ b & c` as `a ^ (b & c)`, and `x & mask == 0` as `(x & mask) == 0`.

### Multiple positions of the same token

When the same token can appear as prefix, binary, and postfix (for example `-`), each position has its own precedence (prefix is usually stronger than binary).

```
-a + b          →   (-a) + b              // prefix `-` binds tighter than `+`
```

## 3.2.4. Conflicts

Because the table fixes precedence, registrations cannot conflict on precedence. Connecting different traits to the same (token, position) is a compile error per [6.7.5](../006-meta-system/607-operator-magic.md#675-conflict-checking).

## 3.2.5. Comparison chains

Comparisons of one direction chain, as in mathematics:

```rok
0 <= i < n          // 0 <= i && i < n
a > b >= c          // a > b && b >= c
a == b == c         // a == b && b == c
```

A chain is a run of level-4 operators taken from one group: `<` and `<=` (`≤`), or `>` and `>=` (`≥`), or `==` (`≡`). `e0 o1 e1 o2 e2 ..` means `e0 o1 e1 && e1 o2 e2 && ..`, with three guarantees:

- each operand is evaluated once, left to right;
- evaluation stops at the first comparison that is `false`, and the operands after it are not evaluated;
- the result is `bool`.

A middle operand that is a name, a field of one or a literal is read again for the next comparison, so a literal still takes its type from the other side (`0 <= x < 10` with `x: u8`). Any other middle operand is evaluated once into a temporary.

Everything else stays non-associative: a chain that mixes directions (`a < b > c`), `!=` (`a != b != c`), and membership or subset operators (`x ∈ s == t`) are a compile error (E0019). The help writes the `&&` form (`a < b && b > c`).

```
a < b > c       // ERROR: write a < b && b > c
a != b != c     // ERROR: write a != b && b != c
```

One-direction chains read the way they are written in mathematics and in Python, and a range test (`lo <= x < hi`) names `x` once. A mixed chain has no single reading, and `!=` would not be transitive, so these stay errors.

## 3.2.6. Forcing grouping with parentheses

When the tree produced by these rules differs from the intended shape, parentheses force the grouping.

```
a + b * c       →   a + (b * c)             // default grouping
(a + b) * c     →   (a + b) * c             // explicit
```

Parentheses act as grouping tokens in the lexical and syntactic phases and directly control the shape of the tree.

## 3.2.6a. Implementation limits

A chain of more than 4096 operators / postfix operations, or nesting deeper than 256, is an error. These are Rok's limits (implementation-defined, [0.4.4](../000-meta/004-language-vs-standard-libraries.md#044-implementation-limits)).

## 3.2.7. Two tokens with the same precedence and different associativity

In the 3.2.3 table all tokens in one level share the same associativity, so this case does not arise. Extension tokens also take the associativity of an existing token through `like`, so the same holds.
