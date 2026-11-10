# cpu

Low-level CPU helpers for high-contention, lock-free Go code: cache-line-aware
index remapping and exponential backoff with a hardware `PAUSE`/`YIELD` spin
primitive on amd64/arm64.

## Install

```
go get github.com/tqtrungse/cpu@v0.1.0
```

## API

### `CacheRemap`

```go
func CacheRemap(index, capacity, entrySize uint32) uint32
```

Remaps a logical index into a slot index so that small, consecutively-indexed
entries get spread across different cache-line halves instead of clustering
together — reducing false sharing between goroutines operating on adjacent
slots (e.g. ring buffers, sharded counters).

- If `entrySize` is at least half a cache line (`CacheLineSize / 2`), spreading
  gives no benefit, so the plain `index % capacity` slot is returned.
- If `capacity` is a power of two, the result is a permutation of
  `[0, capacity)` computed by transposing the index between
  "groups" sized to fit half a cache line.
- If `capacity` is not a power of two, it falls back to `index % capacity`
  (still a valid permutation, just without the cache-line optimization).
- Precondition: `capacity > 0`.

Forked from [bytedance/gopkg](https://github.com/bytedance/gopkg).

`CacheLineSize` (`unsafe.Sizeof(cpu.CacheLinePad{})` from `golang.org/x/sys/cpu`)
is exported alongside it.

### `Backoff`

```go
type Backoff struct{ /* ... */ }

func (b *Backoff) Spin()
func (b *Backoff) Snooze()
func (b *Backoff) IsDone() bool
func (b *Backoff) Reset()
```

An exponential backoff helper for spin-wait / retry loops, intended to reduce
contention in lock-free algorithms. It is **not** a substitute for a proper
blocking synchronization primitive.

- **`Spin()`** — use when backing off because *another thread made progress*
  (e.g. a lock-free retry after a failed CAS). Issues a short, growing burst
  of `procYield` (hardware pause/yield) calls, capped at `1 << 6` cycles.
- **`Snooze()`** — use when *waiting* for another thread to make progress.
  Escalates through three tiers as `step` grows:
  1. hardware pause/yield (only if `GOMAXPROCS > 1`),
  2. `runtime.Gosched()`,
  3. a bounded real `time.Sleep`, growing from 50µs up to a 5ms cap — this
     tier exists so a goroutine can't be starved indefinitely by neighbors
     that only ever call `Gosched()` under `GOMAXPROCS` greater than the
     real CPU quota.
- **`IsDone()`** — returns `true` once backoff has spun long enough (10
  steps) that the caller should stop spinning and park on a different
  synchronization mechanism instead.
- **`Reset()`** — resets the internal step counter to reuse the struct.

### `procYield` (internal)

```go
func procYield(cycles int)
```

Not exported. Executes `cycles` hardware `PAUSE` (amd64, [amd64.s](amd64.s))
or `YIELD` (arm64, [arm64.s](arm64.s)) instructions to back off the CPU
without yielding the OS thread. On other architectures
([procyield_fallback.go](procyield_fallback.go)) it falls back to a plain
empty counting loop.

## Files

| File | Purpose |
|---|---|
| [cache.go](cache.go) | `CacheRemap`, `CacheLineSize` |
| [backoff.go](backoff.go) | `Backoff` type |
| [procyield.go](procyield.go) | `procYield` declaration (amd64/arm64) |
| [amd64.s](amd64.s) | `procYield` asm impl (amd64, `PAUSE`) |
| [arm64.s](arm64.s) | `procYield` asm impl (arm64, `YIELD`) |
| [procyield_fallback.go](procyield_fallback.go) | `procYield` fallback (other arches) |
| [backoff_test.go](backoff_test.go) | Tests for `Backoff` |
| [cache_test.go](cache_test.go) | Tests for `CacheRemap` |

## License

Apache-2.0 — Copyright (c) 2026 tqtrungse@gmail.com
