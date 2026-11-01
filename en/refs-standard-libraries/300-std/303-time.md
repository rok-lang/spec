# 3.3. `time::` - time

Time is split across two modules by what it needs. `rok::core::time` holds `Duration` and `Instant`: the monotonic clock already runs in `rok::core` (task timers, 2.18), so both work on every target and need no `use` (0.2). `rok::std::time` holds the clocks that need the operating system: the wall clock `SystemTime`, the boot clock `BootTime` and `UNIX_EPOCH`. Each type has one path ([0.1.6](../000-meta/001-tier-model.md#016-one-path-per-item)); `rok::std::time` does not re-export the core types. The monotonic clock, the wall clock and time elapsed since boot are kept clearly separate, so the clock that fits the measurement can be chosen.

## 3.3.1. The four-type model

The overall picture:

| Type         | Module            | Meaning                     | Used for                 |
|--------------|-------------------|-----------------------------|--------------------------|
| `Instant`    | `rok::core::time` | Monotonic clock             | Duration measurement / timeouts |
| `Duration`   | `rok::core::time` | Time difference (i64 nanoseconds) | Difference between two times / sleep argument |
| `SystemTime` | `rok::std::time`  | Wall clock (UTC, NTP-adjustable) | File mtime / display to users |
| `BootTime`   | `rok::std::time`  | Time since boot (including suspend) | Mobile / laptop apps |

The internal representation of `Duration` is `i64` ns. The `u128` that Rust adopted was judged excessive; `i64` already covers about ±292 years, which is ample for practical use.

**One rule for every clock.** The span between two readings of any clock is the exact signed `Duration`: `a.duration_since(b)` is negative when `b` is later, never an error, `none` or zero. `elapsed()` measures on a monotonic clock, so a step of the wall clock cannot make it negative. This is Go's rule (`time.Sub`, `time.Since`).

## 3.3.2. `Instant` - monotonic clock

```rok
// rok::core::time
struct Instant { /* opaque (CLOCK_MONOTONIC / QueryPerformanceCounter) */ }

impl Instant {
    fn now() -> Instant;

    fn elapsed(self) -> Duration;
    const fn duration_since(self, earlier: Instant) -> Duration;   // self - earlier, signed

    // arithmetic: returns none on failure (avoids Rust's silent saturating-on-overflow)
    const fn checked_add(self, dur: Duration) -> Instant | none;
    const fn checked_sub(self, dur: Duration) -> Instant | none;
}

// ordinary arithmetic; on overflow, panic with overflow checks on, saturate with them off (3.3.5)
impl Add<Duration> for Instant { type Output = Instant; ... }
impl Sub<Instant> for Instant { type Output = Duration; ... }
impl Sub<Duration> for Instant { type Output = Instant; ... }
```

The key properties of `Instant`: it never goes backward, so it is unaffected by NTP adjustment. It has no absolute time meaning of its own (its reference is process start), so it **cannot be serialized**. For the same reason, comparisons across processes are meaningless; it is meaningful only within the same process.

`duration_since` returns a negative `Duration` when `earlier` is later. To clamp a span, compare it: `d.max(Duration::ZERO)`.

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
// rok::std::time
struct SystemTime { /* opaque (CLOCK_REALTIME / GetSystemTimePreciseAsFileTime) */ }

impl SystemTime {
    fn now() -> SystemTime;

    const fn duration_since(self, earlier: SystemTime) -> Duration;   // self - earlier, signed
    fn elapsed(self) -> Duration;                                    // on the monotonic clock

    const fn checked_add(self, dur: Duration) -> SystemTime | none;
    const fn checked_sub(self, dur: Duration) -> SystemTime | none;
}

const UNIX_EPOCH: SystemTime;     // 1970-01-01 00:00:00 UTC

// SystemTime ↔ unix timestamp
impl SystemTime {
    const fn from_unix(secs: i64, nanos: u32) -> SystemTime;
    const fn unix_secs(self) -> i64;    // rounded down: negative before the epoch
    const fn unix_nanos(self) -> i64;   // rok has no i128; i64 ns covers about ±292 years
}

impl Add<Duration> for SystemTime { type Output = SystemTime; ... }
impl Sub<Duration> for SystemTime { type Output = SystemTime; ... }
```

`SystemTime` is `i64` nanoseconds relative to `UNIX_EPOCH`, so its range is about ±292 years on every target. 32-bit targets read the clock through 64-bit time interfaces (`time64`), so there is no 2038 limit.

`SystemTime` behaves quite differently. Time may go backward through NTP adjustment or user changes, and since it is an absolute time relative to UNIX_EPOCH it **can be serialized**. It can therefore be compared meaningfully across processes.

`duration_since` is the signed difference of the wall readings: a later `earlier` gives a negative `Duration`, not an error. `unix_secs` rounds down, so `SystemTime::from_unix(t.unix_secs(), n)` gives `t` back for its sub-second part `n`, also before the epoch.

**Monotonic reading.** `SystemTime::now()` also reads the monotonic clock (`Instant`'s). `elapsed()` measures on that reading, so a step of the wall clock between `now()` and `elapsed()` does not change the result. `+` / `-` / `checked_*` with a `Duration` move both readings. A `SystemTime` that did not come from `now()` (`from_unix`, `UNIX_EPOCH`, `Metadata::modified`) has none, and its `elapsed()` is `SystemTime::now().duration_since(self)`. Comparison, equality and hashing use the wall reading only.

### Example

Converting to a unix timestamp and reading a file's mtime:

```rok
use rok::std::time::SystemTime;

let now = SystemTime::now();
log(f"unix timestamp: {now.unix_secs()}");

// file mtime
let meta = try fs::metadata("data.txt");
let mtime: SystemTime = try meta.modified();
log(f"file age: {mtime.elapsed():?}");     // negative if the mtime is in the future
```

## 3.3.4. `BootTime` - time since boot (including suspend)

```rok
// rok::std::time
struct BootTime { /* CLOCK_BOOTTIME (Linux) / CLOCK_MONOTONIC = mach_continuous_time (macOS) */ }

impl BootTime {
    fn now() -> BootTime;
    fn elapsed(self) -> Duration;
    const fn duration_since(self, earlier: BootTime) -> Duration;   // self - earlier, signed
}
```

As with `Instant`, `duration_since` is negative when `earlier` is later. `BootTime` and `Instant` differ in their behavior across suspend. `Instant`, like Linux `CLOCK_MONOTONIC`, stops during suspend, whereas `BootTime` follows `CLOCK_BOOTTIME` and keeps advancing during suspend.

This difference is decisive when a mobile or laptop app must measure "real elapsed time (including suspend)". Rust still has no stable API for this as of 2026; Rok exposes it on the surface early.

## 3.3.5. `Duration`

```rok
// rok::core::time
struct Duration { /* opaque: i64 nanoseconds (about ±292 years) */ }

impl Duration {
    const ZERO: Duration;             // 0 ns
    const MAX: Duration;              // I64_MAX ns

    // constructors
    const fn ns(n: i64) -> Duration;
    const fn us(n: i64) -> Duration;        // microseconds
    const fn ms(n: i64) -> Duration;        // milliseconds
    const fn secs(n: i64) -> Duration;
    const fn mins(n: i64) -> Duration;
    const fn hours(n: i64) -> Duration;
    const fn days(n: i64) -> Duration;

    const fn secs_f64(s: f64) -> Duration;  // floating point seconds, truncated to ns

    // extraction
    const fn as_ns(self) -> i64;
    const fn as_us(self) -> i64;
    const fn as_ms(self) -> i64;
    const fn as_secs(self) -> i64;
    const fn as_secs_f64(self) -> f64;

    const fn subsec_ns(self) -> u32;          // sub-second nanosecond part
    const fn subsec_us(self) -> u32;
    const fn subsec_ms(self) -> u32;
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

`a.min(b)` on two durations is `Ord::min`.

`Display` writes a single unit fitting the magnitude: `s` if the absolute value is at least 1 second, `ms` if at least 1 ms, `µs` if at least 1 µs, and `ns` otherwise. The integer part divided by that unit is followed by the remainder as a fraction; trailing zeros are removed, and if the remainder is 0 no decimal point is written. Negative values are prefixed with `-`.

```rok
Duration::ms(1234);      // "1.234s"
Duration::ms(5);         // "5ms"
Duration::ns(1500);      // "1.5µs"
Duration::ns(-20);       // "-20ns"
```

All time types implement `Clone` / `Copy` / `PartialEq` / `Eq` / `PartialOrd` / `Ord` / `Hash` / `Debug`. `Duration` also implements `Default` (zero).

### Duration arithmetic policy

Arithmetic on `Duration` / `SystemTime` / `Instant` / `BootTime` (including `duration_since` and `from_unix`) panics on overflow when overflow checks are on and saturates when they are off (`-C overflow-checks`, [refs-lang 9.1.2](../../refs-lang/009-tools/901-rokc-cli.md#rokc-build-aot-build)). The `f64` forms (`Duration::secs_f64`, `f64 * Duration`) follow the same rule: a non-finite or out-of-range result panics with checks on and saturates (NaN to zero) with them off. Division by zero always panics. The `checked_*` / `saturating_*` variants of `Duration` choose explicitly.

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

## 3.3.6. Policy - no TAI / leap second support

`SystemTime` is strictly **UTC** and does not handle leap seconds (taking into account that the Linux kernel itself is ambiguous here). Cases that need TAI (atomic time) or exact leap second handling belong to separate packages (`rok/chrono` or `rok/tai`).

| Use                        | Recommendation            |
|----------------------------|---------------------------|
| Ordinary apps (timestamps / measurement) | rok::core::time, rok::std::time |
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

The chapter's key decisions are four. First, `Instant` / `SystemTime` / `Duration` are separated, distinguishing monotonic time from wall-clock time semantically; `Duration` and `Instant` live in `rok::core::time`, the OS clocks in `rok::std::time`. Second, every span is the exact signed `Duration`, and `elapsed()` runs on a monotonic clock. Third, `BootTime`, which Rust's std does not cover, is exposed separately so that elapsed time since boot, including suspend, can be read directly. Fourth, TAI and leap seconds are split out into separate packages outside this module.
