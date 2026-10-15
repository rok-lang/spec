# 7.5. 동시 수집 (없음)

rok 의 GC 는 stop-the-world 수집기입니다 ([7.3](703-gc-algorithm.md)). 동시 mark, 동시 evacuation, SATB barrier, Brooks 포인터, RC + trace 결합은 설계에 포함되지 않습니다.

- 수집 중 모든 mutator thread 는 safepoint 에 멈춥니다 ([7.4.3](704-barriers-and-roots.md#743-safepoint-polling), [8.4](../008-abi/804-safepoint.md)).
- write barrier 는 세대 수집용 object-remembering barrier 와 card marking 하나뿐입니다 ([7.4.2](704-barriers-and-roots.md#742-write-barrier-incremental-update)).
- pause 시간은 minor 수집으로 줄입니다. 대부분의 수집은 old 객체를 훑지 않습니다 ([7.3.2](703-gc-algorithm.md#732-sticky-mark-bit-세대-수집)).
