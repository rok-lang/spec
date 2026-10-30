# Rok 언어 사양

[English](../en/README.md) · **한국어**

> **상태**: 작업 중인 초안입니다. 사양은 영어판([`en/`](../en/README.md), 기본)과 한국어판(`ko/`) 두 가지로 나옵니다. 두 판은 같은 내용을 같은 절 번호로 다룹니다.

Rok은 프로그래밍 언어이고, 이 저장소는 Rok의 형식 사양(formal specification)입니다. 어휘 구조와 타입 시스템부터 표현식, 런타임 모델, ABI, 사양에 딸린 도구까지 언어의 모든 층을 한곳에서 정의합니다. 컴파일러와 표준 라이브러리 구현은 별도 저장소에 있습니다. 이 저장소에는 "Rok이란 무엇인가"라는 질문의 규범적 답만 담습니다.

설계와 유지보수는 [한다민(Han Damin)](https://github.com/miniex)(`@miniex`)이 맡고 있습니다.

| 자리                 | 위치                                                    |
|----------------------|---------------------------------------------------------|
| GitHub 조직          | [rok-lang](https://github.com/rok-lang)                 |
| 사양 저장소          | <https://github.com/rok-lang/spec>                      |
| 웹사이트             | <https://rok-lang.org>                                  |
| 문서                 | <https://docs.rok-lang.org>                             |
| stone 레지스트리     | <https://rokstones.io>                                  |
| 레지스트리 index     | <https://github.com/rok-lang/rokstones.io-index>        |

## Rok이 존재하는 이유

Rok은 저자가 직접 쓰고 싶은 코드 한 가지에서 출발한 언어입니다. 특정 사용자층을 겨냥하지 않고, 학술적 입장을 내세우지도 않습니다. 모든 설계 결정의 중심에는 "저자가 하고 싶은 것을 모두 할 수 있는 언어 하나"라는 단순한 동기가 있습니다. Rok은 거의 이 한 문장으로 설명됩니다.

## 철학

개인적인 동기에서 출발한 언어가 일관성을 지키려면 결정할 때마다 기댈 원칙이 필요합니다. Rok의 설계는 두 슬로건을 따릅니다.

> [!IMPORTANT]
> **"언어는 작게, 라이브러리는 풍부하게."**

첫 번째 원칙은 사양의 크기에 관한 약속입니다. Rok 사양은 되도록 적게 정의합니다. 숫자 의미론, 형 변환, 컬렉션(collection), 입출력, 동시성처럼 언어의 *느낌*을 좌우하는 요소는 거의 모두 표준 라이브러리에 있습니다. 코어가 작으면 언어를 발전시키기도, 검증하기도, 가르치기도 쉽습니다. 각 도메인(domain)의 무게는 대신 풍부한 라이브러리가 짊어집니다.

> [!IMPORTANT]
> **"규칙이 명확할수록 그 안에서 자유로워집니다."**

두 번째 원칙은 자유와 제약의 관계를 다룹니다. Rok은 제약을 피하지 않습니다. 정적 타입, 닫힌 합 타입, 트레잇-as-타입(trait-as-type) 금지, 묵시적 승급 금지, AOT 전용 컴파일은 모두 사용자가 고를 수 있는 길을 일부러 좁히는 결정입니다. 그렇게 좁아진 공간 안에서 코드는 예측할 수 있게 되고, 깊이 통제할 수 있게 됩니다.

통제와 자유의 경계는 분명합니다. 타입, 디스패치(dispatch), 메모리 모델, 평가 순서처럼 한 번 정하면 줄마다 다시 정할 필요가 없는 것은 언어가 한꺼번에 정합니다. 라이브러리 설계, 도메인 모델링, 알고리듬, 코드 구조처럼 사람마다 다를 수밖에 없는 영역은 사용자에게 맡깁니다. 그리고 메타 시스템으로 언어가 스스로를 확장할 수 있게 해서, 의미를 라이브러리가 정할 수 있게 합니다.

따라서 자유와 통제는 서로 대립하지 않습니다. 자유로운 영역은 잘 통제된 코어 위에서만 안전하게 살아남는다는 것이 Rok의 입장입니다. 두 슬로건은 [`refs-lang/000-meta/`](refs-lang/000-meta/)에서 자세히 다룹니다.

## 레이아웃

저장소는 정해진 구조를 따릅니다. 최상위 디렉토리는 언어판별로 나뉘고, 각 판에는 참조(reference) 두 가지가 들어 있습니다.

- [`refs-lang/`](refs-lang/) - 언어 참조. 어휘, 타입, 표현식, 런타임 모델, ABI, 도구를 다룹니다.
- [`refs-standard-libraries/`](refs-standard-libraries/) - 표준 라이브러리 참조.

모든 판이 이 구조를 따릅니다. `../en/refs-lang/`, `ko/refs-lang/`처럼 파일 구성과 절 번호가 같습니다.

## 판

사양은 두 언어 판으로 나옵니다.

- **English** ([`en/`](../en/README.md)) - 기본 판.
- **한국어** (`ko/`) - 이 판.

두 판은 파일 구성과 절 번호(`3.11.4`)가 같으므로, 같은 절 번호는 어느 판에서나 같은 규칙을 가리킵니다. 두 판이 어긋나면 영어 본문이 기준이고, 그 차이는 양쪽에서 고쳐야 할 버그입니다.

## 읽는 순서

사양은 0장부터 9장까지 차례로 읽도록 쓰였습니다. 새 개념은 항상 그때까지 소개한 개념만으로 설명하며, 이 불변식은 모든 장에서 지켜집니다. 상호 참조도 같은 규칙을 따라 **앞쪽으로만**(backward-only) 가리킵니다. 어느 장을 펼치든 개념을 찾으러 그보다 앞으로 거슬러 올라갈 일은 없습니다. 처음 읽는다면 [`refs-lang/README.md`](refs-lang/README.md)부터 시작하세요.

## 표기 규약

본문, 표, 인용문은 모두 한국어로 씁니다. 코드 펜스 안의 ASCII 다이어그램과 box-drawing 다이어그램만 예외로 영어 라벨을 씁니다. GitHub의 고정폭 글꼴은 한글을 ASCII 폭의 정확히 두 배가 아니라 1.5-1.7배로 그리기 때문에, 한글 라벨이 섞이면 다이어그램 정렬이 어긋납니다. 그래서 다이어그램은 영어로 그리고, 바로 아래 한국어 인용문에서 라벨을 풀어 설명합니다. 코드 예제는 번역한 주석, 부수적인 문자열, 사용자가 채울 부분을 나타내는 문법 자리표시자(메타변수)를 빼면 두 판이 같습니다.

## 용어

한국어판 전체에서 아래 용어를 일관되게 씁니다. 번역자를 위해 영어판의 대응어를 함께 적었습니다. 핵심 용어의 정의는 [용어집](refs-lang/000-meta/007-glossary.md)에 있습니다.

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

이미 영어인 이름은 두 판 모두 그대로 씁니다. ROKON, `rok.rokon`, `rok.lock`, `rokc`, `rok`, `rokls`, `rokr`, ROKHIR, HIR, MIR, MLIR, AOT, JIT, GC, Immix, LOS, TLAB, shadow stack, `scan_table`, niche optimization, sticky mark-bit, work-stealing, ABI, FFI, toolchain, sysroot.

## 라이선스

이 사양은 [Creative Commons Attribution 4.0 International (CC BY 4.0)](../LICENSE)으로 배포됩니다.

Copyright © 2026 [Han Damin](https://github.com/miniex) &lt;miniex@daminstudio.net&gt;

출처만 표기하면 텍스트를 자유롭게 공유, 각색, 번역할 수 있습니다. 라이선스 전문은 [LICENSE](../LICENSE)에, 읽기 쉬운 요약은 <https://creativecommons.org/licenses/by/4.0/deed.ko>에 있습니다.

## 상표 (trademark)

"Rok"은 이 프로젝트의 이름이자 정본 사양을 가리키는 식별자입니다. CC BY 4.0은 **텍스트**만 다룹니다. 출처를 표기하면 텍스트 자체는 자유롭게 복사, 번역, 각색할 수 있습니다. 다만 포크한 사양, 대체 사양, 경쟁 구현에는 "Rok"이라는 이름을 쓰지 말아 주세요. 정본 프로젝트와 혼동되지 않도록 다른 이름을 쓰세요.

등록 상표가 아니라 **요청**입니다. 프로젝트가 정식 보호가 필요할 만큼 커지면 등록을 검토합니다.
