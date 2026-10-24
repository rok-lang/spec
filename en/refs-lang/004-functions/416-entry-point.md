# 4.16. Entry point

This section defines what code runs, and in what order, when an executable package starts. It is the place that [source units](../000-meta/003-source-units.md) left open by saying "the exact form of the entry point is defined in the function specification".

## 4.16.1. Entry module: `main.rok`

The root file `src/main.rok` of an executable package ([0.3.1](../000-meta/003-source-units.md#executable-package-srcmainrok)) is the **entry module**. The entry module provides the entry point in **exactly one** of the following two forms.

| Form                  | Entry point                                          | Use                              |
|-----------------------|------------------------------------------------------|----------------------------------|
| explicit `fn main`    | the user-declared `fn main` function                 | ordinary applications            |
| top-level statements  | an **implicit `main`** whose body is the top-level statements | scripts, examples, short programs |

```ebnf
main_file   ::= item* fn_main item*            // form 1: explicit main
              | (item | top_stmt)*             // form 2: top-level statements (no fn main)
top_stmt    ::= let_stmt | let_else_stmt | expr ";" | block_like_expr ";"?   // 3.9.1
module_file ::= item*                          // lib.rok and every module file other than main.rok
```

`item` is an [item in a module](../005-modules/504-modules.md#547-items-within-a-module) (`fn`, `struct`, `enum`, `trait`, `impl`, `const`, `static`, `mod`, `use`, etc.), and `let_stmt` / `let_else_stmt` / `expr ";"` are the same as the statements of [block expressions](../003-expressions/309-block-expressions.md). Control flow (`if`, `match`, `for`, `while`, `loop`, `try`) consists of expressions and is thus included in `expr ";"`. Unlike a function body, there is no final expression without a terminator (`tail_expr`) - the implicit `main` returns no value. A `main.rok` with only items and no statements (and no `fn main`) gets an empty implicit `main`, except in a package that exports WIT: that package is a reactor with no entry point ([stdlib 1.8.2](../../refs-standard-libraries/100-bare/108-wit.md#exports)).

> [!IMPORTANT]
> If a single `main.rok` contains **both** top-level statements and a `fn main` declaration, it is a **compile error**. This rules out any situation where the reader must guess which of two entry points runs.

```rok
// main.rok - ERROR
fn main() { run(); }
run();                      // ERROR: top-level statement in a file with fn main
```

## 4.16.2. Meaning of top-level statements: implicit `main`

The top-level statements of a `main.rok` without `fn main` are gathered **in source order** into one function body, which becomes the body of a compiler-synthesized implicit `main`. Items are not executed and are left out of the body; even when items and statements are interleaved, only the relative order among statements matters.

```rok
// main.rok
let names = ["ada", "grace"];
for n in names {
    greet(n);               // greet is declared below - visible regardless of order
}

fn greet(name: str) {
    println(f"hello, {name}");
}

let total = names.len();
println(f"{total} greeted");
```

The file above means the same as:

```rok
fn greet(name: str) {
    println(f"hello, {name}");
}

fn main() throws {          // form of the implicit main - see 4.16.4
    let names = ["ada", "grace"];
    for n in names {
        greet(n);
    }
    let total = names.len();
    println(f"{total} greeted");
}
```

The following rules follow from this equivalence.

- **Items are visible regardless of order.** Items in `main.rok` belong to module scope like ordinary module items, so a top-level statement may refer to a `fn` / `struct` etc. declared after it.
- **A top-level `let` is a local variable of the implicit `main`.** It is not a global variable and is visible only in top-level statements after its declaration (the block scope rules of [variable declarations](../005-modules/501-variable-declaration.md) apply as-is).
- **A top-level `let` is not visible inside item bodies.** In the example above, if the body of `greet` referred to `names` or `total`, it would be a name resolution error. Capturing the local variable in a closure is possible (`let shout = |s| println(f"{s}! ({total})");`) - the closure is a statement and is created inside the implicit `main` body.

> [!NOTE]
> **Why top-level `let` is not visible to items.** Items exist regardless of order, but a `let` gets its value only at run time. If `fn greet` could see a top-level `let`, the meaning would have to be defined for the case where `greet` is called **before** that `let` (e.g., calling `greet()` on the first line), which amounts to bringing "global initialization order" semantics into the language. rok's principle is to have no global initialization order (4.16.6), so top-level `let` is restricted to local variables of the implicit `main`. State that must be shared between items is passed as parameters or declared `static`.

## 4.16.3. Signature of an explicit `main`

An explicit `fn main` has no parameters and no generic parameters, and must have one of the following forms. Any other form is a compile error.

| Signature                         | Exit code on normal exit | Notes                                  |
|-----------------------------------|--------------------------|----------------------------------------|
| `fn main()`                       | `0`                      |                                        |
| `fn main() -> i32`                | the return value         | the return value is the process exit code |
| `fn main() throws` / `throws (E)` | `0`                      | unhandled exception → 4.16.4           |
| the three forms above with `async` | same as above           | 4.16.5                                 |

There is no `main` returning `Result`, and no `main` combining `throws` with `-> i32`. If a specific exit code is needed on error, call [`rok::std::env::exit(code)`](../../refs-standard-libraries/300-std/306-env.md) in a `catch`.

Command-line arguments and environment variables are obtained through the standard library [`rok::std::env`](../../refs-standard-libraries/300-std/306-env.md), not through parameters. The `unsafe` and `yield` modifiers cannot be applied to `main`. Visibility (`pub`) does not affect whether it is the entry point.

A `fn main` declared in `lib.rok` or an ordinary module is not an entry point; it is just an ordinary function named `main`.

## 4.16.4. Exit codes and unhandled exceptions

The process exit code is determined by how the entry point finishes.

- **Normal exit**: `0` or the returned `i32`, per the table in 4.16.3.
- **Unhandled exception**: if an error propagates out of the entry point in a `throws` main, the runtime prints the exception value to standard error (stderr) and exits with code **`1`**. The output format is `Display` if the value implements [`Display`](../../refs-standard-libraries/200-core/210-fmt.md), otherwise `Debug`, and the type name if it has neither (the [`Error`](../../refs-standard-libraries/200-core/214-error.md) trait requires `Display`, so ordinary error types print with `Display`). The next line gives the location where the error left `main`, as `  --> path:line:col`.
- **Panic**: panics and fatal errors are not caught by `catch`; the runtime prints the message to standard error and exits with code **`101`** ([4.14.13](414-throws.md#41413-panics-and-fatal-errors)).
- **Explicit exit**: [`rok::std::env::exit(code)`](../../refs-standard-libraries/300-std/306-env.md) terminates the process immediately from any point. This path does not run `finally` or `Drop`.

The **implicit `main`** is treated as an untyped `fn main() throws` (error type `any Error`) (or `async fn main() throws` if the body contains `await`, 4.16.5). Errors from throwing calls marked with `try` in top-level statements may therefore propagate out of the entry point without a separate `catch`, and an unhandled exception yields exit code `1` per the rules above. If execution runs to the end normally, the exit code is `0`.

On `wasm32-wasi` (a preview 2 component), `wasi:cli/exit` carries only success or failure, so any nonzero exit code (including `1` and `101`) reaches the host as failure (status 1). `wasm32-wasi-threads` (preview 1) carries the full code, except that hosts may turn a `proc_exit` of 126 or more into failure (status 1). A trap such as `abort` exits with 134.

> [!IMPORTANT]
> **Top-level `return` is not allowed.** A `return` at the top level of `main.rok` is a compile error. If the implicit `main` needs a nonzero exit code, call `env::exit(code)`; to express the exit code as a return value, use the explicit `fn main() -> i32` form. Allowing top-level `return` would add rules only for the implicit `main`, such as "what type of `return` is allowed" and "what does a `return` inside a top-level block return", so splitting the entry point into two forms is simpler.

## 4.16.5. Async `main`

`async fn main()` runs as the top-level task on the runtime [scheduler](../007-runtime/706-scheduler.md). The platform entry wrapper initializes the runtime, hands the Future created by `main` to the scheduler, waits until that Future completes, and then determines the exit code by the rules of 4.16.4 (the same behavior as [`task::block_on`](../../refs-standard-libraries/200-core/220-future.md)). Other tasks still running when `main` finishes are not waited for.

The implicit `main` becomes `async fn main() throws` when `await` appears directly in a top-level statement (not inside another `async` block or closure); otherwise it is a synchronous `fn main() throws`.

```rok
// main.rok - implicit async main
let body = await http::get("https://example.com");
println(body.len());
```

## 4.16.6. Modules are definition-only

Top-level statements are allowed **only in the entry module `main.rok`**. `lib.rok` and every module file other than `main.rok` (and inline `mod { ... }` bodies) may contain only items (`module_file ::= item*`); a top-level statement there is a **compile error**. For statements in an inline `mod x { stmt }` body, a dedicated diagnostic pointing to this section is issued.

As a result, **importing a module (`mod`, `use`) runs no code.** This deliberately differs from Python. In Python, `import` runs a module file's top-level code, so a file used both as a library and as a script must separate the two uses with an `if __name__ == "__main__":` guard. rok needs no such guard: code run as a script exists only in `main.rok`, and every other file causes nothing at import time.

The rationale for this decision:

- **AOT is the reference for meaning** ([0.1.4](../000-meta/001-execution-modes.md#014-semantic-baseline)). Supporting import-time execution in AOT output would require synthesizing per-module initialization functions and their call order at link time. Definition-only modules need no such machinery.
- **No initialization order semantics.** Import-time code across modules would require defining "which module is initialized first" and "what is visible during cyclic imports". rok ensures these questions never arise. Initial values of global state are determined at compile time (4.16.7).
- **Caching and incremental builds.** Since a module's meaning is determined by its definitions alone, caches based on content hashes ([0.3.3](../000-meta/003-source-units.md#033-package-identifier)) and incremental builds need not account for execution side effects.
- **Package kind is determined by file presence** ([0.3.4](../000-meta/003-source-units.md#034-the-manifest-rokrokon-is-optional)). Executable code lives exactly in `src/main.rok`, so the answers to "is this an executable package?" and "where is the executable code?" always coincide.

## 4.16.7. Global state

Module-level global state is declared only with [`static`](../005-modules/503-static.md), and its initial value must be a `const` expression evaluable at compile time ([5.3.9](../005-modules/503-static.md#539-const-restriction-on-initializers)). Lazy initialization, computed on first access at run time, is not a language feature but the responsibility of standard library wrappers. Program execution therefore always begins at the first statement of the entry point, and no user code runs before it.

## 4.16.8. Single-File execution

A single `.rok` file that does not belong to a package can be run directly.

```
rokc run hello.rok
rokc build hello.rok
```

If the file given as an argument is not in the `src/` of a package, `rokc` constructs an **anonymous executable package** consisting of that file alone and treats the file as the package's `main.rok`. The file name has no meaning, and the entry point rules (4.16.1 - 4.16.5) apply exactly as for `main.rok`. A `.rok` file beside a package's `src/`, such as a script in its root, is also run this way. An anonymous package may have file submodules: `mod foo;` loads `foo.rok` beside the file (its own submodules in `foo/`), exactly as in a package's `src/` ([5.4.4](../005-modules/504-modules.md#544-package-root)). Only source without a file (standard input, a REPL session) cannot; it uses inline modules. Dependencies, if needed, are declared in an inline manifest block at the head of the file and of any of its submodule files ([9.10.7](../009-tools/910-rok-frontend.md#9107-inline-manifest-for-single-file-scripts)). The detailed command forms are defined in [`rokc` CLI](../009-tools/901-rokc-cli.md).

## 4.16.9. No entry point attribute

rok has **no** attribute for designating the entry point (e.g., `#[main]`). The root file `src/main.rok` already designates the entry module, and the name `main` (or top-level statements) designates the entry function, so there is no second means of expressing the same information. Keeping a single mechanism for determining the entry point means "which function runs" can be determined from the file structure alone. `main` is not a reserved word, and the language does not reserve the name `#[main]` either.

## 4.16.10. Relationship to JIT / REPL

REPL input and Jupyter cells ([JIT cells](../009-tools/907-jit-cells.md)) mean the same as **appending top-level statements and items, in order, to the end of an implicit `main.rok`**. A cell's `let` is a local variable of the implicit `main` and is visible in later cells; a cell's `fn` / `struct` become module items. The rules of 4.16.2 apply as-is, so the body of a `fn` declared in a cell cannot refer to a top-level `let` from an earlier cell (closure capture is possible).

Thanks to this equivalence, the principle of [0.1.1](../000-meta/001-execution-modes.md#the-semantics-are-the-same) that "JIT and AOT share the same semantics" also holds for the entry point. A file made by concatenating a notebook's cells in order is a `main.rok` that produces the same result. The following three, however, are **session operations** that exist only in interactive sessions and are not extensions of the language semantics:

- Re-declaring the same name with `let` at the top level of a cell ([9.7.3](../009-tools/907-jit-cells.md#redeclaration-of-the-same-name)). This creates a new binding.
- Deleting and re-running cells ([9.7.4](../009-tools/907-jit-cells.md#974-undo--cell-re-execution)).
- Automatic display of the value of a final expression without a terminator ([9.7.7](../009-tools/907-jit-cells.md#977-visualization--output)). In a file, this is the same as appending `;` to that expression.
