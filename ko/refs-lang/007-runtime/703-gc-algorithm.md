# 7.3. GC 알고리듬

rok 의 GC 는 [Immix](../000-meta/007-glossary.md#immix) mark-region[^immix] 위에 sticky mark-bit 세대 수집[^whippet] 을 얹은 stop-the-world 수집기입니다. 수집 중 mutator 는 safepoint ([8.4](../008-abi/804-safepoint.md)) 에서 멈춥니다. 루트와 write barrier 는 [7.4](704-barriers-and-roots.md) 에서 정의합니다.

## 7.3.1. Immix mark-region

### block 과 line

```
block (32 KiB) = line (128 B) × 256
```

[block](../000-meta/007-glossary.md#block-블록) 은 OS 에서 받습니다 (mmap). 객체는 line 경계를 넘을 수 있습니다. mark 는 객체와 함께 그 객체가 걸친 line 을 표시하며, sweep 은 표시 없는 line 을 회수합니다. 산 객체 사이의 작은 틈도 line 단위로 다시 씁니다.

### sweep: block 분류

| 분류       | 상태                       | 처리                         |
|------------|----------------------------|------------------------------|
| clean      | 모든 line 이 비어 있음     | free pool 로                 |
| recyclable | 일부 line 만 살아 있음     | 빈 line 을 할당에 다시 씀    |
| full       | 거의 모든 line 이 살아 있음 | 그대로 둠                    |

sweep 은 죽은 객체를 0 으로 지우지 않습니다. 할당기가 TLAB 을 채울 때 line 을 0 으로 지웁니다.

### opportunistic evacuation

full 수집은 단편화된 block 의 산 객체를 다른 block 으로 옮길 수 있습니다.

- 후보는 직전 sweep 이 남긴 line 상태로 고릅니다 (절반 이하로 찬 block). 단편화 임계를 넘었을 때만 옮깁니다.
- 옮긴 객체의 옛 자리에는 forwarding pointer 를 남기고, 같은 수집 안에서 모든 정밀 참조를 새 위치로 고칩니다.
- 옮기지 않는 객체: 루트가 직접 가리키는 객체 ([7.4.1](704-barriers-and-roots.md#루트가-가리키는-객체는-옮기지-않음)), 모호한 word (`AMBIGUOUS_WORD`, enum / union payload) 가 가리키는 객체, `PINNED` / `NOMOVE` 객체, LOS 객체.
- 다음 cycle 의 할당 예산을 넘는 빈 block 은 OS 에 돌려줍니다.
- `ROK_GC_EVAC=always|off` 로 evacuation 을 강제하거나 끕니다 (기본은 자동, [9.9](../009-tools/909-debug-tools.md)).

## 7.3.2. sticky mark-bit 세대 수집

nursery 공간을 따로 두지 않고, 헤더의 `STICKY` 비트 ([8.1.3](../008-abi/801-object-header-abi.md#813-gc_flags-offset-4-u32)) 로 세대를 나눕니다. 수집에서 살아남은 객체는 `STICKY` 를 켜고 old 가 됩니다. 객체는 제자리에 있으며 promotion 복사가 없습니다.

| 수집          | mark 의 출발점                                   | 범위                                          |
|---------------|--------------------------------------------------|-----------------------------------------------|
| minor (young) | 루트, remembered set, dirty card ([7.4.2](704-barriers-and-roots.md#742-write-barrier-incremental-update)) | old 객체 안으로 들어가지 않음. sweep 은 지난 수집 뒤 나눠 준 line 만 |
| full (old)    | 루트                                             | `STICKY` 를 모두 지우고 전체 mark / sweep     |

- young line 은 객체 조회가 닿은 block 만 파싱합니다.
- full 수집은 old 객체의 바이트가 `max(4 MiB, 직전 full 수집 뒤 live × 2)` 에 닿을 때, 또는 `rok_gc_collect` 를 부를 때 일어납니다.
- `ROK_GC_GEN=0` 은 세대 수집을 끄고 매번 full 수집합니다.

## 7.3.3. mark 의 절차

1. 루트 ([7.4.1](704-barriers-and-roots.md#741-루트-추적-컴파일러가-직접-emit-하는-shadow-stack)) 에서 시작합니다.
2. 도달한 객체마다 헤더의 `MARK` 를 켜고, 걸친 line 을 표시하고, `type_id` 의 [ScanTable](../008-abi/802-scan-table.md) 로 자식을 따라갑니다.
3. 큐가 비면 끝납니다.

ScanTable 은 `rok_gc_register_type` ([8.5.1](../008-abi/805-extern-symbols.md#851-gc-영역)) 으로 등록됩니다.

## 참고 문헌

[^immix]: Blackburn & McKinley, _Immix: A Mark-Region Garbage Collector with Space Efficiency, Fast Collection, and Mutator Performance_, PLDI 2008.

[^whippet]: Wingo, Whippet: a sub-toolkit for embeddable garbage collection. <https://wingolog.org/tags/whippet>
