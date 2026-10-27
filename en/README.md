# The Rok language specification

**English** · [한국어](../ko/README.md)

> **Status** - active draft. The specification is published in two editions: English (`en/`, the default) and Korean ([`ko/`](../ko/README.md)). Both cover the same content with the same section numbers.

Rok is a programming language, and this repository is its formal specification. Starting from lexical structure and the type system, it defines every layer of the language in one place: expressions, the runtime model, the ABI, and the tools that accompany the specification. The compiler and the standard library implementation live in a separate repository; this one holds only the normative answer to the question "what is Rok".

Designed and maintained by [Han Damin](https://github.com/miniex) (`@miniex`).

| What                 | Where                                                   |
|----------------------|---------------------------------------------------------|
| GitHub organization  | [rok-lang](https://github.com/rok-lang)                 |
| Spec repository      | <https://github.com/rok-lang/spec>                      |
| Website              | <https://rok-lang.org>                                  |
| Docs                 | <https://docs.rok-lang.org>                             |
| stone registry       | <https://rokstones.io>                                  |
| Registry index       | <https://github.com/rok-lang/rokstones.io-index>        |

## Why Rok exists

Rok started from one kind of code its author wants to write. It does not target a particular audience and does not argue an academic position. At the center of every design decision is one simple motive: "a single language that can do everything the author wants to do". That sentence is, more or less, all of rok.

## Philosophy

A language born of a personal motive needs principles it can fall back on at every decision to stay coherent. Two slogans guide the design of rok.

> [!IMPORTANT]
> **"A small language, rich libraries."**

The first principle is a promise about the size of the specification. The Rok specification defines as little as possible. Almost everything that shapes how the language *feels* - numeric semantics, conversions, collections, I/O, concurrency - lives in the standard library. A small core is easier to evolve, verify and teach; the weight of each domain is carried by a rich library instead.

> [!IMPORTANT]
> **"The clearer the rules, the freer you are within them."**

The second principle concerns the relation between freedom and constraint. Rok does not avoid constraints. Static types, closed sum types, the trait-as-type ban, no implicit promotion, AOT-only compilation - each of these decisions deliberately narrows the paths a user can take. Within that narrower space, code becomes predictable and can be controlled in depth.

The line between control and freedom is clear. What can be decided once and need not be decided again on every line - types, dispatch, the memory model, evaluation order - the language decides, all at once. What necessarily differs from person to person - library design, domain modeling, algorithms, code structure - is left to the user, and the meta-system lets the language extend itself so that libraries can decide meaning.

Freedom and control are therefore not opposites. Rok's position is that a free area survives safely only on top of a well-controlled core. The full discussion of both slogans continues in [`refs-lang/000-meta/`](refs-lang/000-meta/).

## Layout

The repository has a fixed shape. Top-level directories are per language edition, and each edition holds two references:

- [`refs-lang/`](refs-lang/) - the language reference: lexical structure, types, expressions, the runtime model, the ABI, tools.
- [`refs-standard-libraries/`](refs-standard-libraries/) - the standard library reference.

Every edition follows this structure: `en/refs-lang/`, `ko/refs-lang/`, and so on, with the same files and the same section numbers.

## Editions

The specification exists in two language editions:

- **English** (`en/`) - the default edition.
- **한국어** ([`ko/`](../ko/README.md)) - Korean.

Both editions use the same file layout and section numbering (`3.11.4`), so a section number names the same rule in either. If the two editions disagree, the English text is the reference, and the discrepancy is a bug to be fixed in both.

## Reading order

The specification is written to be read from chapter 0 to chapter 9. A new concept is always explained only in terms of concepts introduced up to that point, and this invariant holds in every chapter. Cross-references follow the same rule and point **backward only**. Whichever chapter you open, you never need to climb back above it to find a concept; a first-time reader should start at [`refs-lang/README.md`](refs-lang/README.md).

## Conventions

ASCII and box-drawing diagrams inside code fences use English labels, explained in the text below them. Code examples are identical in both editions apart from translated comments, incidental strings, and the grammar placeholders (metavariables) that stand for user-written parts.

## Terminology

The terms below are used consistently across the English edition (the Korean edition's counterparts are listed for translators). The [glossary](refs-lang/000-meta/007-glossary.md) defines the core terms.

| English | 한국어 |
|---|---|
| specification, chapter, section | 사양, 장, 절 |
| language reference / standard library reference | 언어 참조 / 표준 라이브러리 참조 |
| sum type | 합 타입 |
| variant | 변형 |
| trait; trait-as-type ban | 트레잇; 트레잇-as-타입 금지 |
| associated type / constant | 연관 타입 / 상수 |
| monomorphization; concrete | 단형화; 구체 |
| slot; `any` slot; lang slot ("magic") | 슬롯; any 슬롯; lang slot (매직) |
| implicit promotion / conversion | 묵시적 승급 / 암묵 변환 |
| conversion / cast | 형 변환 / 캐스트 |
| expression / statement; statement terminator | 표현식(식) / 문장; 문장 종결자 |
| block; body | 블록; 본체 |
| arm (of `match`) / branch | 가지 / 분기 |
| pattern; binding; destructuring (assignment) | 패턴; 바인딩; 분해 (대입) |
| optional chaining / coalescing | 옵셔널 체이닝 / 병합 |
| broadcast | 브로드캐스트 |
| literal; identifier; keyword; reserved word | 리터럴; 식별자; 키워드; 예약어 |
| boolean; sentinel; floating-point; integer | 진리값; 센티넬; 부동소수; 정수 |
| punctuation; trailing comma | 구두점; 트레일링 쉼표 |
| parameter / argument | 매개변수 / 인자 |
| closure; capture; coroutine; async | 클로저; 캡처; 코루틴; 비동기 |
| entry point | 진입점 |
| module; visibility; path; item | 모듈; 가시성; 경로; 항목 |
| package; stone; registry | 패키지; stone; 레지스트리 |
| manifest; lockfile; environment; dependency; pin | 매니페스트; lockfile; 환경; 의존성; 고정 |
| meta-system; attribute; template tokens | 메타 시스템; 속성; 템플릿 토큰 |
| runtime; object; root; barrier; generation | 런타임; 객체; 루트; 배리어; 세대 |
| lowering; dialect; symbol | 하강 (lowering); 다이얼렉트; 심볼 |
| error; "is an error" / "is a compile error"; diagnostic | 에러; "에러입니다" / "컴파일 에러"; 진단 |
| raw pointer; collection; iterator | 원시 포인터; 컬렉션; 반복자 |
| evaluation order; incremental build | 평가 순서; 증분 빌드 |

Names that are already English stay as they are in both editions: ROKON, `rok.rokon`, `rok.lock`, `rokc`, `rok`, `rokls`, `rokr`, ROKHIR, HIR, MIR, MLIR, AOT, JIT, GC, Immix, LOS, TLAB, shadow stack, `scan_table`, niche optimization, sticky mark-bit, work-stealing, ABI, FFI, toolchain, sysroot.

## License

This specification is distributed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](../LICENSE).

Copyright © 2026 [Han Damin](https://github.com/miniex) &lt;miniex@daminstudio.net&gt;

Sharing, adapting and translating the text are freely permitted as long as attribution is given. The full license is in [LICENSE](../LICENSE); a human-readable summary is at <https://creativecommons.org/licenses/by/4.0/>.

## Trademark

"Rok" is the name of this project and the identifier of the canonical specification. CC BY 4.0 covers the **text** only: with attribution, the text itself may be copied, translated and adapted freely. Please do not, however, use the name "Rok" for forked specifications, alternative specifications or competing implementations. Use a different name so they are not confused with the canonical project.

This is a **request**, not a registered trademark. If the project grows to need formal protection, registration will be considered.
