# 1. 어휘 구조

이 장은 rok 소스 코드를 토큰으로 분해하는 규칙을 다룹니다. 구체적으로는 파일과 줄을 시작으로 식별자, 키워드, 리터럴, 그리고 연산자 토큰까지 토큰화 단계에서 마주치는 모든 요소를 살펴봅니다.

타입과 표현식, 문장과 같은 상위 레이어는 이미 토큰화가 끝난 입력을 받아서 처리하기 때문에, 토큰화 단계에서 적용되는 규칙은 모두 이 장 한 곳에 모여 있습니다.

## 목차

- [1.1. 소스 파일](101-source-files.md). 패키지, 디렉토리, `.rok` 파일.
- [1.2. 어휘 구조](102-lexical-structure.md). 줄, 공백, 주석.
- [1.3. 문장 종결자](103-statement-terminator.md). `;` 와 자동 삽입 규칙.
- [1.4. 식별자](104-identifiers.md). 이름 형식.
- [1.5. 키워드](105-keywords.md). 예약어 목록.
- [1.6. 숫자 리터럴](106-numeric-literals.md).
- [1.7. 문자열 리터럴](107-string-literals.md).
- [1.8. 문자 리터럴](108-character-literals.md).
- [1.9. 진리값과 센티넬 리터럴](109-other-literals.md). `true`, `false`, `none`.
- [1.10. 구두점과 연산자 토큰](110-punctuation-and-operator-tokens.md).
