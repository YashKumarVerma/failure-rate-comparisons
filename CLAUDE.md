# Failure rate calculators

A single-page simulator that compares ways of computing a per-stream error rate (EWMA variants, sliding windows), what each costs on Redis, and a dev handover page with the Redis implementation. Used to pick the algorithm for a circuit-breaker style error-rate check backed by Redis.

## Files

- `index.html`: the whole app. HTML, CSS and JS in one file, no dependencies, no build. Open it in a browser.
- `CLAUDE.md`: this file.

Not a git repo.

## Published copy

The page is also published on an internal docs host; ids, URL and the push command are in `CLAUDE.local.md` (git-ignored). Constraints from that host that shaped the page:
- Theme follows `<html data-slashdocs-theme="light|dark">`; palettes are defined for both, plus `prefers-color-scheme` for local use. Don't add a theme toggle.
- The reader intercepts `<a href>` clicks, so **never use `href="#..."` for in-page navigation**. Tabs and section links use `data-go="<id>"` and the `go()` function; `data-go=""` is the Simulator tab, `data-go="handover"` the handover tab, `data-go="h-a"` etc. jump to a handover section.

## Page structure

Header: page title, tabs (Simulator / Dev handover), traffic inputs, preset buttons, arrivals/speed/pause/reset, sim clock. On the handover tab everything except title and tabs is hidden (`body.handover`).

Simulator tab (`<main>`):
1. **Cost vs accuracy** table: per estimator RTT/s, cmds/s, work/s, memory, ticker, MAE, max err. Green = best, red = worst per column.
2. **Error ratio (level)** chart: estimators with signal `ratio`, dashed configured truth, grey dots = observed failures/total per 0.5 s bin.
3. **Rate (events/s)** chart: estimators with signal `fail` / `ok` / `all`, dashed truth per signal.
4. **Redis commands/s (all streams)** chart.
5. **Error vs truth, % (rolling 10 s)** chart: all estimators on one normalised scale.
6. **What to try** guide.
Right column: estimator cards (params, live hint, Redis commands and live cost) and an Add dropdown.

Dev handover tab (`<article id="handover">`): recommendation, comparison table, rules for every option, each algorithm A–F with key layout, Lua scripts, Go (go-redis v9) code, choosing τ, scaling, tests to write.

## Core JS concepts

### Adapter contract (`ADAPTERS` array)

Each algorithm is one object; to add one, append to `ADAPTERS`:

```js
{
  id, label, tag,            // tag: 'per-event' | 'ticker' | 'baseline' (see TAGS)
  formula, desc,             // shown on the card
  params: { key: { label, min, max, step, v, help } | { label, options, v, help } },
  hint(p, rps) -> html,      // live derived numbers (half-life, equivalent τ, ...)
  redis(p, x) -> { how, ticker, mem, rtt, cmds, work },  // per stream; x = { ok, fail, reads } per second
  create(p) -> { observe(t, ok), read(t) },              // t = sim seconds; read returns NaN if undefined
}
```

- `p` is live: slider changes mutate it and apply on the next observe/read. Changing `signal` clears that card's history.
- `signal` param (`SIGNAL`): `ratio` → level chart; `fail` / `ok` / `all` → rate chart. `kindOf(p)` decides which.
- `pick(sig, f, a)` turns failure/total values into the requested signal.
- `cost(x, okEv, failEv, read, extra)` builds the per-second Redis cost from per-event `[rtt, cmds, work]` tuples; `extra` is for ticker work (already per second).

Current adapters and defaults:

| id | label | default signal | notes |
|---|---|---|---|
| `event-tau` | Per-event time-decayed | ratio (only) | α = 1 − e^(−Δt/τ); level freezes with no traffic |
| `event-count` | Per-event fixed α | ratio (only) | ignores time; memory ≈ 1/α events |
| `window` | Sliding window | ratio | `impl`: `zset` (exact, sim uses 10 ms buckets) or `buckets` (`bucket` param); bucket trimmed at its midpoint |
| `tick` | Fixed-tick | fail | needs ticker; ratio is 0/0 with no traffic |
| `decay-rate` | Decayed counts (recommended) | ratio | f and a decayed counters; ratio freezes, rates decay |

Default cards on load: event-tau, event-count, window (zset), window (buckets), tick, decay-rate.

### Simulation (`S`, `step`, `frame`)

- Arrivals per stream: Poisson (exponential gaps) or uniform; `resched()` redraws next arrivals whenever rates change.
- `step(dt)` generates events, feeds every estimator, bins observed events (`BIN` = 0.5 s), samples each estimator once per frame into `hist`, `cost`, `err`, and trims to `WINDOW` = 60 s.
- Rolling error (`ERRW` = 10 s): Σ|estimate − truth| ÷ Σ|truth|, kept incrementally (`eq`, `se`, `st`).
- Table MAE / max err (`errStats`): over the visible 60 s, normalised by truth so ratio and rate rows compare; raw values are in the cell popup. Startup samples (~100% error) dominate the first 60 s.
- Truth = the configured slider values, not observed counts.
- `S.script`: timed traffic changes on the sim clock (used by the Spike preset); cleared on Reset.

### Presets (`PRESETS`)

