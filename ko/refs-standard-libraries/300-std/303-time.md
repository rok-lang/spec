# 3.3. `time::` - 시간

`rok::std::time` 모듈은 세 종류의 시간 타입과 `Duration` 을 함께 제공합니다. 단조 시계, 벽시계, 그리고 부팅 후 경과 시간을 명확히 분리해 두어, 측정 목적에 맞는 시계를 골라 쓸 수 있도록 했습니다.

## 3.3.1. 4 타입 모델

전체 그림은 아래 표와 같습니다.

| 타입         | 의미                        | 사용처                   |
|--------------|-----------------------------|--------------------------|
| `Instant`    | 단조 시계 (monotonic)       | duration 측정 / timeout  |
| `SystemTime` | 벽시계 (UTC, NTP 가능)      | 파일 mtime / 사용자 표시 |
| `BootTime`   | 부팅 후 시간 (suspend 포함) | 모바일 / laptop 앱       |
| `Duration`   | 시간 차이 (i64 nanosecond)  | 둘 사이 차 / sleep 인자  |

`Duration` 의 내부 표현은 `i64` ns 입니다. Rust 가 채택한 `u128` 은 다소 과잉이라고 판단했으며, `i64` 로도 약 ±292 년을 표현할 수 있어 실용 범위에서는 부족함이 없습니다.

## 3.3.2. `Instant` - 단조 시계

```rok
struct Instant { /* opaque - CLOCK_MONOTONIC / QueryPerformanceCounter */ }

impl Instant {
    fn now() -> Instant;

    fn elapsed(self) -> Duration;
    fn duration_since(self, earlier: Instant) -> Duration;       // self - earlier
    fn checked_duration_since(self, earlier: Instant) -> Duration | none;
    fn saturating_duration_since(self, earlier: Instant) -> Duration;

    // 산술 - 실패 시 none 반환 (Rust 의 saturating-on-overflow silent 회피)
    fn checked_add(self, dur: Duration) -> Instant | none;
    fn checked_sub(self, dur: Duration) -> Instant | none;
}

// 일반 산술 - overflow 시 overflow 검사가 켜져 있으면 panic, 꺼져 있으면 saturate (3.3.5)
impl Add<Duration> for Instant { type Output = Instant; ... }
impl Sub<Instant> for Instant { type Output = Duration; ... }
impl Sub<Duration> for Instant { type Output = Instant; ... }
```

`Instant` 의 핵심 특성은 다음과 같이 정리됩니다. 우선 절대로 뒤로 흐르지 않으므로 NTP 보정의 영향을 받지 않습니다. 또한 자체적으로 절대 시간 의미를 가지지 않기 때문에 (process 시작 시점이 기준이므로) **serialize 가 불가능** 합니다. 같은 이유로 process 사이의 비교도 무의미하며, 동일 process 내부에서만 의미가 성립합니다.

### 사용 예

가장 흔한 패턴은 측정 시작점을 잡아 두고 `elapsed` 로 경과를 얻는 것입니다.

```rok
let start = Instant::now();
heavy_computation();
let elapsed = start.elapsed();          // Duration
log(f"computation took {elapsed:?}");
```

## 3.3.3. `SystemTime` - 벽시계

```rok
struct SystemTime { /* opaque - CLOCK_REALTIME / GetSystemTimeAsFileTime */ }

impl SystemTime {
    fn now() -> SystemTime;

    fn duration_since(self, earlier: SystemTime) throws (SystemTimeError) -> Duration;
                                            // earlier > self 면 Err

    fn elapsed(self) throws (SystemTimeError) -> Duration;

    fn checked_add(self, dur: Duration) -> SystemTime | none;
    fn checked_sub(self, dur: Duration) -> SystemTime | none;
}

const UNIX_EPOCH: SystemTime;     // 1970-01-01 00:00:00 UTC

// SystemTime ↔ unix timestamp
impl SystemTime {
    fn from_unix(secs: i64, nanos: u32) -> SystemTime;
    fn unix_seconds(self) throws (SystemTimeError) -> i64;
    fn unix_nanos(self) throws (SystemTimeError) -> i64;   // rok 에 i128 이 없음 - i64 ns 는 약 ±292 년
}

impl Add<Duration> for SystemTime { type Output = SystemTime; ... }
impl Sub<Duration> for SystemTime { type Output = SystemTime; ... }

struct SystemTimeError {
    duration: Duration;        // earlier 가 앞서 있던 만큼
}
```

