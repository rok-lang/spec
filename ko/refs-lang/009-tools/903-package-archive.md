# 9.3. 패키지 아카이브(.a) 포맷

라이브러리 패키지 `foo`의 빌드 산출물은 두 파일입니다([9.2.15](902-aot-pipeline.md#9215-산출물)).

| 파일        | 내용                                                                          |
|-------------|-------------------------------------------------------------------------------|
| `libfoo.a`  | 패키지의 오브젝트를 codegen unit마다 하나씩 담은 표준 `ar` 아카이브([9.2.16](902-aot-pipeline.md#9216-증분-빌드)) |
| `foo.rokhir`  | 다운스트림이 단형화와 이름 해소에 쓰는 인터페이스([9.4](904-rokhir-bytecode.md)) |

두 파일은 같은 디렉터리에 놓입니다. `--extern foo=libfoo.a`와 `--extern foo=foo.rokhir`은 어느 쪽을 주든 옆에 있는 짝을 찾습니다([9.2.3](902-aot-pipeline.md#923-의존성-해소)). 버전과 producer 같은 메타데이터는 ROKHIR 헤더에 있습니다([9.4](904-rokhir-bytecode.md)).

## 9.3.1. 표준 ar 아카이브

`libfoo.a`는 표준 Unix `ar` 아카이브입니다. `ar`, `ranlib`, `nm`, `lld`, `clang`이 변환 없이 다룹니다.

```
ar t libfoo.a
part0.o
part1.o
...
```

## 9.3.2. 오브젝트

각 오브젝트는 [AOT 빌드 파이프라인](902-aot-pipeline.md)이 codegen unit 하나로 만든 표준 ELF / Mach-O / COFF 오브젝트입니다. 오브젝트를 모두 합치면 패키지의 [concrete fn](../000-meta/007-glossary.md#concrete-fn--generic-fn) 가운데 패키지 루트에서 닿는 것만 들어 있습니다([9.2.8](902-aot-pipeline.md#928-specialize-단형화)).

| 섹션      | 내용                                                        |
|-----------|-------------------------------------------------------------|
| `.text`   | concrete `fn`의 코드                                       |
| `.rodata` | `const`, 리터럴, [ScanTable](../008-abi/802-scan-table.md)  |
| `.data`   | `static`의 초기값                                          |
| `.bss`    | 0으로 초기화되는 `static`                                  |

ScanTable은 일반 상수 데이터입니다. 컴파일러가 생성한 등록 코드가 프로세스를 시작할 때 `rok_gc_register_type`으로 ScanTable을 런타임에 등록합니다([8.2.13](../008-abi/802-scan-table.md#8213-등록-시점)). Rok 전용 섹션은 없습니다.

## 9.3.3. 정적 .a 원칙

모든 라이브러리 산출물은 정적 `.a`입니다. 동적 라이브러리는 Rok의 표준 산출물이 아닙니다. [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface)로 동적 라이브러리를 호출할 수는 있습니다.

## 9.3.4. 버전 호환 검사

컴파일러는 의존 패키지의 ROKHIR을 읽을 때 그 `version`과 producer(그 ROKHIR을 만든 `rokc`)를 지금의 `rokc`와 비교합니다. 다르면 에러이고, 에러 메시지는 무엇을 다시 빌드할지 밝힙니다([9.4.9](904-rokhir-bytecode.md#949-읽기와-거부)).

## 9.3.5. 심볼 노출

아카이브가 노출하는 심볼은 그 안 오브젝트의 전역 심볼입니다. 이름은 [심볼 맹글링](905-symbol-mangling.md)을 따릅니다.

```
nm libfoo.a
0000000000000000 T _Y3f8a1c2b_foo_geom_distance_a7c91d8e44b2f190
0000000000000040 T _Y3f8a1c2b_foo_geom_Point_origin_b1234567...
```

## 9.3.6. 표준 라이브러리

표준 라이브러리(`rok::bare` / `core` / `std`)는 별도 아카이브로 배포하지 않습니다. 소스에서 컴파일하고(`--bootstrap`), toolchain과 옵션과 소스가 그대로인 동안은 `rokc` 캐시에 있는 것을 다시 씁니다([9.2.19](902-aot-pipeline.md#9219-설치-레이아웃)).

## 9.3.7. 링크

```
clang main.o libfoo.a librokr.a -o my_binary
```

링커는 아카이브의 오브젝트 가운데 필요한 심볼만 링크합니다. `#[export_name]`으로 노출한 함수는 C에서 직접 부를 수 있습니다.

```c
extern uint32_t rok_compute(int32_t a, int32_t b);   // #[export_name("rok_compute")]
int main(void) { return (int)rok_compute(1, 2); }
```

```
clang main.c libfoo.a librokr.a -o my_binary
```

## 9.3.8. 안정성

아카이브 형식은 [ABI 안정성 정책](../008-abi/README.md)을, ROKHIR 형식은 [9.4](904-rokhir-bytecode.md)를 따릅니다.
