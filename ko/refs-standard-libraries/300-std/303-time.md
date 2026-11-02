# 3.3. `time::`: 시간

시간은 필요한 것에 따라 두 모듈로 나뉩니다. `rok::core::time`에는 `Duration`과 `Instant`가 있습니다. 단조 시계는 이미 `rok::core`에서 돌고 있으므로(태스크 타이머, 2.18) 두 타입은 모든 타깃에서 동작하고 `use` 없이 씁니다(0.2). `rok::std::time`에는 운영체제가 필요한 시계가 있습니다. 벽시계 `SystemTime`, 부팅 시계 `BootTime`, `UNIX_EPOCH`입니다. 타입마다 경로는 하나입니다([0.1.6](../000-meta/001-tier-model.md#016-항목마다-경로-하나)). `rok::std::time`은 core 타입을 다시 내보내지 않습니다. 단조 시계, 벽시계, 부팅 후 경과 시간을 명확히 나눠 두었습니다. 그래서 측정 목적에 맞는 시계를 고를 수 있습니다.

## 3.3.1. 타입 4개 모델

전체 구성은 다음과 같습니다.

| 타입         | 모듈              | 의미                        | 사용처                   |
|--------------|-------------------|-----------------------------|--------------------------|
| `Instant`    | `rok::core::time` | 단조 시계 (monotonic)       | 경과 시간 측정 / 타임아웃 |
| `Duration`   | `rok::core::time` | 시간 차이 (i64 나노초)      | 둘 사이 차 / sleep 인자  |
| `SystemTime` | `rok::std::time`  | 벽시계 (UTC, NTP 가능)      | 파일 mtime / 사용자 표시 |
| `BootTime`   | `rok::std::time`  | 부팅 후 시간 (절전 포함)    | 모바일 / 노트북 앱       |

`Duration`의 내부 표현은 `i64` ns입니다. Rust가 채택한 `u128`은 과하다고 판단했습니다. `i64`로도 약 ±292년을 표현할 수 있으므로 실용 범위로는 충분합니다.

**모든 시계에 같은 규칙.** 어느 시계든 두 값 사이의 간격은 부호 있는 정확한 `Duration`입니다. `b`가 더 나중이면 `a.duration_since(b)`는 음수입니다. 오류도, `none`도, 0도 아닙니다. `elapsed()`는 단조 시계로 재므로 벽시계가 바뀌어도 음수가 되지 않습니다. Go의 규칙(`time.Sub`, `time.Since`)과 같습니다.

## 3.3.2. `Instant`: 단조 시계

```rok
// rok::core::time
struct Instant { /* opaque (CLOCK_MONOTONIC / QueryPerformanceCounter) */ }

impl Instant {
    fn now() -> Instant;

    fn elapsed(self) -> Duration;
    const fn duration_since(self, earlier: Instant) -> Duration;   // self - earlier, 부호 있음

    // 산술: 실패 시 none 반환 (Rust 의 saturating-on-overflow silent 회피)
    const fn checked_add(self, dur: Duration) -> Instant | none;
    const fn checked_sub(self, dur: Duration) -> Instant | none;
}

// 일반 산술. overflow 시 overflow 검사가 켜져 있으면 panic, 꺼져 있으면 saturate (3.3.5)
impl Add<Duration> for Instant { type Output = Instant; ... }
impl Sub<Instant> for Instant { type Output = Duration; ... }
impl Sub<Duration> for Instant { type Output = Instant; ... }
```

`Instant`의 핵심 특성은 다음과 같습니다. 절대 뒤로 가지 않으므로 NTP 보정의 영향을 받지 않습니다. 기준이 프로세스 시작 시점이라 그 자체로는 절대 시간의 의미가 없습니다. 그래서 **직렬화할 수 없습니다**. 같은 이유로 프로세스 사이의 비교도 무의미합니다. 같은 프로세스 안에서만 의미가 있습니다.

`earlier`가 더 나중이면 `duration_since`는 음수 `Duration`을 돌려줍니다. 간격을 0 아래로 내려가지 않게 하려면 비교합니다. `d.max(Duration::ZERO)`.

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
// rok::std::time
struct SystemTime { /* opaque (CLOCK_REALTIME / GetSystemTimePreciseAsFileTime) */ }

impl SystemTime {
    fn now() -> SystemTime;

    const fn duration_since(self, earlier: SystemTime) -> Duration;   // self - earlier, 부호 있음
    fn elapsed(self) -> Duration;                                    // 단조 시계로 잼

    const fn checked_add(self, dur: Duration) -> SystemTime | none;
    const fn checked_sub(self, dur: Duration) -> SystemTime | none;
}

const UNIX_EPOCH: SystemTime;     // 1970-01-01 00:00:00 UTC

// SystemTime ↔ unix timestamp
impl SystemTime {
    const fn from_unix(secs: i64, nanos: u32) -> SystemTime;
    const fn unix_secs(self) -> i64;    // 내림: epoch 이전이면 음수
    const fn unix_nanos(self) -> i64;   // rok 에 i128 이 없음. i64 ns 는 약 ±292 년
}

impl Add<Duration> for SystemTime { type Output = SystemTime; ... }
impl Sub<Duration> for SystemTime { type Output = SystemTime; ... }
```

`SystemTime`은 `UNIX_EPOCH` 기준 `i64` 나노초입니다. 그래서 표현 범위는 모든 타깃에서 약 ±292년입니다. 32비트 타깃도 64비트 시간 인터페이스(`time64`)로 시계를 읽으므로 2038년 한계가 없습니다.

`SystemTime`의 성격은 꽤 다릅니다. NTP 보정이나 사용자 변경으로 시간이 뒤로 갈 수 있습니다. UNIX_EPOCH 기준의 절대 시간이므로 **직렬화할 수 있습니다**. 그래서 프로세스 사이에서도 의미 있게 비교할 수 있습니다.

`duration_since`는 벽시계 값의 부호 있는 차입니다. `earlier`가 더 나중이면 오류가 아니라 음수 `Duration`이 나옵니다. `unix_secs`는 내림하므로 epoch 이전에도 `SystemTime::from_unix(t.unix_secs(), n)`이 초 아래 부분 `n`과 함께 `t`를 그대로 돌려줍니다.

**단조 시계 값.** `SystemTime::now()`는 단조 시계(`Instant`의 시계)도 함께 읽습니다. `elapsed()`는 이 값으로 잽니다. 그래서 `now()`와 `elapsed()` 사이에 벽시계가 바뀌어도 결과는 달라지지 않습니다. `Duration`과의 `+` / `-` / `checked_*`는 두 값을 함께 옮깁니다. `now()`에서 오지 않은 `SystemTime`(`from_unix`, `UNIX_EPOCH`, `Metadata::modified`)에는 단조 시계 값이 없고, 그 `elapsed()`는 `SystemTime::now().duration_since(self)`입니다. 비교, 같음, 해시는 벽시계 값만 봅니다.

### 사용 예

Unix 타임스탬프 변환과 파일 mtime 조회는 다음과 같습니다.

```rok
use rok::std::time::SystemTime;

let now = SystemTime::now();
log(f"unix timestamp: {now.unix_secs()}");

// 파일 mtime
let meta = try fs::metadata("data.txt");
let mtime: SystemTime = try meta.modified();
log(f"file age: {mtime.elapsed():?}");     // mtime 이 미래면 음수
```

## 3.3.4. `BootTime`: 부팅 후 시간(suspend 포함)

```rok
// rok::std::time
struct BootTime { /* CLOCK_BOOTTIME (Linux) / CLOCK_MONOTONIC = mach_continuous_time (macOS) */ }

impl BootTime {
    fn now() -> BootTime;
    fn elapsed(self) -> Duration;
    const fn duration_since(self, earlier: BootTime) -> Duration;   // self - earlier, 부호 있음
}
```

`Instant`처럼 `earlier`가 더 나중이면 `duration_since`는 음수입니다. `BootTime`과 `Instant`는 절전(suspend) 동안의 동작이 다릅니다. `Instant`는 Linux `CLOCK_MONOTONIC`처럼 절전 동안 멈춥니다. `BootTime`은 `CLOCK_BOOTTIME`을 따라 절전 중에도 계속 흐릅니다.

이 차이는 모바일이나 노트북 앱이 "실제 경과 시간(절전 포함)"을 측정해야 할 때 결정적입니다. Rust에는 2026년 현재도 이를 위한 안정 API가 없습니다. Rok은 처음부터 노출합니다.

## 3.3.5. `Duration`

```rok
// rok::core::time
struct Duration { /* opaque: i64 nanosecond (약 ±292년) */ }

impl Duration {
    const ZERO: Duration;             // 0 ns
    const MAX: Duration;              // I64_MAX ns

    // 생성자: i64로 손실 없이 바뀌는 정수 타입이면 무엇이든
    // (i8 .. i64, u8 .. u32), u64 / usize는 try_into
    const fn ns<N>(n: N) -> Duration where i64: From<N>;
    const fn us<N>(n: N) -> Duration where i64: From<N>;        // microseconds
    const fn ms<N>(n: N) -> Duration where i64: From<N>;        // milliseconds
    const fn secs<N>(n: N) -> Duration where i64: From<N>;
    const fn mins<N>(n: N) -> Duration where i64: From<N>;
    const fn hours<N>(n: N) -> Duration where i64: From<N>;
    const fn days<N>(n: N) -> Duration where i64: From<N>;

    const fn secs_f64(s: f64) -> Duration;  // 부동소수 초, ns 단위로 버림

    // 추출
    const fn as_ns(self) -> i64;
    const fn as_us(self) -> i64;
    const fn as_ms(self) -> i64;
    const fn as_secs(self) -> i64;
    const fn as_secs_f64(self) -> f64;

    const fn subsec_ns(self) -> u32;          // 하위 nanosecond 부분
    const fn subsec_us(self) -> u32;
    const fn subsec_ms(self) -> u32;
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

두 duration의 `a.min(b)`는 `Ord::min`입니다.

`Display`는 크기에 맞는 단위 하나로 씁니다. 절댓값이 1초 이상이면 `s`, 1 ms 이상이면 `ms`, 1 µs 이상이면 `µs`, 그 밖에는 `ns`입니다. 그 단위로 나눈 정수부 뒤에 나머지를 소수로 붙입니다. 끝의 0은 지우고, 나머지가 0이면 소수점도 쓰지 않습니다. 음수는 앞에 `-`를 붙입니다.

```rok
Duration::ms(1234);      // "1.234s"
Duration::ms(5);         // "5ms"
Duration::ns(1500);      // "1.5µs"
Duration::ns(-20);       // "-20ns"
```

시간 타입은 모두 `Clone` / `Copy` / `PartialEq` / `Eq` / `PartialOrd` / `Ord` / `Hash` / `Debug`를 구현합니다. `Duration`은 `Default`(0)도 구현합니다.

### Duration 산술 정책

`Duration` / `SystemTime` / `Instant` / `BootTime`의 산술(`duration_since`, `from_unix` 포함)은 오버플로 검사 설정을 따릅니다(`-C overflow-checks`, [refs-lang 9.1.2](../../refs-lang/009-tools/901-rokc-cli.md#rokc-build-aot-빌드)). 검사가 켜져 있으면 오버플로에서 panic이 일어나고, 꺼져 있으면 포화합니다. `f64` 형태(`Duration::secs_f64`, `f64 * Duration`)도 같은 규칙을 따릅니다. 결과가 유한하지 않거나 범위를 벗어나면 검사가 켜져 있을 때 panic이 일어나고, 꺼져 있을 때 포화합니다(NaN은 0). 0으로 나누면 항상 panic이 일어납니다. `Duration`의 `checked_*` / `saturating_*` 변형으로 동작을 명시적으로 고를 수 있습니다.

```rok
impl Duration {
    const fn checked_add(self, rhs: Duration) -> Duration | none;
    const fn checked_sub(self, rhs: Duration) -> Duration | none;
    const fn checked_mul(self, factor: i64) -> Duration | none;
    const fn checked_div(self, divisor: i64) -> Duration | none;

    const fn saturating_add(self, rhs: Duration) -> Duration;
    const fn saturating_sub(self, rhs: Duration) -> Duration;
    const fn saturating_mul(self, factor: i64) -> Duration;
}
```

## 3.3.6. 정책: TAI / 윤초 미지원

`SystemTime`은 엄밀히 **UTC**이고, 윤초(leap second)는 처리하지 않습니다. Linux 커널 자체가 이 부분에서 모호하다는 점도 고려했습니다. TAI(원자시)나 정확한 윤초 처리가 필요하면 별도 패키지(`rok/chrono` 또는 `rok/tai`)를 씁니다.

| 자리                       | 권장                      |
|----------------------------|---------------------------|
| 일반 앱 (타임스탬프 / 측정) | rok::core::time, rok::std::time |
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

이 장의 핵심 결정은 네 가지입니다. 첫째, `Instant` / `SystemTime` / `Duration`을 나눠 단조 시간과 벽시계 시간을 의미상 구분합니다. `Duration`과 `Instant`는 `rok::core::time`에, OS 시계는 `rok::std::time`에 있습니다. 둘째, 모든 간격은 부호 있는 정확한 `Duration`이고 `elapsed()`는 단조 시계로 잽니다. 셋째, Rust의 std에 없는 `BootTime`을 따로 노출합니다. 그래서 절전을 포함한 부팅 후 경과 시간을 직접 읽을 수 있습니다. 넷째, TAI와 윤초는 이 모듈 밖의 별도 패키지로 나눕니다.
