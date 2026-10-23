# 3.3. `time::` - time

The `rok::std::time` module provides three kinds of time types plus `Duration`. The monotonic clock, the wall clock and time elapsed since boot are kept clearly separate, so the clock that fits the measurement can be chosen.

## 3.3.1. The four-type model

The overall picture:

| Type         | Meaning                     | Used for                 |
|--------------|-----------------------------|--------------------------|
| `Instant`    | Monotonic clock             | Duration measurement / timeouts |
| `SystemTime` | Wall clock (UTC, NTP-adjustable) | File mtime / display to users |
| `BootTime`   | Time since boot (including suspend) | Mobile / laptop apps |
| `Duration`   | Time difference (i64 nanoseconds) | Difference between two times / sleep argument |

The internal representation of `Duration` is `i64` ns. The `u128` that Rust adopted was judged excessive; `i64` already covers about ±292 years, which is ample for practical use.

## 3.3.2. `Instant` - monotonic clock

```rok
struct Instant { /* opaque - CLOCK_MONOTONIC / QueryPerformanceCounter */ }

impl Instant {
    fn now() -> Instant;

    fn elapsed(self) -> Duration;
    fn duration_since(self, earlier: Instant) -> Duration;       // self - earlier
    fn checked_duration_since(self, earlier: Instant) -> Duration | none;
    fn saturating_duration_since(self, earlier: Instant) -> Duration;

    // arithmetic - returns none on failure (avoids Rust's silent saturating-on-overflow)
    fn checked_add(self, dur: Duration) -> Instant | none;
    fn checked_sub(self, dur: Duration) -> Instant | none;
}

// ordinary arithmetic - on overflow: panic with overflow checks on, saturate with them off (3.3.5)
impl Add<Duration> for Instant { type Output = Instant; ... }
impl Sub<Instant> for Instant { type Output = Duration; ... }
impl Sub<Duration> for Instant { type Output = Instant; ... }
```

The key properties of `Instant`: it never goes backward, so it is unaffected by NTP adjustment. It has no absolute time meaning of its own (its reference is process start), so it **cannot be serialized**. For the same reason, comparisons across processes are meaningless; it is meaningful only within the same process.

### Example

The most common pattern is to take a starting point and get the elapsed time with `elapsed`.

```rok
let start = Instant::now();
heavy_computation();
let elapsed = start.elapsed();          // Duration
log(f"computation took {elapsed:?}");
```

## 3.3.3. `SystemTime` - wall clock

```rok
struct SystemTime { /* opaque - CLOCK_REALTIME / GetSystemTimeAsFileTime */ }

impl SystemTime {
    fn now() -> SystemTime;

    fn duration_since(self, earlier: SystemTime) throws (SystemTimeError) -> Duration;
                                            // Err if earlier > self

    fn elapsed(self) throws (SystemTimeError) -> Duration;

    fn checked_add(self, dur: Duration) -> SystemTime | none;
    fn checked_sub(self, dur: Duration) -> SystemTime | none;
}

const UNIX_EPOCH: SystemTime;     // 1970-01-01 00:00:00 UTC

// SystemTime ↔ unix timestamp
impl SystemTime {
    fn from_unix(secs: i64, nanos: u32) -> SystemTime;
    fn unix_seconds(self) throws (SystemTimeError) -> i64;
    fn unix_nanos(self) throws (SystemTimeError) -> i64;   // rok has no i128 - i64 ns covers about ±292 years
}

impl Add<Duration> for SystemTime { type Output = SystemTime; ... }
impl Sub<Duration> for SystemTime { type Output = SystemTime; ... }

struct SystemTimeError {
    duration: Duration;        // how far earlier was ahead
}
```

`SystemTime` is `i64` nanoseconds relative to `UNIX_EPOCH`, so its range is about ±292 years on every target. 32-bit targets read the clock through 64-bit time interfaces (`time64`), so there is no 2038 limit.

`SystemTime` behaves quite differently. Time may go backward through NTP adjustment or user changes, and since it is an absolute time relative to UNIX_EPOCH it **can be serialized**. It can therefore be compared meaningfully across processes.

### Example

Converting to a unix timestamp and reading a file's mtime:

```rok
let now = SystemTime::now();
let unix = try now.unix_seconds();
log(f"unix timestamp: {unix}");

// file mtime
let meta = try fs::metadata("data.txt");
let mtime: SystemTime = try meta.modified();
let age = try SystemTime::now().duration_since(mtime);
log(f"file age: {age:?}");
```

## 3.3.4. `BootTime` - time since boot (including suspend)

```rok
struct BootTime { /* CLOCK_BOOTTIME (Linux) / CLOCK_MONOTONIC = mach_continuous_time (macOS) */ }

impl BootTime {
    fn now() -> BootTime;
    fn elapsed(self) -> Duration;
    fn duration_since(self, earlier: BootTime) -> Duration;
}
```

