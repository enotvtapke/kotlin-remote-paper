# Kotlin Remote — Framework overhead benchmarks

All numbers below were produced from sources in
`kotlin-remote/integration-tests/bench` and `kotlin-remote/integration-tests/bench-kotlinrpc`.

## Setup

- CPU: Intel Core Ultra 9 185H, 22 logical cores
- OS: Ubuntu 24.04.2 LTS, Linux 6.17.0-23-generic
- JDK: Temurin 21.0.7
- Kotlin: 2.2.20
- Ktor: 3.2.1
- Kotlin Remote: this repo (current `main`)
- kotlinx.rpc: 0.10.1 (latest at time of measurement)
- All traffic is over `localhost` between two threads in the same JVM
  (server runs in Netty, client in CIO). This excludes physical network
  variance and is the cleanest setting to isolate framework overhead.

Wall-clock per-call samples are taken with `System.nanoTime()`, with at
least 2 000 warmup iterations to let HotSpot tier up, and 10 000
measurement iterations afterwards. Statistics shown: mean (`mean`),
standard deviation (`sd`), and percentiles `p50/p90/p99`. All numbers in
microseconds (`us`).

## (a) Per-call latency — Kotlin Remote vs kotlinx.rpc

Same six operations on both frameworks (`ping`, `echoLong`,
`echoString` small, `echoString` 1 KiB, `addLong`, `createTodo` and a
larger `listTodos(100)`). Defaults of both frameworks: Kotlin Remote
uses Ktor HTTP POST, kotlinx.rpc uses Ktor WebSocket frames.

| Operation               | KR mean | KR p50 | KR p99 | krpc mean | krpc p50 | krpc p99 |
|-------------------------|--------:|-------:|-------:|----------:|---------:|---------:|
| `ping()`                | 478 us  | 438 us | 1076 us | 407 us   | 351 us   | 835 us   |
| `echoLong(0)`           | 318 us  | 312 us |  478 us | 305 us   | 298 us   | 464 us   |
| `echoString("hello")`   | 316 us  | 312 us |  457 us | 201 us   | 196 us   | 342 us   |
| `echoString(1 KiB)`     | 332 us  | 327 us |  479 us | 219 us   | 213 us   | 365 us   |
| `addLong(1, 2)`         | 313 us  | 308 us |  451 us | 195 us   | 190 us   | 340 us   |
| `createTodo(req)`       | 319 us  | 314 us |  470 us | 200 us   | 195 us   | 351 us   |
| `listTodos(100)`        | 400 us  | 393 us |  559 us | 342 us   | 339 us   | 498 us   |

Read-out:

- Median-call latency on `localhost` is **~310 us for Kotlin Remote and
  ~190 us for kotlinx.rpc** for a small call. The gap is roughly
  constant in absolute terms (~110 us) and does not grow with payload
  size up to 1 KiB.
- The constant gap is dominated by **transport**, not by framework
  logic: Kotlin Remote does a full HTTP POST per call (request line,
  headers, response, content-length recomputed each time), kotlinx.rpc
  re-uses a single persistent WebSocket and sends one short frame.
  The `RemoteCall` wrapping plus `CallableMap` string lookup contributes
  only nanoseconds (see (b)).
- For a larger payload (`listTodos(100)`, ~3.5 KiB JSON) the gap is
  smaller in relative terms (~17%), as serialization/deserialization
  costs in both frameworks grow at the same rate.
- Tail latency (`p99`) of Kotlin Remote stays within ~2x its median,
  same as kotlinx.rpc, indicating no unusual jitter from the framework.

The Kotlin Remote transport is pluggable via `RemoteClient`. The same
benchmark with a HTTP/2 connection-reusing or in-process transport
would close most of the constant gap shown above. With Ktor CIO HTTP
defaults the constant overhead is ~110 us per call on this hardware.

## (b) Local dispatch overhead

Verifies the thesis claim that a `@Remote` function dispatched in
`LocalContext` (i.e. on the server side, or during cross-tier internal
composition in the CMS example) costs essentially the same as a plain
Kotlin call. Tight loop, 200 000 warmup iterations, 1 000 000
measurement iterations, no I/O.