Each preset resets the sim (unless `reset: false`) and sets ok, fail, reads-per-request, reads/s, streams, optional `script`:
- 500 rps · 10% errors · read every request (450 / 50)
- 1000 rps · 1% errors · read every request (990 / 10)
- Spike: 500 rps, 1% → 50% at +10 s → 1% at +20 s
- Low traffic: 5 rps · 20% errors (4 / 1), 1 read/s
- Traffic stops (0 / 0, no reset)
- Fleet: 10k streams × 100 rps · read every request

### Tooltips / popups

- Any element with `data-tip="..."` gets a styled popup (`#pop`); content is HTML, newlines render (`white-space: pre-line`). `class=tip` adds the dotted underline.
- Table header definitions are the source for `GLOSS` (read from `th .tip`), reused by table cells and card cost lines. Change a definition in the `<th>` and it propagates.
- Table cells show a per-row breakdown (per-stream value × streams, memory formula for zset, RTT formula).

## Redis cost model

Per stream, then × streams in the table. Commands match the handover scripts exactly; keep them in sync when either changes.

| Algorithm | per event (rtt, cmds) | per read | ticker | memory/stream |
|---|---|---|---|---|
| Per-event time-decayed | 1, 3 (HMGET, HSET, EXPIRE) | 1, 1 (HGET) | no | **120 B (measured)** |
| Per-event fixed α | 1, 2 (GET, SET EX) | 1, 1 | no | 60 B (estimate) |
| Fixed-tick | 1, 1 (HINCRBY) | 1, 1 (HMGET) | 1 RTT, 5 cmds per tick | 150 B (estimate) |
| Decayed counts | 1, 3 (HMGET, HSET, EXPIRE) | 1, 1 (EVALSHA→HMGET) | no | **144 B (measured)** |
| Sliding window zset | 1, 3 ok / 6 fail | 1, 2 (ZCOUNT ×2) | no | **120 B (measured) × (events/s + fail/s) × W** |
| Sliding window buckets | 1, 2 ok / 4 fail (INCR, EXPIRE) | 1, 1 (MGET of 2·⌈W/bucket⌉ keys) | no | 70 B (estimate) × 2·⌈W/bucket⌉ |

- `work` = hash fields / keys touched, sorted-set ops costed at log₂(N). A CPU proxy, not a benchmark.
- Measured sizes come from `MEMORY USAGE` on Redis 7 (zset of 100k members with 36-char ids = 12.0 MB). They exclude the expires-dict entry and allocator fragmentation (add ~10–30%).
- "reads = traffic" sets reads/s to the stream's request rate (breaker checks on every call).

## Recommendation (what the handover says)

**A. Decayed counts**: one hash `err:{stream}` with `f`, `a`, `ts`; per request (or per batch) `f = f·k + failures/τ`, `a = a·k + requests/τ`, `k = e^(−Δt/τ)`, using Redis `TIME`. Ratio = f/a (freezes during silence), rates = decayed f and a. Breaker trips only when `rps ≥ minRPS && ratio ≥ threshold`. Time to cross threshold θ after a jump p₀ → p₁: `t = τ · ln((p₁ − p₀)/(p₁ − θ))`.

Rules in the handover that apply to all options: Redis clock via `TIME` (Redis ≥ 5), hash-tag the stream id (`err:{merchant_42}`), return floats as strings from Lua, load scripts with go-redis `redis.NewScript` (EVALSHA + EVAL fallback), TTL every key (5τ for decayed state), pre-aggregate per host for hot streams.

## Verifying changes

1. **Self-test** (adapters converge to truth within 10% on 90 ok + 10 fail/s, every signal and window impl; Redis cost is positive). Runs on page load via `console.assert`; run it in Node by extracting the script (keep exactly one `<script>` block, and keep DOM code inside `if (typeof document !== 'undefined')`):
   ```bash
   sed -n '/<script>/,/<\/script>/p' index.html | sed '1d;$d' > /tmp/ewma.js && node /tmp/ewma.js
   ```
   No output = pass.
2. **Render check** with headless Chrome (rAF barely advances in headless, so the clock stays near 0.1 s; inject a `setTimeout(() => el.click(), 300)` script into a copy of the page to test clicks):
   ```bash
   "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless=new --disable-gpu \
     --window-size=1400,1800 --virtual-time-budget=3000 --screenshot=/tmp/shot.png "file://$PWD/index.html"
   ```
   `--dump-dom` is useful to assert state (e.g. `<body class="handover">`).
3. **Lua scripts**: test against a throwaway Redis, then stop it:
   ```bash
   docker run -d --rm --name ewma-redis -p 56379:6379 redis:7-alpine
   docker exec ewma-redis redis-cli EVAL "$(cat script.lua)" 1 'err:{m1}' 1 1 5 600
   docker stop ewma-redis
   ```
   The Lua source lives only inside `index.html` (HTML-escaped in `<pre><code>`); copy it out from the handover tab's Copy buttons. Verified so far: decayed counts write/read (per-event and batched, ratio frozen after silence), level EWMA, fixed-tick tick script, zset write/read. Bucket commands and fixed-α script are reviewed, not run.
4. Push the published copy (see `CLAUDE.local.md`).

## Conventions

- Keep it one file with no external dependencies.
- Colors only via CSS variables (`--bg --panel --fg --muted --line --grid --accent --good --bad`), defined for light, dark media query, and both slashdocs theme attributes. Canvas reads them with `getComputedStyle`.
- Card/series colors come from `PAL` in add order.
- Prose on the handover page: plain, specific, no em dashes.
- When the cost model or a script changes, update all three: the adapter's `redis()`, the handover table/section, and this file.
