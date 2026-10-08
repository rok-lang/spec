# nok 언어 참조

이 디렉토리는 nok 언어의 _형식 사양 (formal specification)_ 이다. nok 의 _동작_ 을 정의한다.

## 챕터

- [0. 메타 — 언어 / 빌드 모델](000-meta/README.md)
- [1. 어휘 구조](001-lexical/README.md)
- [2. 타입 시스템](002-types/README.md)
- [3. 표현식](003-expressions/README.md)
- [4. 함수](004-functions/README.md)
- [5. 변수 / 모듈 / 가시성 / 패키지](005-modules/README.md)
- [6. 메타 시스템](006-meta-system/README.md)
- [7. 런타임 모델](007-runtime/README.md)
- [8. 런타임 ABI / MLIR lowering](008-abi/README.md)
- [9. 실행 / 툴](009-tools/README.md)

## 읽는 순서

본 사양은 _0 → 9 순서대로 읽으면 새 개념이 반드시 직전까지 본 개념으로만 설명_ 되도록 작성
되었다. 임의 챕터를 펼쳐도 _상위 챕터의 개념만 사용_ 한다는 invariant 가 유지된다.

cross-reference 는 _backward-only_ — 현재 문서가 다음 문서의 정의에 의존하면 안 된다.
"후속 사양에서 정의한다" 는 표시는 _그 자리에서 짧게_ 만 등장하고 _해당 문서로 링크하지
않는다_.

## 표기 규약

- _키워드 / 식별자_ — `` `let` `` (코드 폰트)
- _타입 이름_ — `` `i32` `` (코드 폰트)
- _개념 강조_ — `*emphasis*` (이탤릭)
- _경고 / 미결정_ — `**strong**` (볼드)
- _어드모니션_ — GitHub 스타일 인용 블록 `> [!NOTE]` / `> [!WARNING]`
- _문법 정의_ — BNF 스타일 `name ::= ...`
