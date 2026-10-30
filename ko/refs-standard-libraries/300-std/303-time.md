# 3.3. `time::`: 시간

`rok::std::time` 모듈은 시간 타입 세 가지와 `Duration`을 정의합니다. 단조 시계, 벽시계, 부팅 후 경과 시간을 명확히 나눠 두었습니다. 그래서 측정 목적에 맞는 시계를 고를 수 있습니다.

## 3.3.1. 타입 4개 모델

전체 구성은 다음과 같습니다.

| 타입         | 의미                        | 사용처                   |
|--------------|-----------------------------|--------------------------|
| `Instant`    | 단조 시계 (monotonic)       | 경과 시간 측정 / 타임아웃 |
| `SystemTime` | 벽시계 (UTC, NTP 가능)      | 파일 mtime / 사용자 표시 |
| `BootTime`   | 부팅 후 시간 (절전 포함)    | 모바일 / 노트북 앱       |
| `Duration`   | 시간 차이 (i64 나노초)      | 둘 사이 차 / sleep 인자  |

`Duration`의 내부 표현은 `i64` ns입니다. Rust가 채택한 `u128`은 과하다고 판단했습니다. `i64`로도 약 ±292년을 표현할 수 있으므로 실용 범위로는 충분합니다.

## 3.3.2. `Instant`: 단조 시계

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

`Instant`의 핵심 특성은 다음과 같습니다. 절대 뒤로 가지 않으므로 NTP 보정의 영향을 받지 않습니다. 기준이 프로세스 시작 시점이라 그 자체로는 절대 시간의 의미가 없습니다. 그래서 **직렬화할 수 없습니다**. 같은 이유로 프로세스 사이의 비교도 무의미합니다. 같은 프로세스 안에서만 의미가 있습니다.

### 사용 예

가장 흔한 패턴은 시작점을 잡아 두고 `elapsed`로 경과 시간을 얻는 방식입니다.

```rok
let start = Instant::now();
heavy_computation();
let elapsed = start.elapsed();          // Duration
log(f"computation took {elapsed:?}");
```

## 3.3.3. `SystemTime`: 벽시계

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

`SystemTime`은 `UNIX_EPOCH` 기준 `i64` 나노초입니다. 그래서 표현 범위는 모든 타깃에서 약 ±292년입니다. 32비트 타깃도 64비트 시간 인터페이스(`time64`)로 시계를 읽으므로 2038년 한계가 없습니다.

`SystemTime`의 성격은 꽤 다릅니다. NTP 보정이나 사용자 변경으로 시간이 뒤로 갈 수 있습니다. UNIX_EPOCH 기준의 절대 시간이므로 **직렬화할 수 있습니다**. 그래서 프로세스 사이에서도 의미 있게 비교할 수 있습니다.

### 사용 예

Unix 타임스탬프 변환과 파일 mtime 조회는 다음과 같습니다.

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

## 3.3.4. `BootTime`: 부팅 후 시간(suspend 포함)

```rok
struct BootTime { /* CLOCK_BOOTTIME (Linux) / CLOCK_MONOTONIC = mach_continuous_time (macOS) */ }

impl BootTime {
    fn now() -> BootTime;
    fn elapsed(self) -> Duration;
    fn duration_since(self, earlier: BootTime) -> Duration;
}
```

`BootTime`과 `Instant`는 절전(suspend) 동안의 동작이 다릅니다. `Instant`는 Linux `CLOCK_MONOTONIC`처럼 절전 동안 멈춥니다. `BootTime`은 `CLOCK_BOOTTIME`을 따라 절전 중에도 계속 흐릅니다.

이 차이는 모바일이나 노트북 앱이 "실제 경과 시간(절전 포함)"을 측정해야 할 때 결정적입니다. Rust에는 2026년 현재도 이를 위한 안정 API가 없습니다. Rok은 처음부터 노출합니다.

## 3.3.5. `Duration`

```rok
struct Duration { ns: i64; }       // i64 nanosecond - 약 ±292년

impl Duration {
    const ZERO: Duration = Duration { ns: 0 };
    const MAX: Duration = Duration { ns: I64_MAX };

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

`Display`는 크기에 맞는 단위 하나로 씁니다. 절댓값이 1초 이상이면 `s`, 1 ms 이상이면 `ms`, 1 µs 이상이면 `µs`, 그 밖에는 `ns`입니다. 그 단위로 나눈 정수부 뒤에 나머지를 소수로 붙입니다. 끝의 0은 지우고, 나머지가 0이면 소수점도 쓰지 않습니다. 음수는 앞에 `-`를 붙입니다.

```rok
Duration::ms(1234);      // "1.234s"
Duration::ms(5);         // "5ms"
Duration::ns(1500);      // "1.5µs"
Duration::ns(-20);       // "-20ns"
```

시간 타입은 모두 `PartialEq` / `Eq` / `PartialOrd` / `Ord` / `Hash`를 구현합니다. `Instant`, `SystemTime`, `BootTime`은 `Debug`도 구현합니다.

### Duration 산술 정책

`Duration` / `SystemTime` / `Instant` / `BootTime`의 산술(`duration_since`, `from_unix` 포함)은 오버플로 검사 설정을 따릅니다(`-C overflow-checks`, [refs-lang 9.1.2](../../refs-lang/009-tools/901-rokc-cli.md#rokc-build-aot-빌드)). 검사가 켜져 있으면 오버플로에서 panic이 일어나고, 꺼져 있으면 포화합니다. 0으로 나누면 항상 panic이 일어납니다. `checked_*` / `saturating_*` 변형으로 동작을 명시적으로 고를 수 있습니다.

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

## 3.3.6. 정책: TAI / 윤초 미지원

`SystemTime`은 엄밀히 **UTC**이고, 윤초(leap second)는 처리하지 않습니다. Linux 커널 자체가 이 부분에서 모호하다는 점도 고려했습니다. TAI(원자시)나 정확한 윤초 처리가 필요하면 별도 패키지(`rok/chrono` 또는 `rok/tai`)를 씁니다.

| 자리                       | 권장                      |
|----------------------------|---------------------------|
| 일반 앱 (타임스탬프 / 측정) | rok::std::time            |
| 시뮬레이션 / 정밀 천체     | 별도 `chrono`-like 패키지 |
| 금융 / 정확한 윤초         | 별도 패키지               |

## 3.3.7. 구현 / 타깃

플랫폼별 구현 매핑은 다음과 같습니다.

| 항목         | linux           | macos            | windows                 | wasi         |
|--------------|-----------------|------------------|-------------------------|--------------|
| `Instant`    | CLOCK_MONOTONIC | CLOCK_UPTIME_RAW | QueryPerformanceCounter | wasi clock   |
| `SystemTime` | CLOCK_REALTIME  | CLOCK_REALTIME   | GetSystemTimePreciseAsFileTime | wasi  |
| `BootTime`   | CLOCK_BOOTTIME  | CLOCK_MONOTONIC (mach_continuous_time, 절전 포함) | QueryPerformanceCounter (~ Instant) | (절전 없음) |

## 3.3.8. 정책 요약

이 장의 핵심 결정은 세 가지입니다. 첫째, `Instant` / `SystemTime` / `Duration`을 나눠 단조 시간과 벽시계 시간을 의미상 구분합니다. 둘째, Rust의 std에 없는 `BootTime`을 따로 노출합니다. 그래서 절전을 포함한 부팅 후 경과 시간을 직접 읽을 수 있습니다. 셋째, TAI와 윤초는 이 모듈 밖의 별도 패키지로 나눕니다.
