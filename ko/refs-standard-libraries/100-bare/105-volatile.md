# 1.5. volatile 메모리 접근

`rok::bare::volatile` 모듈은 컴파일러가 최적화로 없애지 않는 메모리 읽기 / 쓰기를
정의합니다. 하드웨어 레지스터, MMIO, 시그널 핸들러, DMA 버퍼처럼 모든 접근을 쓴 그대로
실행해야 하는 곳에서는 이 모듈이 꼭 필요합니다.

## 1.5.1. 동기

컴파일러는 평범한 읽기 / 쓰기를 마음대로 지우거나, 순서를 바꾸거나, 합칠 수 있습니다.
다음 코드를 보세요.

```rok
let p: Ptr<u32> = mmio_register;
unsafe {
    p.write(1);                // 컴파일러: dead store 가능 (다음 write 가 덮어씀)
    p.write(2);                // 실제 hardware 는 1 → 2 두 번 받아야 함
}
```

컴파일러는 첫 쓰기를 죽은 저장(dead store)으로 보고 지울 수 있습니다. 하지만 어떤 하드웨어는
1과 2를 차례로 받아야 제대로 동작합니다. `volatile`은 이런 최적화를 막고 접근을 쓴 그대로 실행하게 합니다.

```rok
unsafe {
    rok::bare::volatile::write(p, 1);    // 반드시 hardware 에 1 write
    rok::bare::volatile::write(p, 2);    // 반드시 2 write
}
```

## 1.5.2. API

API는 읽기와 쓰기 두 함수뿐입니다.

```rok
// rok::bare::volatile
unsafe fn read<T>(src: ConstPtr<T>) -> T;
unsafe fn write<T>(dst: Ptr<T>, value: T);
```

같은 동작을 `Ptr<T>::read_volatile` / `write_volatile` 메서드로도 쓸 수 있습니다.
그쪽이 더 자연스러운 경우가 많습니다([1.1.2](101-ptr.md#112-ptrt-strict-provenance-api)).

## 1.5.3. 사용처

`volatile`이 필요한 곳은 대체로 다음과 같습니다.

| 자리                   | 이유                                  |
|------------------------|---------------------------------------|
| MMIO 레지스터          | 하드웨어는 읽기만으로도 부수 효과가 있음 |
| 메모리 매핑 디바이스   | DMA 등                                |
| 시그널 핸들러 변수     | 시그널 안에서 바꾸고 바깥에서 봐야 함 |
| 스핀락 재시도 루프     | 다른 스레드가 바꾼 값을 반복해 읽음  |

## 1.5.4. 주의: atomic과 다름

흔히 오해하는 점이 있습니다. `volatile`은 컴파일러 최적화를 막을 뿐,
원자성을 보장하지 않습니다.

- 여러 스레드 사이의 경쟁 상태(race condition)는 여전히 일어납니다.
- 메모리 순서(memory ordering)도 강제하지 않습니다.

따라서 스레드 사이에서 공유하는 변수에는
[`Atomic<T>`](../200-core/216-sync.md#2162-atomict-lock-free-원자-타입)를 써야 합니다.
`volatile`은 **하드웨어 MMIO** 전용 도구로 남겨 두기를 권장합니다.

## 1.5.5. lowering

컴파일러는 `volatile::read`와 `volatile::write`를 LLVM의 `load volatile` / `store volatile`로 낮춥니다. intrinsic 이름은 각각 `rok.volatile.read.T`와 `rok.volatile.write.T`입니다.

원자 연산은 [2.16 sync](../200-core/216-sync.md)에서 자세히 다룹니다. 이 장은
원자 연산이 아닌 경우만 정합니다.
