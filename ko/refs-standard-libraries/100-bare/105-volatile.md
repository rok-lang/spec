# 1.5. volatile 메모리 접근

`rok::bare::volatile` 모듈은 컴파일러가 함부로 최적화하지 않는 메모리 read / write 를
제공합니다. hardware register, MMIO, signal handler, DMA buffer 같이 매 접근이 그대로
실행되어야 하는 자리에서는 이 통로가 필수입니다.

## 1.5.1. 동기

평범한 read / write 는 컴파일러가 자유롭게 제거하거나 재배열하거나 합쳐 버릴 수 있습니다.
다음 코드를 봅니다.

```rok
let p: Ptr<u32> = mmio_register;
unsafe {
    p.write(1);                // 컴파일러: dead store 가능 (다음 write 가 덮어씀)
    p.write(2);                // 실제 hardware 는 1 → 2 두 번 받아야 함
}
```

여기서 컴파일러는 첫 write 를 dead store 로 보고 지워 버릴 수 있지만, 실제 hardware 는
1 과 2 를 차례로 받아야 정상 동작하는 경우가 있습니다. 그래서 `volatile` 은 이러한 최적화를
거부하고 작성된 그대로의 접근을 강제합니다.

```rok
unsafe {
    rok::bare::volatile::write(p, 1);    // 반드시 hardware 에 1 write
    rok::bare::volatile::write(p, 2);    // 반드시 2 write
}
```

## 1.5.2. API

API 는 단출하게 read 와 write 두 함수로 구성됩니다.

```rok
// rok::bare::volatile
unsafe fn read<T>(src: ConstPtr<T>) -> T;
unsafe fn write<T>(dst: Ptr<T>, value: T);
```

같은 동작을 `Ptr<T>::read_volatile` / `write_volatile` 메서드 형태로도 쓸 수 있고,
이쪽이 더 자연스러운 자리도 많습니다 ([1.1.2](101-ptr.md#112-ptrt---strict-provenance-api)).

## 1.5.3. 사용처

`volatile` 이 필요한 자리는 대체로 다음 같은 패턴으로 정리됩니다.

| 자리                   | 이유                                  |
|------------------------|---------------------------------------|
| MMIO register          | hardware 가 read 만으로도 side effect |
| Memory-mapped 디바이스 | DMA 등                                |
| signal handler 변수    | signal 안에서 변경 - 외부에서 봐야 함 |
| spinlock retry loop    | 다른 thread 가 변경한 값 반복 read    |

## 1.5.4. 주의 - atomic 과 다름

흔한 오해 하나를 짚고 갑니다. `volatile` 은 어디까지나 컴파일러 최적화를 차단할 뿐,
atomic 보장을 주지는 않습니다.

- multi-thread 에서 race condition 은 여전히 발생합니다
- memory ordering 도 강제하지 않습니다

따라서 multi-thread 사이에서 공유하는 변수에는
[`Atomic<T>`](../200-core/216-sync.md#2162-atomict---lock-free-원자) 를 써야 하고,
`volatile` 은 **hardware MMIO** 에 한정해서 쓰는 도구로 남겨 두어야 합니다.

## 1.5.5. lowering

`volatile::read` 와 `volatile::write` 는 LLVM 의 `load volatile` / `store volatile` 로 lower 되며니다. 둘은 `ConstPtr` / `Ptr` 의 메서드 `read_volatile` / `write_volatile` 을 부르며, 그 intrinsic 은 `rok.ptr.read_volatile` 과 `rok.ptr.write_volatile` 입니다.

자세한 atomic 관련 내용은 [2.16 sync](../200-core/216-sync.md) 에서 다루고, 이 장은
atomic 이 아닌 자리에 한해서만 권위를 가집니다.
