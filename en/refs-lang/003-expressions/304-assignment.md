# 3.4. Assignment and compound assignment

Assignment (`=`) and compound assignment (`+=`, `-=`, etc.) update the value of a variable, a field, or an indexed place. Both are statements; a simple assignment may also stand in expression position in the forms of 3.4.5, while a compound assignment may not. The grammar is:

```
assign_stmt     ::= lvalue "=" expr ";"
compound_stmt   ::= lvalue compound_op expr ";"
assign_expr     ::= lvalue "=" expr                    // in parentheses or an arm body (3.4.5)
compound_op     ::= "+=" | "-=" | "*=" | "/=" | "%=" | "%%=" | "&=" | "|="
                  | "^=" | "<<=" | ">>="
lvalue          ::= identifier
                  | lvalue "." identifier
                  | lvalue "." int_lit                 // tuple position
                  | lvalue "[" expr "]"                // index
```

## 3.4.1. Simple assignment

Simple assignment has the following shape.

```rok
x = expr;
obj.field = expr;
arr[i] = expr;
```

The left-hand side (lvalue) is a reassignable place, the right-hand side (rvalue) is any expression, and the result value has the unit type (`()`). Assignment is normally a statement; where it stands in expression position (3.4.5) its result is unit. As a result, patterns like C's `if (x = read())`, which are rarely needed, are blocked by the unit result.

### Kinds of lvalue

The forms allowed on the left-hand side and their meanings:

| Form             | Example           | Meaning                                          |
|------------------|-------------------|--------------------------------------------------|
| simple variable  | `x = ...`         | writes the value to the variable's slot          |
| field access     | `obj.field = ...` | writes the value to the object's field           |
| tuple position   | `pair.0 = ...`    | writes the value to a position of a tuple `struct` / tuple |
| indexing         | `arr[i] = ...`    | writes the value to the indexed place            |

Indexing assignment desugars to the write traits of [indexing and slicing](306-indexing-and-slicing.md). A slice is not an lvalue: `arr[1:3] = other;` is rejected ([3.6](306-indexing-and-slicing.md)).

### Places that cannot be lvalues

The following cannot be lvalues.

```
1 = 2;                    // a literal is not an lvalue
foo() = 2;                // a call result is generally not an lvalue
(x + 1) = 2;              // an arithmetic result is not an lvalue
(a, b) = (b, a);          // a tuple is not an lvalue; write let (a, b) = (b, a);
```

Whether something can be an lvalue is checked at compile time and determined by form (shape) alone. No runtime check is performed.

## 3.4.2. Declaring a new variable vs. reassignment

Combined with the `let` [keyword](../001-lexical/105-keywords.md), two forms are distinguished.

```rok
let x = 1;       // new declaration, 'let' required
x = 2;           // reassignment, without 'let'
```

Creating a new variable requires `let`; writing again to an existing variable's slot is written without `let`. Assigning without `let` to an undeclared name is a compile error. The exact rules are defined in [variable declaration](../005-modules/501-variable-declaration.md).

```
counter = counter + 1;     // OK if counter is declared
coutner = ...;             // ERROR: `coutner` is not declared; a variable is created only by `let`
```

This rule blocks the trap of a typo silently creating a new variable (a common bug in Julia and Python).

## 3.4.3. Compound assignment

Compound assignment combines an operation and an assignment in one form.

```rok
x += 1;            // semantically the same as x = x + 1
x -= 1;
x *= 2;
arr[i] += 1;
```

All compound assignment tokens of [punctuation and operator tokens](../001-lexical/110-punctuation-and-operator-tokens.md) (`+=`, `-=`, `*=`, `/=`, `%=`, `%%=`, `&=`, `|=`, `^=`, `<<=`, `>>=`) follow the same pattern. A compound assignment is a statement and ends with `;` (optional before `}`, [1.3.4](../001-lexical/103-statement-terminator.md#134-block-like-expression-statements)); it has no value and cannot appear in expression position (`let y = (x += 1);` does not parse).

### Desugaring and lvalue evaluation

Compound assignment desugars as follows. If the type implements the `OpAssign` trait, `a += b` calls it; otherwise it is `a = a + b` ([stdlib 2.4.1](../../refs-standard-libraries/200-core/204-ops-assign.md)). The same holds for every operator.

```
lvalue OP= rhs       →   OpAssign::op_assign(ref lvalue, rhs)   // if implemented
                     →   lvalue = (lvalue OP rhs)               // otherwise
```

The compiler guarantees that the lvalue is not evaluated twice.

```rok
arr[expensive_index()] += 1;
// expensive_index() is called only once
```

If the lvalue is a plain variable or field, it is referenced twice as is; if it contains a call or computation, that part is bound to a temporary and evaluated once. This guarantee prevents an index or method with side effects from being called twice unintentionally.

### Trait mapping

Each compound assignment maps to a separate **Assign** trait family.

```
x += y       →   <typeof(x) as AddAssign<typeof(y)>>::add_assign(ref x, y)
x -= y       →   <typeof(x) as SubAssign<typeof(y)>>::sub_assign(ref x, y)
```

(There are `AddAssign`, `SubAssign`, `MulAssign`, `DivAssign`, ... and so on.)

These traits may be implemented as in-place updates, allowing a more efficient implementation than `x = x + y`. For example, appending to a large collection can happen without creating a new collection.

Implementing `AddAssign` is optional: without it, `x += y` falls back to `x = x + y`. `x + y` itself always needs `Add`.

### Signature

Signatures such as `AddAssign` have the following shape.

```rok
trait AddAssign<Rhs = Self> {
    #[infix("+=")]
    fn add_assign(ref self, rhs: Rhs);
}
```

Like other operators, the `*Assign` traits are registered through `#[infix("op=")]` ([3.1](301-operator-dispatch.md)). `ref self` refers to the lvalue's slot; there is no return type and the result is unit. The exact meaning of `ref self` and of updating the lvalue slot is defined in the function specification.

## 3.4.4. Chained assignment

C- and Python-style chained assignment (`a = b = c = 1`) is not allowed in rok.

```
a = b = c = 1;             // ERROR
```

Because the result of an assignment is unit, `b = c = 1` evaluates to unit and the whole becomes `a = (unit)`, which is meaningless. Write the intended meaning as separate assignments:

```rok
c = 1; b = 1; a = 1;
```

## 3.4.5. Assignment inside an expression

A simple assignment can be placed in expression position, where its result is unit, but there is little value in doing so.

```rok
let x = (y = 1);           // x = unit (nearly meaningless)
```

This pattern is discouraged, and tools or linters may warn about it.

Outside statement position, an assignment is written in parentheses `(y = 1)` or as a match arm body. An unparenthesized `let x = y = 1;` is not supported. Compound assignment has no expression form.