`BootTime` and `Instant` differ in their behavior across suspend. `Instant`, like Linux `CLOCK_MONOTONIC`, stops during suspend, whereas `BootTime` follows `CLOCK_BOOTTIME` and keeps advancing during suspend.

This difference is decisive when a mobile or laptop app must measure "real elapsed time (including suspend)". Rust still has no stable API for this as of 2026; rok exposes it on the surface early.

## 3.3.5. `Duration`

```rok
struct Duration { ns: i64; }       // i64 nanoseconds - about ±292 years

impl Duration {
    const ZERO: Duration = Duration { ns: 0 };
    const MAX: Duration = Duration { ns: I64_MAX };

    // constructors
    fn ns(n: i64) -> Duration;
    fn us(n: i64) -> Duration;        // microseconds
    fn ms(n: i64) -> Duration;        // milliseconds
    fn sec(n: i64) -> Duration;
    fn min(n: i64) -> Duration;
    fn hours(n: i64) -> Duration;
    fn days(n: i64) -> Duration;

    fn from_secs_f64(s: f64) -> Duration;        // floating point seconds

    // extraction
    fn as_ns(self) -> i64;
    fn as_us(self) -> i64;
    fn as_ms(self) -> i64;
    fn as_secs(self) -> i64;
    fn as_secs_f64(self) -> f64;

    fn subsec_ns(self) -> u32;          // sub-second nanosecond part
    fn subsec_us(self) -> u32;
    fn subsec_ms(self) -> u32;
}

// arithmetic
impl Add for Duration { ... }
impl Sub for Duration { ... }
impl Mul<i64> for Duration { ... }     // Duration × i64
impl Div<i64> for Duration { ... }
impl Mul<Duration> for f64 { ... }      // f64 × Duration

impl Display for Duration { ... }       // "1.234s" / "5ms" / "1.5µs" / "100ns"
impl Debug for Duration { ... }         // same as Display
```

`Display` writes a single unit fitting the magnitude: `s` if the absolute value is at least 1 second, `ms` if at least 1 ms, `µs` if at least 1 µs, and `ns` otherwise. The integer part divided by that unit is followed by the remainder as a fraction; trailing zeros are removed, and if the remainder is 0 no decimal point is written. Negative values are prefixed with `-`.

```rok
Duration::ms(1234);      // "1.234s"
Duration::ms(5);         // "5ms"
Duration::ns(1500);      // "1.5µs"
Duration::ns(-20);       // "-20ns"
```

All time types implement `PartialEq` / `Eq` / `PartialOrd` / `Ord` / `Hash`. `Instant`, `SystemTime` and `BootTime` also implement `Debug`.

### Duration arithmetic policy

Arithmetic on `Duration` / `SystemTime` / `Instant` / `BootTime` (including `duration_since` and `from_unix`) panics on overflow when overflow checks are on and saturates when they are off (`-C overflow-checks`, [refs-lang 9.1.2](../../refs-lang/009-tools/901-rokc-cli.md#rokc-build-aot-build)). Division by zero always panics. The `checked_*` / `saturating_*` variants choose explicitly.

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

## 3.3.6. Policy - no TAI / leap second support

`SystemTime` is strictly **UTC** and does not handle leap seconds (taking into account that the Linux kernel itself is ambiguous here). Cases that need TAI (atomic time) or exact leap second handling belong to separate packages (`rok/chrono` or `rok/tai`).

| Use                        | Recommendation            |
|----------------------------|---------------------------|
| Ordinary apps (timestamps / measurement) | rok::std::time |
| Simulation / precise astronomy | A separate `chrono`-like package |
| Finance / exact leap seconds | A separate package      |

## 3.3.7. impl / target

Per-platform implementation mapping:

| Item         | linux           | macos            | windows                 | wasi         |
|--------------|-----------------|------------------|-------------------------|--------------|
| `Instant`    | CLOCK_MONOTONIC | CLOCK_UPTIME_RAW | QueryPerformanceCounter | wasi clock   |
| `SystemTime` | CLOCK_REALTIME  | CLOCK_REALTIME   | GetSystemTimePreciseAsFileTime | wasi  |
| `BootTime`   | CLOCK_BOOTTIME  | CLOCK_MONOTONIC (mach_continuous_time, includes sleep) | QueryPerformanceCounter (~ Instant) | (no suspend) |

## 3.3.8. Policy summary

The chapter's key decisions are three. First, `Instant` / `SystemTime` / `Duration` are separated, distinguishing monotonic time from wall-clock time semantically. Second, `BootTime`, which Rust's std does not cover, is exposed separately so that elapsed time since boot, including suspend, can be read directly. Third, TAI and leap seconds are split out into separate packages outside this module.
