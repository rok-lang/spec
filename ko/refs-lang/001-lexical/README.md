# 1. 어휘 구조

nok 소스 코드를 토큰으로 분해하는 규칙. 파일, 줄, 식별자, 키워드, 리터럴, 연산자 토큰을
다룬다.

상위 레이어(타입, 표현식, 문장)는 이미 토큰화된 입력을 받아 처리한다. 토큰화 단계의 규칙은
이 장에 모두 모인다.

## 목차

- [1.1. 소스 파일](101-source-files.md). 패키지, 디렉토리, `.nok` 파일.
- [1.2. 어휘 구조](102-lexical-structure.md). 줄, 공백, 주석.
- [1.3. 문장 종결자](103-statement-terminator.md). `;` 와 자동 삽입 규칙.
- [1.4. 식별자](104-identifiers.md). 이름 형식.
- [1.5. 키워드](105-keywords.md). 예약어 목록.
- [1.6. 숫자 리터럴](106-numeric-literals.md).
- [1.7. 문자열 리터럴](107-string-literals.md).
- [1.8. 문자 리터럴](108-character-literals.md).
- [1.9. 진리값과 센티넬 리터럴](109-other-literals.md). `true`, `false`, `none`.
- [1.10. 구두점과 연산자 토큰](110-punctuation-and-operator-tokens.md).