`SystemTime` 은 `UNIX_EPOCH` 기준 `i64` 나노초이므로 표현 범위는 모든 타깃에서 약 ±292 년입니다. 32-bit 타깃도 64-bit 시간 인터페이스 (`time64`) 로 시계를 읽으므로 2038 한계가 없습니다.

반면 `SystemTime` 의 성격은 사뭇 다릅니다. NTP 보정이나 사용자 변경에 의해 시간이 뒤로 갈 수도 있고, UNIX_EPOCH 를 기준으로 한 절대 시간이므로 **serialize 가 가능** 합니다. 그 덕분에 process 사이에서도 의미 있게 비교할 수 있습니다.

### 사용 예

unix timestamp 변환과 파일 mtime 조회는 다음과 같이 자연스럽게 표현됩니다.

```rok
let now = SystemTime::now();
let unix = try now.unix_seconds();
log(f"unix timestamp: {unix}");

// 파일 mtime
let meta = try fs::metadata("data.txt");
let mtime: SystemTime = try meta.modified();
let age = try SystemTime::now().duration_since(mtime);
log(f"file age: {age:?}");
```

## 3.3.4. `BootTime` - 부팅 후 시간 (suspend 포함)

```rok
struct BootTime { /* CLOCK_BOOTTIME (Linux) / CLOCK_MONOTONIC = mach_continuous_time (macOS) */ }

impl BootTime {
    fn now() -> BootTime;
    fn elapsed(self) -> Duration;
    fn duration_since(self, earlier: BootTime) -> Duration;
}
```

`BootTime` 과 `Instant` 의 차이는 suspend 동작에서 갈립니다. `Instant` 는 Linux `CLOCK_MONOTONIC` 처럼 suspend 동안에는 멈춰 있는 반면, `BootTime` 은 `CLOCK_BOOTTIME` 을 따라 suspend 중에도 계속 진행합니다.

이러한 차이는 모바일이나 laptop 앱에서 "실제 경과 시간 (suspend 포함)" 을 측정해야 할 때 결정적입니다. Rust 는 2026 시점에도 stable API 가 없는 영역인데, Rok 은 이를 일찍 표면에 노출시켜 둡니다.

## 3.3.5. `Duration`

```rok
struct Duration { /* opaque: i64 nanosecond - 약 ±292년 */ }

impl Duration {
    const ZERO: Duration;             // 0 ns
    const MAX: Duration;              // I64_MAX ns

    // 생성자
    fn ns(n: i64) -> Duration;
    fn us(n: i64) -> Duration;        // microseconds
    fn ms(n: i64) -> Duration;        // milliseconds
    fn sec(n: i64) -> Duration;
    fn min(n: i64) -> Duration;
    fn hours(n: i64) -> Duration;
    fn days(n: i64) -> Duration;

    fn from_secs_f64(s: f64) -> Duration;        // floating point seconds

    // 추출
    fn as_ns(self) -> i64;
    fn as_us(self) -> i64;
    fn as_ms(self) -> i64;
    fn as_secs(self) -> i64;
    fn as_secs_f64(self) -> f64;

    fn subsec_ns(self) -> u32;          // 하위 nanosecond 부분
    fn subsec_us(self) -> u32;
    fn subsec_ms(self) -> u32;
}

// 산술
impl Add for Duration { ... }
impl Sub for Duration { ... }
impl Mul<i64> for Duration { ... }     // Duration × i64
impl Div<i64> for Duration { ... }
impl Mul<Duration> for f64 { ... }      // f64 × Duration

impl Display for Duration { ... }       // "1.234s" / "5ms" / "1.5µs" / "100ns"
impl Debug for Duration { ... }         // Display 와 같음
```

