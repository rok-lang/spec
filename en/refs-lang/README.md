# rok language reference

This is the formal specification of the rok programming language. It defines the language's **syntax** and **semantics** and provides the normative baseline that compilers and runtimes must follow.

The reference date is 2026-04-30. This specification is the normative baseline for compiler and runtime implementations; the actual implementations of the compiler and the standard libraries live in separate repositories.

---

## Chapters

The rok specification consists of ten chapters, numbered 0 through 9. The chapters are not listed arbitrarily: they are arranged in dependency order so that each chapter's definitions use only concepts from earlier chapters. Chapter 0 (meta) comes first; the sequence then starts from lexical structure, moves through progressively more abstract areas, and ends with the execution environment and tooling.

| # | Chapter                                                       | Scope                                                                       |
|---|---------------------------------------------------------------|-----------------------------------------------------------------------------|
| 0 | [Meta - language / build model](000-meta/README.md)           | Execution modes, pipeline, standard library tiers, platforms, notation, glossary |
| 1 | [Lexical structure](001-lexical/README.md)                    | Identifiers, keywords, tokens, literals, strings, punctuation               |
| 2 | [Type system](002-types/README.md)                            | Primitives, sum / composite, generics, niche, conversion, paths             |
| 3 | [Expressions](003-expressions/README.md)                      | Operators, pattern matching, control flow, optional chaining, throw         |
| 4 | [Functions](004-functions/README.md)                          | Parameters, closures, trailing closures, coroutines, async/await            |
| 5 | [Variables / modules / visibility / packages](005-modules/README.md) | let, const, static, mod, use, visibility, paths, re-exports          |
| 6 | [Meta-system](006-meta-system/README.md)                      | Attributes, lang/literal/keyword magic, meta blocks, reflection             |
| 7 | [Runtime model](007-runtime/README.md)                        | Object layout, GC, scheduler, coroutines, safepoints                        |
| 8 | [Runtime ABI / MLIR lowering](008-abi/README.md)              | Headers, scan tables, TLS, calling convention, MLIR dialects                |
| 9 | [Execution / tools](009-tools/README.md)                      | rokc CLI, AOT pipeline, .a packages, ROKHIR, JIT, Jupyter                     |

Chapter 0 collects the build model, platform assumptions and notation that every later chapter presupposes; reading it once means the conventions repeated in the body need not be tracked down again. Chapter 1, lexical structure, comes next for a simple reason: types and expressions can take form only once the shape of tokens is defined. Chapters 2 (types), 3 (expressions) and 4 (functions) then introduce, in turn, the core structures needed to create and manipulate values inside a single module.

Modules, visibility and packages appear only in Chapter 5 by design: file boundaries and encapsulation can be discussed only after the semantics within a single file are fully defined. Chapter 6, the meta-system, presupposes all of Chapters 1–5. Attributes, `lang` / `literal` / `keyword` magic, and reflection operate *on* lexical structure, types, expressions, functions and modules, so their meaning is clear only once all of those targets are defined.

Finally, Chapter 7 (runtime model), Chapter 8 (ABI and MLIR lowering) and Chapter 9 (execution and tools) cover the steps that turn the surface language into actual execution. Chapter 7 describes object layout, GC and the scheduler abstractly; Chapter 8 pins down which bits in memory and registers implement them; and Chapter 9 ties it all together and specifies the `rokc` CLI and the form of build outputs that users interact with.

The following appendices and references are cited frequently in the body.

- [Notation (BNF, cross-refs, code conventions)](000-meta/008-notation.md)
- [Glossary](000-meta/007-glossary.md)

---

## How to read

### First reading (0 → 9 in order)

This specification is written so that, read in order from Chapter 0 to Chapter 9, every newly introduced concept is explained using only concepts already seen. That is, the invariant holds that any chapter uses only concepts from chapters before it. A first-time reader should therefore follow the order without skipping; it is the shortest path.

Definitional dependencies in cross-references are backward-only: understanding the current document must not require a definition from a later document. For cross-cutting topics (method dispatch ↔ trait dispatch, arithmetic expressions ↔ operator magic, etc.), forward pointers may appear as supplementary references. Even then, the forward target must not be *required* to read the current document; it serves only as a pointer to additional context.

### As a reference (looking up a specific item)

The specification is also often used to find a specific item quickly rather than to learn. For this, every item has a stable section number (`<chapter>.<section>` or `<chapter>.<section>.<subsection>`), and cross-references always use this number. File paths may change over time and are not used in citations.

```
spec 2.7.4    →  002-types/207-niche-optimization.md (4th subsection)
spec 4.8      →  004-functions/408-trailing-closure.md
spec 6.10     →  006-meta-system/610-reflection.md
```

### Search

The whole specification is Markdown, so it can be searched directly with ripgrep or grep. Requiring no separate index or search tool is intentional: reading the specification and building it are handled with the same tools.

```bash
rg "trait" spec/en/refs-lang/
rg "::=" spec/en/refs-lang/                # search BNF definitions
rg "i32" spec/en/refs-lang/002-types/      # limit to one chapter
```

---

## Notational conventions

The conventions used consistently throughout the body are summarized below. Full definitions and exceptions are in [000-meta/008-notation.md](000-meta/008-notation.md).

| Notation                     | Meaning                                    |
|------------------------------|--------------------------------------------|
| `` `let` ``                  | Keyword / identifier (code font)           |
| `` `i32` ``                  | Type name (code font)                      |
| `*emphasis*`                 | Conceptual emphasis (italic)               |
| `**strong**`                 | Warning / key decision (bold)              |
| `> [!NOTE]` / `> [!WARNING]` | GitHub-style callout                       |
| `name ::= ...`               | BNF-style grammar definition               |
| `<chapter>.<section>`        | Stable cross-reference key (e.g. `2.7.4`)  |

Code font is used only for tokens that actually appear in the surface language; conceptual emphasis uses italics. Key decisions and points requiring attention are marked in bold.

---

## Directory / file naming

File and directory names are designed to serve as cross-reference keys themselves.

| Location | Form                                   | Example                          |
|----------|----------------------------------------|----------------------------------|
| Chapter  | `<NNN>-<descriptive-name>/`            | `002-types/`, `006-meta-system/` |
| Section  | `<NNN>-<name>.md`                      | `207-niche-optimization.md`      |
|          | (one digit chapter + two digit section) |                                 |
| README   | `README.md` in each chapter            | Chapter contents + guide         |

The first digit of the three-digit prefix is the chapter number and the last two digits are the section number. For example, `207` denotes "2.7", the 7th section of the type system chapter, and this number serves directly as the stable cross-reference key.

This naming scheme is a compromise between the descriptive file names and hierarchical chapter organization of the [Rust Reference](https://doc.rust-lang.org/reference/) and the numbered chapter / section scheme of the [Python Language Reference](https://docs.python.org/3/reference/). It satisfies two needs at once: numbers are friendlier to machines, names to humans.
