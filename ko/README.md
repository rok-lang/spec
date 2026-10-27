# Rok 언어 사양

[English](../en/README.md) · **한국어**

> **상태** - 활발한 초안. 사양은 영어 ([`en/`](../en/README.md), 기본) 와 한국어 (`ko/`) 두 판으로 나옵니다. 두 판은 같은 내용을 같은 절 번호로 다룹니다.

Rok 은 프로그래밍 언어이며, 이 저장소는 그 언어의 형식 사양 (formal specification) 입니다. 어휘 구조와 타입 시스템에서 출발해 표현식, 런타임 모델, ABI, 그리고 사양에 부속하는 도구까지 - 언어를 구성하는 모든 층위를 한 자리에서 정의합니다. 컴파일러나 표준 라이브러리의 실제 구현은 별도의 저장소에서 다루며, 여기에 모이는 것은 오로지 "무엇이 Rok 인가" 라는 질문에 대한 규범적 답입니다.

설계와 유지보수는 [한다민 (Han Damin)](https://github.com/miniex) (`@miniex`) 이 맡고 있습니다.

| 자리                 | 위치                                                    |
|----------------------|---------------------------------------------------------|
| GitHub 조직          | [rok-lang](https://github.com/rok-lang)                 |
| 사양 저장소          | <https://github.com/rok-lang/spec>                      |
| 웹사이트             | <https://rok-lang.org>                                  |
| 문서                 | <https://docs.rok-lang.org>                             |
| stone 레지스트리     | <https://rokstones.io>                                  |
| 레지스트리 index     | <https://github.com/rok-lang/rokstones.io-index>        |

## Rok 이 존재하는 이유

Rok 은 저자가 직접 쓰고 싶은 코드 한 가지에서 출발한 언어입니다. 특정한 사용자층을 겨냥하지도, 학술적인 입장을 대변하지도 않습니다. 모든 설계 결정의 한가운데에는 "저자가 하고 싶은 모든 것을 할 수 있는 언어 하나" 라는 단순한 동기가 놓여 있을 뿐이며, 이 한 문장이 곧 Rok 의 전부라고 해도 과언이 아닙니다.

## 철학

이렇게 개인적인 동기에서 출발한 언어가 일관성을 유지하려면, 결정을 내릴 때 매번 의지할 수 있는 원칙이 필요합니다. Rok 의 설계는 두 개의 슬로건이 이끌어 갑니다.

> [!IMPORTANT]
> **"언어는 작게, 라이브러리는 풍부하게."**

첫 번째 원칙은 사양의 크기에 대한 약속입니다. Rok 의 사양은 가능한 한 적게 정의합니다. 숫자 의미론, 형 변환, 컬렉션 (collection), 입출력, 동시성처럼 언어의 *느낌* 을 결정짓는 거의 모든 요소는 표준 라이브러리에 자리 잡습니다. 코어를 작게 유지하면 언어를 진화시키기도, 검증하기도, 가르치기도 쉬워지며, 그 대신 도메인 (domain) 의 무게는 풍부한 라이브러리가 받아 줍니다.

> [!IMPORTANT]
> **"규칙이 명확할수록 그 안에서 자유로워집니다."**

두 번째 원칙은 자유와 제약 사이의 관계를 다룹니다. Rok 은 제약을 회피하지 않습니다. 정적 타입, 닫힌 합 타입, 트레잇-as-타입 (trait-as-type) 금지, 묵시적 승급 금지, AOT-only 컴파일 - 이 모든 결정은 사용자가 선택할 수 있는 길을 의도적으로 좁힙니다. 그러나 그렇게 좁아진 공간 안에서 코드는 비로소 예측 가능해지고 깊이 통제할 수 있는 대상이 됩니다.

다시 말해, 통제와 자유의 경계가 분명합니다. 타입, 디스패치 (dispatch), 메모리 모델, 평가 순서처럼 한 번 정해 두면 매 줄마다 다시 결정할 필요가 없는 것들은 언어가 책임지고 한꺼번에 정합니다. 반대로 라이브러리 설계, 도메인 모델링, 알고리듬, 코드 구조처럼 사람마다 다를 수밖에 없는 영역은 사용자의 손에 맡겨 두고, 메타 시스템이 언어 자체의 자기 확장을 허용함으로써 의미를 라이브러리가 결정할 수 있도록 합니다.

따라서 자유와 통제는 대립이 아닙니다. 잘 통제된 코어 위에서만 자유로운 영역이 안전하게 살아남을 수 있다는 것이 Rok 의 입장이며, 두 슬로건에 대한 본격적인 논의는 [`refs-lang/000-meta/`](refs-lang/000-meta/) 에서 이어집니다.

## 레이아웃

이러한 철학을 사양 문서로 풀어내기 위해 저장소는 정해진 모양으로 정리됩니다. 최상위 디렉토리는 언어별로 나뉘며, 각 언어 디렉토리 안에는 두 종류의 참조 (reference) 가 함께 들어갑니다.

- [`refs-lang/`](refs-lang/) - 언어 참조. 어휘, 타입, 표현식, 런타임 모델, ABI, 도구를 다룹니다.
- [`refs-standard-libraries/`](refs-standard-libraries/) - 표준 라이브러리 참조.

모든 판이 이 구조를 따릅니다. `../en/refs-lang/`, `ko/refs-lang/` 처럼 같은 파일과 같은 절 번호를 가집니다.

## 판

사양은 두 언어 판으로 나옵니다.

- **English** ([`en/`](../en/README.md)) - 기본 판.
- **한국어** (`ko/`) - 이 판.

두 판은 파일 구성과 절 번호 (`3.11.4`) 가 같으므로, 절 번호는 어느 판에서나 같은 규칙을 가리킵니다. 두 판이 어긋나면 영어 본문이 기준이며, 그 차이는 양쪽에서 고칠 버그입니다.

## 읽는 순서

사양은 0장부터 9장까지 차례로 읽었을 때 가장 자연스럽게 이해되도록 짜여 있습니다. 새로 등장하는 개념은 반드시 그 시점까지 본 개념만으로 설명되며, 이 invariant 는 어느 장을 펼쳐도 깨지지 않습니다. cross-reference 도 같은 원칙을 따라 **backward-only** 로만 이어집니다. 따라서 임의의 장에 들어가더라도 그 위에 쌓인 개념을 다시 거슬러 올라갈 일은 없으며, 처음 시작하는 독자는 [`refs-lang/README.md`](refs-lang/README.md) 부터 펼치면 됩니다.

## 표기 규약

본문, 표, 인용문은 모두 한국어로 적습니다. 다만 코드 펜스 안에 들어가는 ASCII 또는 box-drawing 다이어그램만은 예외적으로 영어 라벨을 사용합니다. GitHub 의 monospace 폰트가 한글을 ASCII 폭의 정확히 두 배가 아니라 1.5-1.7 배로 렌더링하기 때문에, 한글 라벨이 섞인 다이어그램은 정렬이 어긋날 수밖에 없기 때문입니다. 그래서 다이어그램 자체는 영어로 그리고, 그 바로 아래에 한국어 인용문으로 라벨을 풀어 설명하는 방식을 택했습니다. 코드 예제는 번역된 주석, 부수적인 문자열, 그리고 사용자가 채울 부분을 나타내는 문법 자리표시자 (메타변수) 를 제외하면 두 판에서 동일합니다.

## 용어

아래 용어는 한국어 판 전체에서 일관되게 쓰입니다 (번역자를 위해 영어 판의 대응어를 함께 적습니다). 핵심 용어의 정의는 [용어집](refs-lang/000-meta/007-glossary.md) 에 있습니다.

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

이미 영어인 이름은 두 판 모두 그대로 둡니다. ROKON, `rok.rokon`, `rok.lock`, `rokc`, `rok`, `rokls`, `rokr`, ROKHIR, HIR, MIR, MLIR, AOT, JIT, GC, Immix, LOS, TLAB, shadow stack, `scan_table`, niche optimization, sticky mark-bit, work-stealing, ABI, FFI, toolchain, sysroot.

## 라이선스

본 사양은 [Creative Commons Attribution 4.0 International (CC BY 4.0)](../LICENSE) 으로 배포됩니다.

Copyright © 2026 [Han Damin](https://github.com/miniex) &lt;miniex@daminstudio.net&gt;

출처를 표기하기만 하면 텍스트의 공유, 각색, 번역이 자유롭게 허용됩니다. 라이선스 전문은 [LICENSE](../LICENSE) 에서, 사용자용 요약은 <https://creativecommons.org/licenses/by/4.0/deed.ko> 에서 확인할 수 있습니다.

## 상표 (trademark)

"Rok" 은 본 프로젝트의 이름이며 정본 사양을 가리키는 식별자입니다. CC BY 4.0 은 어디까지나 **텍스트** 만을 다루는 라이선스라서, 출처 표기만 지키면 텍스트 자체는 얼마든지 복사하고 번역하고 각색해도 됩니다. 그러나 포크된 사양, 대체 사양, 경쟁 구현에까지 "Rok" 이라는 이름을 그대로 가져다 쓰는 일은 삼가 주십시오. 정본 프로젝트와 혼동되지 않도록 다른 이름을 사용해야 합니다.

이는 등록 상표가 아니라 어디까지나 **요청** 입니다. 다만 프로젝트가 정식 보호가 필요할 만큼 자라난다면 그때는 등록도 함께 고려할 예정입니다.