`Display` 는 크기에 맞는 단위 하나로 씁니다. 절댓값이 1 초 이상이면 `s`, 1 ms 이상이면 `ms`, 1 µs 이상이면 `µs`, 그 밖에는 `ns` 입니다. 그 단위로 나눈 정수부 뒤에 나머지를 소수로 붙이되, 끝의 0 은 지우고 나머지가 0 이면 소수점도 쓰지 않습니다. 음수는 앞에 `-` 를 붙입니다.

```rok
Duration::ms(1234);      // "1.234s"
Duration::ms(5);         // "5ms"
Duration::ns(1500);      // "1.5µs"
Duration::ns(-20);       // "-20ns"
```

시간 타입은 모두 `PartialEq` / `Eq` / `PartialOrd` / `Ord` / `Hash` 를 구현합니다. `Instant`, `SystemTime`, `BootTime` 은 `Debug` 도 구현합니다.

### Duration 산술 정책

`Duration` / `SystemTime` / `Instant` / `BootTime` 의 산술 (`duration_since`, `from_unix` 포함) 은 overflow 검사가 켜져 있으면 overflow 에 panic 하고, 꺼져 있으면 saturate 합니다 (`-C overflow-checks`, [refs-lang 9.1.2](../../refs-lang/009-tools/901-rokc-cli.md#rokc-build-aot-빌드)). 0 으로 나누기는 언제나 panic 입니다. `checked_*` / `saturating_*` 변형으로 명시적으로 고를 수 있습니다.

```rok
impl Duration {
    fn checked_add(self, rhs: Duration) -> Duration | none;
    fn checked_sub(self, rhs: Duration) -> Duration | none;
    fn checked_mul(self, factor: i64) -> Duration | none;
    fn checked_div(self, divisor: i64) -> Duration | none;

    fn saturating_add(self, rhs: Duration) -> Duration;
    fn saturating_sub(self, rhs: Duration) -> Duration;
    fn saturating_mul(self, factor: i64) -> Duration;
}
```

## 3.3.6. 정책 - TAI / leap second 미지원

`SystemTime` 은 어디까지나 **UTC** 이며, leap second 는 처리하지 않습니다 (Linux kernel 자체가 이 부분에서 모호하다는 점도 고려했습니다). TAI (atomic time) 나 leap second 의 정확한 처리가 필요한 경우는 별도 패키지 (`rok/chrono` 또는 `rok/tai`) 의 영역으로 분리합니다.

| 자리                       | 권장                      |
|----------------------------|---------------------------|
| 일반 앱 (timestamp / 측정) | rok::std::time            |
| 시뮬레이션 / 정밀 천체     | 별도 `chrono`-like 패키지 |
| 금융 / 정확한 leap second  | 별도 패키지               |

## 3.3.7. impl / target

플랫폼별 구현 매핑은 아래 표를 참고합니다.

| 항목         | linux           | macos            | windows                 | wasi         |
|--------------|-----------------|------------------|-------------------------|--------------|
| `Instant`    | CLOCK_MONOTONIC | CLOCK_UPTIME_RAW | QueryPerformanceCounter | wasi clock   |
| `SystemTime` | CLOCK_REALTIME  | CLOCK_REALTIME   | GetSystemTimePreciseAsFileTime | wasi  |
| `BootTime`   | CLOCK_BOOTTIME  | CLOCK_MONOTONIC (mach_continuous_time, sleep 포함) | QueryPerformanceCounter (~ Instant) | (no suspend) |

## 3.3.8. 정책 요약

본 장의 핵심 결정은 세 가지로 정리됩니다. 먼저 `Instant` / `SystemTime` / `Duration` 을 분리하여 단조 시간과 벽시계 시간을 의미상 구분합니다. 다음으로 Rust 의 std 가 다루지 않는 `BootTime` 을 별도로 노출하여, suspend 를 포함한 부팅 이후 경과 시간을 직접 확인할 수 있게 합니다. 마지막으로 TAI 와 leap second 는 본 모듈 바깥의 별도 패키지로 분리합니다.
