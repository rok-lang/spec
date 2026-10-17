# 7.5. 동시 mark

full 수집은 mark 를 mutator 와 동시에 할 수 있습니다. minor 수집, 명시적 `rok_gc_collect`, evacuation 은 언제나 stop-the-world 입니다 ([7.3](703-gc-algorithm.md)).

## 7.5.1. 언제

| `ROK_GC_CONCURRENT` | full 수집                                    |
|---------------------|----------------------------------------------|
| `auto` (기본)       | old 객체가 32 MiB 이상이면 동시 mark          |
| `always`            | 언제나 동시 mark                              |
| `off`               | 언제나 STW                                    |

## 7.5.2. 단계

1. **시작 pause (STW).** minor 수집을 한 번 하고, safepoint 에서 루트의 snapshot 을 뜹니다. 전역 `MARKING` 플래그를 켭니다.
2. **동시 mark.** marker thread 하나가 snapshot 에서 출발해 mark 합니다. mutator 는 계속 실행됩니다.
3. **마지막 pause (STW).** SATB buffer 를 비우고, 루트를 다시 훑고, sweep 합니다. `MARKING` 을 끕니다.

## 7.5.3. SATB barrier

snapshot-at-the-beginning (Yuasa) 이며 객체 단위입니다. 세대 수집의 write barrier ([7.4.2](704-barriers-and-roots.md#742-write-barrier-incremental-update)) 를 그대로 씁니다.

- `UNLOGGED` 객체에 처음 쓰는 slow path 는, `MARKING` 이 켜져 있으면 그 객체의 현재 참조를 모두 SATB buffer 에 기록합니다 (TLS `rok_tls_satb_cursor` / `rok_tls_satb_limit`, [8.3](../008-abi/803-tls-slots.md)).
- 기록하는 동안 헤더의 `SATB` 비트를 켭니다. 같은 객체에 경쟁하는 store 는 그 비트가 꺼질 때까지 기다립니다.
- 그래서 compiler 가 내는 barrier 코드는 동시 mark 가 있어도 같습니다.

## 7.5.4. 할당과 수집

- mark 중의 할당은 line 단위로 black 입니다 (그 line 의 객체는 산 것으로 봄). mark 중에 할당된 객체는 mark 가 끝난 뒤 young 입니다.
- mark 중에는 minor 수집을 하지 않습니다. 할당이 수집을 요구하면 진행 중인 mark 를 끝냅니다.
- evacuation 은 STW full 수집에서만 합니다. 동시 mark 뒤 heap 이 단편화되어 있으면 다음 full 수집을 evacuation 이 있는 STW 로 합니다.