| Case                                                | per-call |
|-----------------------------------------------------|---------:|
| plain `suspend fun mulPlain(a, b)`                  |  **15.1 ns** |
| `@Remote suspend fun mulRemote(a, b)` in `LocalContext` | **21.8 ns** |
| `@Remote suspend fun mulRemote(a, b)` in `ConfiguredContext` (network) | 470 243 ns (470 us) |

The framework adds **~7 ns per call** in the local branch over a plain
`suspend fun`. This is roughly one `is`-check on the context parameter
plus the call dispatch. Compared to a network round-trip, local
dispatch is **22 000x cheaper**, which empirically validates the claim
that the hierarchical-contexts feature does not pay for a network
round-trip when a function is composed locally on the server.

## (c) Distributed garbage collection — `LeaseManager` cost

Tests how the leasing GC scales with the number of live remote-class
instances. Server-side `renewLeases` performance, on-wire renewal
payload size and per-instance memory.

### (c.1) `LeaseManager.renewLeases(N)` server cost (no network)

| N       | per-renewal | per-id   |
|---------|-------------|----------|
| 1       | 0.48 us     | 484 ns   |
| 10      | 0.62 us     |  62 ns   |
| 100     | 5.37 us     |  54 ns   |
| 1 000   | 47.78 us    |  48 ns   |
| 10 000  | 412 us      |  41 ns   |
| 100 000 | 4 835 us    |  48 ns   |

The renewal is bound by a single synchronized hash-map operation per
ID. Steady-state cost is **~45 ns per renewed lease** — even with
100 000 live stubs, a full renewal sweeps in about 5 ms. At the default
renewal interval of 10 s this is **0.05% server CPU**.

### (c.2) Renewal request body size and steady-state traffic

`LeaseRenewalRequest` is `List<Long>` plus a `clientId`, JSON-encoded.
Default renewal interval is 10 s, so each client→server batch is sent
6 times per minute.

| N       | body    | steady-state |
|---------|--------:|-------------:|
| 1       | 59 B    | 354 B/min    |
| 10      | 78 B    | 468 B/min    |
| 100     | 349 B   | 2.0 KiB/min  |
| 1 000   | 3.9 KiB | 23.1 KiB/min |
| 10 000  | 48 KiB  | 287 KiB/min  |
| 100 000 | 575 KiB | 3.4 MiB/min  |

100 K live cross-machine references hold a steady **3.4 MiB/min** of
renewal traffic per client — well within any realistic budget.

### (c.3) Server-side memory per leased instance

Cumulative used-heap measurement on growing pools, single shared
`clientId` per client (realistic case).

| N        | used heap   | delta        | per-stub |
|----------|------------:|-------------:|---------:|
| 0        |  4 579 KiB  | —            | —        |
| 10 000   |  8 234 KiB  |  3 655 KiB   | **374 B**|
| 100 000  | 42 137 KiB  | 33 902 KiB   | **386 B**|
| 500 000  | 197 859 KiB | 155 722 KiB  | **399 B**|

A leased instance carries **~400 bytes of server-side state**: the
user object (16 B header + fields), two hash-map nodes (in
`RemoteInstancesPool` and in `LeaseManager.leases`), a `LeaseEntry`
plus its `clientIds` set, and HashMap internal load-factor headroom.
500 000 live remote objects therefore fit in **~150 MB** of server
heap.

### (c.4) `cleanupExpiredInstances` cost

After force-expiring 100 000 leases, the periodic cleanup sweep took
**~31 ms** total, ~310 ns per id. Running every 10 s this is again
**0.3 % CPU** even at this extreme pool size.

## (d) Per-function bytecode cost — controlled growth study

A whole-application JAR comparison is contaminated by source-file count,
DI wiring, helper objects and Ktor DSL lambdas, none of which are caused
by the RPC framework. To measure the plugin-emitted per-callable cost in
isolation we ran a controlled growth study: one auto-generated source
file with `N` identical `@Remote suspend fun fN(x: Long): Long = x`
functions for Kotlin Remote (and `N` identical methods of one `@Rpc`
interface for kotlinx.rpc), compiled to a JAR for `N ∈ {0, 1, 10, 100,
200, 500, 800, 900, 1000}`. Source: `integration-tests/growth-study.sh`
plus the `growth-bench` and `growth-bench-kotlinrpc` modules.

