# 7.5. 동시 mark

full 수집은 mark 단계를 mutator와 동시에 실행할 수 있습니다. minor 수집, 명시적 `rok_gc_collect` 호출, evacuation은 언제나 stop-the-world입니다([7.3](703-gc-algorithm.md)).

## 7.5.1. 언제

| `ROK_GC_CONCURRENT` | full 수집                                    |
|---------------------|----------------------------------------------|
| `auto` (기본)       | old 객체가 32 MiB 이상이면 동시 mark          |
| `always`            | 언제나 동시 mark                              |
| `off`               | 언제나 STW                                    |

설정하지 않았거나 비어 있는 변수는 기본값을 씁니다. 표에 없는 값이면 변수 이름, 그 값, 허용되는 값을 알리는 경고 한 줄을 stderr에 출력하고 기본값을 씁니다.

## 7.5.2. 단계

1. **시작 pause (STW).** minor 수집을 한 번 하고, safepoint에서 루트의 snapshot을 뜹니다. 전역 `MARKING` 플래그를 켭니다.
2. **동시 mark.** marker 스레드 하나가 snapshot에서 출발해 mark합니다. mutator는 계속 실행됩니다.
3. **마지막 pause (STW).** SATB buffer를 비우고, 루트를 다시 훑은 뒤 sweep합니다. `MARKING`을 끕니다.

## 7.5.3. SATB barrier

객체 단위의 snapshot-at-the-beginning(Yuasa) 방식입니다. 세대 수집의 write barrier([7.4.2](704-barriers-and-roots.md#742-write-barrier-incremental-update))를 그대로 씁니다.

- `UNLOGGED` 객체에 처음 쓸 때 타는 slow path는, `MARKING`이 켜져 있으면 그 객체의 현재 참조를 모두 SATB buffer에 기록합니다(TLS `rok_tls_satb_cursor` / `rok_tls_satb_limit`, [8.3](../008-abi/803-tls-slots.md)).
- 기록하는 동안 헤더의 `SATB` 비트를 켭니다. 같은 객체에 경쟁하는 store는 그 비트가 꺼질 때까지 기다립니다.
- 그래서 컴파일러가 내는 barrier 코드는 동시 mark를 쓰든 안 쓰든 같습니다.

## 7.5.4. 할당과 수집

- mark 중의 할당은 line 단위로 black입니다(그 line의 객체는 살아 있는 것으로 봄). mark 중에 할당된 객체는 mark가 끝난 뒤 young 객체가 됩니다.
- mark 중에는 minor 수집을 하지 않습니다. 할당 때문에 수집이 필요하면 진행 중인 mark를 끝냅니다.
- evacuation은 STW full 수집에서만 합니다. 동시 mark 뒤 heap이 단편화되어 있으면, 다음 full 수집은 evacuation을 포함한 STW로 합니다.
