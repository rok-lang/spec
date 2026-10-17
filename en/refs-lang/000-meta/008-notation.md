# 0.8. Notation

This section summarizes the meta-notation conventions used throughout this specification. Chapters 1 through 9 all follow these conventions, so it is worth consulting this section before reading the body.

## 0.8.1. Code / identifier notation

The following table shows the three commonly used forms, their meaning, and examples.

| Notation           | Meaning                                            | Example                    |
|--------------------|----------------------------------------------------|----------------------------|
| `` ` `` (backtick) | Keywords, identifiers, tokens, type names, code fragments | `` `let` ``, `` `i32` `` |
| Fence (` ``` `)    | Multi-line code block — language name given        | ` ```rok ` or ` ```ebnf`   |
| Inline code        | Short code fragment                                | `` `x + y` ``              |

A keyword referred to in running text is always wrapped in backticks, which sets it apart visually from the surrounding prose.

## 0.8.2. Emphasis

Emphasis is not mere visual decoration; each form has a fixed purpose, as listed below.

| Notation         | Meaning                              | Used for                                         |
|------------------|--------------------------------------|--------------------------------------------------|
| `*text*`         | **Italic** — conceptual emphasis     | Introducing new terms, emphasizing meaning       |
| `**text**`       | **Bold** — warnings, key decisions   | "this is a **compile error**", "**rejected outright**" |
| `> [!NOTE]`      | GitHub-style info box                | Side notes, informal guidance                    |
| `> [!WARNING]`   | GitHub-style warning box             | Pitfalls, common misconceptions                  |
| `> [!IMPORTANT]` | GitHub-style important box           | Core invariants, rules that must be followed     |

## 0.8.3. Grammar definitions — BNF variant

Grammar in this specification is written in an EBNF variant. The metacharacters used are as follows.

| Meta                | Meaning                  | Example                        |
|---------------------|--------------------------|--------------------------------|
| `name ::= rhs`      | Definition               | `path ::= ident ("::" ident)*` |
| `\|`                 | Alternative              | `bool ::= "true" \| "false"`    |
| `?`                 | Zero or one (optional)   | `param ::= ident (":" type)?`  |
| `*`                 | Zero or more             | `args ::= expr ("," expr)*`    |
| `+`                 | One or more              | `digits ::= digit+`            |
| `( ... )`           | Grouping                 | `(a \| b) c`                    |
| `"..."`             | Literal text             | `kw_let ::= "let"`             |
| Nonterminal (italic) | Reference to another definition | `expr ::= literal \| call` |

Grammar blocks are written with ` ```ebnf ` or a plain fence. For example, the function declaration grammar can be written as follows.

```ebnf
fn_decl     ::= "fn" ident "(" params? ")" ret_ty? block
params      ::= param ("," param)* ","?
param       ::= ident (":" type_expr)? default_value?
ret_ty      ::= "->" type_expr
default_value ::= "=" expr
```

Note one important point: grammar is only a **reference definition of form**. Semantics are defined in the body prose, so grammar alone cannot determine the meaning of a program.

## 0.8.4. Cross-reference

References to other sections are kept consistent. A reference to another section gives both the **stable section number** and a **relative path link**.

```markdown
[3.2 Precedence and associativity](../003-expressions/302-precedence-and-associativity.md)
```

An informal reference mentioned in passing within the body may use the section number alone.

> See the niche rules in spec 2.7.4.

In this form the meaning is preserved even if a file is renamed. File paths are updated by automated tooling.

## 0.8.5. Code example conventions

Code examples follow fixed conventions as well. Below is the most basic form.

```rok
// short one-line comment
fn example() -> i32 {
    let x = 42;        // x is an implicit generic — simplified example
    return x;
}
```

The rules for writing code examples are as follows. First, every code example must be part of a **compilable** rok program. Second, a snippet is marked with `// ...` and its context is stated in the body. Third, an intentionally incorrect example must be stated clearly in the body and accompanied by a `// ERROR: ...` comment or prose explanation. Finally, execution results or output are marked with a `//→ ` comment.

The following example applies the last rule.

```rok
let r = 1 + 2;        //→ 3
let s = "hello";      //→ str
```

## 0.8.6. Stable section numbers

Each section has a permanent `<chapter>.<section>(.<sub>)*` number. The number does not change even if a file is renamed or moved, so it serves as a stable anchor for references. The numbering scheme is as follows.

| Form      | Example   | Meaning                              |
|-----------|-----------|--------------------------------------|
| `C.S`     | `2.7`     | The S-th section of chapter C        |
| `C.S.s`   | `2.7.4`   | The s-th subsection of C.S           |
| `C.S.s.x` | `2.7.4.1` | Deeper subsection (rare)             |

Cross-references are always made with this number. The file path (`002-types/207-...md`) is merely added for the convenience of IDEs and GitHub; the substance of a reference is always the section number.

## 0.8.7. Limits of the notation

This notation defines only **reference forms**. Exactly how the compiler parses input and assigns meaning depends on the body prose, the code examples and the compiler implementation. That is, **BNF alone does not determine all meaning**. Authority over meaning always rests with the prose.
