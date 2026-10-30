# 1. 어휘 구조

이 장은 Rok 소스 코드를 토큰으로 나누는 규칙을 정합니다. 파일과 줄에서 시작해 식별자, 키워드, 리터럴, 연산자 토큰까지 토큰화 단계에서 만나는 요소를 모두 다룹니다.

타입, 표현식, 문장 같은 상위 계층은 토큰화가 끝난 입력을 받습니다. 그래서 토큰화 단계의 규칙은 모두 이 장에 모아 둡니다.

## 목차

- [1.1. 소스 파일](101-source-files.md). 패키지, 디렉터리, `.rok` 파일.
- [1.2. 어휘 구조](102-lexical-structure.md). 줄, 공백, 주석.
- [1.3. 문장 종결자](103-statement-terminator.md). `;`와 자동 삽입 규칙.
- [1.4. 식별자](104-identifiers.md). 이름 형식.
- [1.5. 키워드](105-keywords.md). 예약어 목록.
- [1.6. 숫자 리터럴](106-numeric-literals.md).
- [1.7. 문자열 리터럴](107-string-literals.md).
- [1.8. 문자 리터럴](108-character-literals.md).
- [1.9. 진리값과 센티넬 리터럴](109-other-literals.md). `true`, `false`, `none`.
- [1.10. 구두점과 연산자 토큰](110-punctuation-and-operator-tokens.md).