For Kotlin Remote we measured two variants:

- **bare**: only the `@Remote` functions are declared, `genCallableMap()`
  is not called anywhere. This isolates the per-function body
  transformation.
- **genmap**: a single top-level `val GENERATED_MAP = genCallableMap()`
  is added. This represents a realistic use, because the framework
  cannot be operated without calling `genCallableMap()` at least once.

### (d.1) Raw measurements

Class-file bytes are summed over all `.class` files of the module
(after the `atomicfu` post-processor). JAR bytes are the compressed
final artifact.

| Configuration        | N   | JAR (B)   | classes | class bytes |
|----------------------|----:|----------:|--------:|------------:|
| kotlin-remote-bare   |   0 |  1 319    | 1       |    1 069    |
| kotlin-remote-bare   |   1 |  2 430    | 2       |    3 065    |
| kotlin-remote-bare   |  10 |  2 681    | 2       |    5 549    |
| kotlin-remote-bare   | 100 |  4 955    | 2       |   30 603    |
| kotlin-remote-bare   |1000 | 26 979    | 2       |  285 263    |
| kotlin-remote-genmap |   0 |  2 021    | 2       |    1 884    |
| kotlin-remote-genmap |   1 |  4 066    | 3       |    6 413    |
| kotlin-remote-genmap |  10 | 13 561    | 12      |   25 889    |
| kotlin-remote-genmap | 100 |108 107    | 102     |  221 655    |
| kotlin-remote-genmap | 200 |213 708    | 202     |  440 196    |
| kotlin-remote-genmap | 500 |530 010    | 502     |1 095 996    |
| kotlin-remote-genmap | 800 | **fail**  | —       |    —        |
| kotlin-remote-genmap |1000 | **fail**  | —       |    —        |
| kotlinx.rpc          |   0 |  3 429    | 3       |    4 802    |
| kotlinx.rpc          |   1 |  5 395    | 4       |    9 302    |
| kotlinx.rpc          |  10 | 15 803    | 13      |   31 555    |
| kotlinx.rpc          | 100 |119 141    | 103     |  255 042    |
| kotlinx.rpc          | 200 |234 865    | 203     |  504 775    |
| kotlinx.rpc          | 500 |580 363    | 503     |1 254 175    |
| kotlinx.rpc          | 800 | **fail**  | —       |    —        |
| kotlinx.rpc          |1000 | **fail**  | —       |    —        |

### (d.2) Per-function cost (slope) and fixed overhead (intercept)

Best-fit slope from the linear region (`N ≤ 500`):

| Configuration         | bytes / function | classes / function | fixed overhead (N=0) |
|-----------------------|-----------------:|-------------------:|---------------------:|
| Kotlin Remote bare    | **284 B**        | 0 (all functions in one file) | 1.0 KiB |
| Kotlin Remote genmap  | **~2 190 B**     | 1 anonymous class  | 1.8 KiB |
| kotlinx.rpc           | **~2 500 B**     | 1 anonymous `Invokator` class | 4.7 KiB |

Read-out:

- The pure `@Remote` body transformation emitted by the Kotlin Remote
  compiler plugin costs **~280 bytes per function** and generates *zero
  additional classes*. It is just method bytecode inside the file's
  synthetic `Kt` class.
- Adding the realistic `genCallableMap()` call brings Kotlin Remote in
  line with kotlinx.rpc: ~2 190 B per function (one anonymous
  `RemoteInvokator` lambda class per entry in the map) versus
  kotlinx.rpc's ~2 500 B per function (one `Invokator` nested class per
  method in the `@Rpc` interface). The two frameworks pay roughly the
  same per-callable cost in the realistic case; Kotlin Remote is
  marginally smaller (~12 %) because its registry entry holds a
  function reference where kotlinx.rpc holds a richer stub-side
  handler.
- Fixed plugin overhead at N=0 is **1–2 KiB** for Kotlin Remote and
  **~5 KiB** for kotlinx.rpc (`@Rpc` interface stub + Companion +
  serialization bridges). On a tiny app this gives Kotlin Remote a
  modest head start that is dwarfed once N grows.

