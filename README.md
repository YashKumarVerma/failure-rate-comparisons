# Failure rate comparisons

An interactive simulator for picking how to compute a per-stream error rate (for example, for a circuit breaker) when the state lives in Redis. It runs EWMA variants and sliding windows side by side on the same synthetic traffic, and shows how accurate each one is and what it would cost on Redis.

Everything is in one file, `index.html`, with no dependencies or build step. Open it in a browser.

## What it compares

| Algorithm | How it works | With no traffic |
|---|---|---|
| Per-event time-decayed | `v += (1 − e^(−Δt/τ)) · (x − v)` on every request | ratio freezes |
| Per-event fixed α | `v += α · (x − v)` on every request, ignores time | ratio freezes |
| Fixed-tick | count per interval Δ, then `v += α · (rate − v)` on a ticker | rates decay to 0, ratio is 0/0 |
| Decayed counts (recommended) | two decayed counters, failures `f` and requests `a`; ratio = `f / a` | ratio freezes, rates decay |
| Sliding window, zset | every request stored in a Redis sorted set for W seconds | window empties |
| Sliding window, buckets | one counter key per time bucket, summed over W | window empties |

## Simulator tab

- **Traffic:** success/s and failure/s per stream, Poisson or uniform arrivals, 1–10× speed.
- **Load:** reads/s (or one read per request, like a breaker check) and the number of streams.
- **Presets:** 500 rps at 10% errors, 1000 rps at 1%, an error spike (1% → 50% → 1%), low traffic, traffic stopping, and a 10,000-stream fleet.
- **Cost vs accuracy table:** Redis round trips/s, commands/s, an estimate of server-side work, memory, whether a background ticker is needed, and error vs the true value (normalised to % of truth so every row compares).
- **Charts:** error ratio (with the observed per-0.5 s ratio as dots), rates, Redis commands/s, and rolling error vs truth.
- **Estimator cards:** add any algorithm more than once to compare settings. Every parameter, column and formula has a hover popup explaining it.

## Dev handover tab

Implementation notes for each algorithm: Redis key layout, Lua scripts, read path, Go code (go-redis v9) for the recommended option, how to choose τ, scaling advice, and a list of tests. The Lua scripts were run against Redis 7.

Recommended approach: **decayed counts**. One hash per stream (`f`, `a`, `ts`), one Lua call per request or per batch, no background job. It returns both the error ratio and the request rate, so a breaker can refuse to trip on low volume.

Time for the estimate to cross a threshold θ after the true ratio jumps from p₀ to p₁:

```
t = τ · ln((p₁ − p₀) / (p₁ − θ))
```

For τ = 10 s, a jump from 1% to 50% and a 25% threshold, t ≈ 6.7 s.

## Redis cost per stream

| Algorithm | Write per request | Read | Memory |
|---|---|---|---|
| Decayed counts | 1 round trip: HMGET, HSET, EXPIRE | 1 round trip: HMGET | ~144 B |
| Per-event time-decayed | 1 round trip: HMGET, HSET, EXPIRE | HGET | ~120 B |
| Fixed-tick | HINCRBY | HMGET | ~150 B, plus a ticker script per stream per interval |
| Sliding window, zset | 1 round trip: ZADD, ZREMRANGEBYSCORE, PEXPIRE (×2 on failure) | 2 × ZCOUNT | ~120 B × requests in the window |
| Sliding window, buckets | INCR + EXPIRE (×2 on failure) | MGET of 2·W/bucket keys | ~70 B × 2·W/bucket |
| Per-event fixed α | 1 round trip: GET, SET | GET | ~60 B |

The zset, decayed counts and time-decayed sizes were measured with `MEMORY USAGE` on Redis 7; the others are estimates. Real usage is 10–30% higher once expiry bookkeeping and allocator overhead are included.

## Checking changes

The page runs a self-test on load (`console.assert`): every algorithm must settle within 10% of the true value on steady traffic. To run it without a browser:

```bash
sed -n '/<script>/,/<\/script>/p' index.html | sed '1d;$d' > /tmp/sim.js && node /tmp/sim.js
```

No output means it passed. See `CLAUDE.md` for the code structure and how to add an algorithm.
