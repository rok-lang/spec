# 1.5. volatile 메모리 접근

`rok::bare::volatile` 모듈은 컴파일러가 최적화로 없애지 않는 메모리 read / write를
정의합니다. hardware register, MMIO, signal handler, DMA buffer처럼 모든 접근이 쓴 그대로
실행되어야 하는 곳에서는 이 모듈이 필수입니다.

## 1.5.1. 동기

컴파일러는 평범한 read / write를 마음대로 제거하거나 재배열하거나 합칠 수 있습니다.
다음 코드를 보세요.

```rok
let p: Ptr<u32> = mmio_register;
unsafe {
    p.write(1);                // 컴파일러: dead store 가능 (다음 write 가 덮어씀)
    p.write(2);                // 실제 hardware 는 1 → 2 두 번 받아야 함
}
```

컴파일러는 첫 write를 dead store로 보고 지울 수 있지만, 어떤 hardware는 1과 2를 차례로
받아야 제대로 동작합니다. `volatile`은 이런 최적화를 막고 접근을 쓴 그대로 실행하게 합니다.

```rok
unsafe {
    rok::bare::volatile::write(p, 1);    // 반드시 hardware 에 1 write
    rok::bare::volatile::write(p, 2);    // 반드시 2 write
}
```

## 1.5.2. API

API는 read와 write 두 함수뿐입니다.

```rok
// rok::bare::volatile
unsafe fn read<T>(src: ConstPtr<T>) -> T;
unsafe fn write<T>(dst: Ptr<T>, value: T);
```

같은 동작을 `Ptr<T>::read_volatile` / `write_volatile` 메서드로도 쓸 수 있으며,
그쪽이 더 자연스러운 경우가 많습니다([1.1.2](101-ptr.md#112-ptrt---strict-provenance-api)).

## 1.5.3. 사용처

`volatile`이 필요한 곳은 대체로 다음과 같습니다.

| 자리                   | 이유                                  |
|------------------------|---------------------------------------|
| MMIO register          | hardware가 read만으로도 side effect |
| Memory-mapped 디바이스 | DMA 등                                |
| signal handler 변수    | signal 안에서 변경, 외부에서 봐야 함 |
| spinlock retry loop    | 다른 thread가 변경한 값 반복 read    |

## 1.5.4. 주의 - atomic 과 다름

흔히 오해하는 점이 있습니다. `volatile`은 컴파일러 최적화를 막을 뿐,
atomic을 보장하지 않습니다.

- multi-thread에서 race condition은 여전히 발생합니다.
- memory ordering도 강제하지 않습니다.

따라서 thread 사이에서 공유하는 변수에는
[`Atomic<T>`](../200-core/216-sync.md#2162-atomict---lock-free-원자)를 써야 하고,
`volatile`은 **hardware MMIO** 전용 도구로 남겨 두어야 합니다.

## 1.5.5. lowering

`volatile::read`와 `volatile::write`는 LLVM의 `load volatile` / `store volatile`로 lower됩니다. 둘은 `ConstPtr` / `Ptr`의 메서드 `read_volatile` / `write_volatile`을 부르며, 그 intrinsic은 `rok.ptr.read_volatile`과 `rok.ptr.write_volatile`입니다.

atomic은 [2.16 sync](../200-core/216-sync.md)에서 자세히 다루며, 이 장은
atomic이 아닌 경우만 규정합니다.