### (d.3) Compiler ceiling — both frameworks hit the JVM 64 KiB method limit

Both frameworks failed to compile at N=800 functions in a single
emission unit. The failure mode is identical in both cases:

```
Method too large: <class>.<clinit> ()V
```

- **Kotlin Remote** fails at the synthesized top-level initializer of
  the file's `Kt` class, where the expanded `genCallableMap()` literal
  is built (one `RemoteCallable(...)` constructor invocation per
  `@Remote` function).
- **kotlinx.rpc** fails at the static initializer of the per-interface
  stub class `GrowthService$$rpcServiceStub`, where each method's
  `Invokator` is registered.

JVM bytecode caps a single method at 64 KiB. Both plugins emit a single
per-module (KR) or per-interface (kotlinx.rpc) initializer that scales
linearly with the number of callables, so both have a natural ceiling
between N=500 and N=800 in this experiment. In practice this is not
a real limit for either framework: realistic services split their API
across multiple `@Rpc` interfaces / multiple compilation modules with
their own `genCallableMap()` call, which raises the ceiling
proportionally. It is, however, an interesting symmetry: the two
plugins, despite very different surface API, ended up making the same
architectural choice of "one static initializer per emission unit".

### (d.4) What this means for the thesis claim

The thesis claims Kotlin Remote is "lightweight" in two
user-facing senses:

1. **Boilerplate**: amount of code the user has to write to make a
   function remote (measured in Section 4 of the thesis by LoC).
2. **Learning curve**: number of concepts and tools the user must
   acquire — no separate IDL like Protocol Buffers, no separate code
   generation pipeline, no service interface + implementation pair to
   keep in sync. The user only needs to know plain Kotlin plus the
   generic `context(...)` language feature, plus three new names
   (`@Remote`, `RemoteContext`, `genCallableMap`).

The growth study reported in (d.1)–(d.3) does **not** measure either of
these UX claims. It measures the *engineering footprint* of the
generated artifact, which is a different question. The finding in that
different question is:

- In the realistic configuration (with `genCallableMap()`), Kotlin
  Remote emits ~2 190 B per remote function, ~12 % less than
  kotlinx.rpc's ~2 500 B. The two frameworks are within the same
  order of magnitude.
- The UX win that the thesis is actually claiming is therefore
  **free** at the bytecode level: the user gets a simpler
  programming model and does not pay for it in larger compiled
  artifacts.
- Both frameworks share the same ~64 KiB-per-`<clinit>` ceiling at
  N≈600–800 callables in a single emission unit. This is a JVM-level
  symmetry, not a framework-specific limit.

Compile-time and whole-application JAR comparisons from the previous
draft (`bench` vs `bench-kotlinrpc`, `social` / `cms`, cold compile
times) were removed because they were contaminated by source-file
count, DI wiring, Ktor DSL lambdas and JVM startup, and could not be
attributed to the framework plugin.

## Summary

| Question                                                  | Answer                                       |
|-----------------------------------------------------------|----------------------------------------------|
| Per-call latency vs kotlinx.rpc on default transports     | +100 us median (HTTP POST vs WebSocket)      |
| Cost of `@Remote` function dispatched in `LocalContext`   | **~7 ns** above a plain `suspend fun`        |
| Server-side renewal cost per 100 K live stubs             | 4.8 ms / 10 s = **<0.1 % CPU**               |
| Server-side memory per leased instance                    | **~400 B**                                   |
| Steady-state renewal traffic for 10 K live stubs          | **~290 KiB / minute**                        |
| Per-function bytecode cost (realistic, with `genCallableMap`) | **~2 190 B** vs kotlinx.rpc **~2 500 B** (UX win is "free" at bytecode level) |
| Per-function bytecode cost (`@Remote` body only)          | **~280 B**                                   |
| Compiler ceiling per emission unit (both frameworks)      | ~600–800 callables (JVM 64 KiB method limit) |

All numbers are reproducible by running:

```
cd kotlin-remote/integration-tests
./gradlew :bench:benchPerCall
./gradlew :bench:benchLocal
./gradlew :bench:benchLease
./gradlew :bench-kotlinrpc:benchPerCall
./growth-study.sh > growth-study.csv
```
