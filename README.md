# Multi-Model TCP Messaging Server

A framed TCP messaging system implemented three different ways — thread-per-connection,
`select()`-based multiplexing, and `epoll()`-based multiplexing — sharing one wire
protocol and one client, and benchmarked against each other under increasing
connection load.

This started as a socket-programming lab assignment (a Base64-encoding chat
client/server). It grew into a small systems project answering a concrete
engineering question: **how does a server's I/O and concurrency model affect
its throughput, latency, and failure rate as concurrent load grows — and
where does each model start to break down?**

This is the same class of problem behind real infrastructure decisions — for
example, Netflix rewrote its Zuul API gateway from a blocking,
thread-per-connection model to Netty's async epoll-based event loop after
thread exhaustion caused cascading failures at scale, reporting an 80%
memory reduction and 5x throughput increase per instance. This project
reproduces that comparison at a small, inspectable scale, with real
measurements rather than a general claim.

---

## Table of contents

- [Architecture](#architecture)
- [Wire protocol](#wire-protocol)
- [Project structure](#project-structure)
- [Build & run](#build--run)
- [Benchmark results](#benchmark-results)
- [Bugs found and fixed](#bugs-found-and-fixed)
- [Design notes / tradeoffs](#design-notes--tradeoffs)

---

## Architecture

```
                        ┌────────────────────┐
                        │      Client         │
                        │  (Client.cpp)        │
                        └─────────┬────────────┘
                                  │ same TCP wire protocol
                                  │ (protocol/framing.h)
              ┌───────────────────┼───────────────────┐
              │                   │                   │
     ┌────────▼────────┐ ┌────────▼────────┐ ┌────────▼────────┐
     │  ServerThread    │ │  ServerSelect    │ │  ServerEpoll     │
     │  1 thread/client │ │  select() loop   │ │  epoll() loop    │
     │  blocking I/O    │ │  non-blocking    │ │  non-blocking,   │
     │                  │ │  O(n) per call   │ │  edge-triggered  │
     └──────────────────┘ └──────────────────┘ └──────────────────┘

     All three implement the identical message-handling logic
     (protocol/common.h: processClientMessage) — only the I/O and
     concurrency strategy around it differs.

                        ┌────────────────────┐
                        │     LoadTest         │  spawns N concurrent
                        │  (loadtest/LoadTest)  │  connections, measures
                        └────────────────────┘  throughput + latency
```

The client is intentionally unaware of which server model it's talking to —
that's the point. The concurrency model is a server-side implementation
detail hidden entirely behind the shared protocol.

---

## Wire protocol

Every message on the wire is a single length-prefixed frame:

```
[ 4 bytes: length N, big-endian (network byte order) ][ N bytes: base64 payload ]
```

The decoded payload (before base64) is:

```
<message type><separator "$$$###$$$"><content>
```

**Message types:**

| Type | Meaning | Direction |
|---|---|---|
| 1 | Text message | client → server |
| 2 | ACK | server → client |
| 3 | Termination ("BYE") | client → server |
| 4 | Server busy (backpressure) | server → client |
| -1 | Malformed frame (protocol error) | internal signal, not sent on wire |
| -2 | Connection closed / socket error | internal signal, not sent on wire |

The length prefix exists specifically because raw TCP does not preserve
message boundaries — see [Bugs found and fixed](#bugs-found-and-fixed) for
why that matters and what broke without it.

---

## Project structure

```
protocol/
  essentials.h    base64 codec + misc helpers (self-contained, no hidden include deps)
  framing.h       length-prefixed blocking send/recv (sendMessage / recvMessage)
  common.h        non-blocking incremental frame parser (feedClient) +
                   shared message-handling logic (processClientMessage) +
                   safePrint (mutex-guarded logging)

server_thread/
  ServerThread.cpp   thread-per-connection server, blocking I/O

server_select/
  ServerSelect.cpp   single-threaded select() event loop

server_epoll/
  ServerEpoll.cpp    single-threaded epoll() event loop, edge-triggered

client/
  Client.cpp         interactive client; works unmodified against all three servers

loadtest/
  LoadTest.cpp       concurrent load generator; measures throughput + p50/p95/p99 latency,
                     appends results to benchmarks/results.csv

benchmarks/
  results.csv          raw benchmark data
  plot.py              generates benchmark_results.png from results.csv
  benchmark_results.png

Makefile             builds all five binaries into bin/
run_benchmarks.sh    automated sweep: runs LoadTest against all three servers
                     across a range of connection counts
```

---

## Build & run

Requires a Linux environment (epoll is Linux-specific; the other two
binaries are portable POSIX) and a C++17 compiler.

```bash
make all
```

This builds `bin/ServerThread`, `bin/ServerSelect`, `bin/ServerEpoll`,
`bin/Client`, `bin/LoadTest`.

**Run any server** (pick one — they all speak the same protocol):

```bash
./bin/ServerThread 9000
# or: ./bin/ServerSelect 9000
# or: ./bin/ServerEpoll 9000
```

**Connect with the interactive client:**

```bash
./bin/Client 127.0.0.1 9000
```

**Run the load generator against it:**

```bash
./bin/LoadTest 127.0.0.1 9000 100 20 my-label
#                ip        port  N   M   label
#   N = concurrent connections, M = messages per connection
```

**Run the full automated benchmark sweep** (starts each server in turn,
runs LoadTest at several connection counts, writes `benchmarks/results.csv`):

```bash
./run_benchmarks.sh 9000 15
cd benchmarks && python3 plot.py   # -> benchmark_results.png
```

Stop any server with `Ctrl+C` — all three handle `SIGINT` and shut down
cleanly (stop accepting new connections, close existing sockets, exit).

---

## Benchmark results

Measured locally: 10 / 50 / 100 / 250 / 500 concurrent connections, 10
messages each, over loopback.

| Connections | ServerThread (msgs/sec) | ServerSelect (msgs/sec) | ServerEpoll (msgs/sec) | ServerThread failures |
|---|---|---|---|---|
| 10  | 126  | 126  | 126  | 0 |
| 50  | 628  | 619  | 629  | 0 |
| 100 | 1242 | 1241 | 1248 | 0 |
| 250 | 2380 | 3054 | 3079 | **490** |
| 500 | 1261 | 2711 | 6015 | **2690** |

![benchmark results](benchmarks/benchmark_results.png)

**What this shows:**

- At low concurrency, all three are statistically indistinguishable — with
  few connections there's no meaningful contention for any model to handle
  differently.
- **`ServerThread` throughput peaks around 250 connections and then
  collapses** at 500, with real, outright failed requests (not just
  slower responses). This matches the expected failure mode of
  thread-per-connection under load: each connection costs a full OS thread
  (stack allocation, scheduler bookkeeping, context-switch overhead), and
  the model degrades non-gracefully once thread count is high enough to
  strain the scheduler.
- **`ServerSelect` and `ServerEpoll` both sustain zero failures** across
  the whole sweep — a single event-loop thread has no per-connection OS
  thread cost to pay.
- **`ServerEpoll` pulls decisively ahead of `ServerSelect` at 500
  connections** (6015 vs 2711 msgs/sec), consistent with `select()`'s
  O(total connections) cost per call (the kernel rescans the entire
  connection set every time) versus `epoll`'s cost being proportional only
  to the connections that actually became ready.

**Caveat, stated honestly:** these numbers are a single local run on
loopback with small text messages and no real network latency, packet loss,
or competing load. The specific crossover point (~250-500 connections) is
illustrative, not universal — it will shift with hardware, message size,
and how idle vs. active the connections are. Properly characterizing *when*
each model wins, rather than claiming one model universally wins, is the
actual point of running this benchmark.

---

## Bugs found and fixed

Documented because finding and diagnosing these was as valuable as the
benchmark results themselves.

### 1. Message framing (TCP is a byte stream, not a message stream)

The original code did one `read()`/`write()` per message and assumed it
would return exactly one logical message. TCP gives no such guarantee — the
kernel is free to split a message across multiple reads or coalesce
multiple sends into one read. This silently breaks under larger payloads or
bursty traffic. Fixed with the length-prefixed frame format in
`protocol/framing.h`, plus a receive loop (`readN`) that keeps reading until
it has accumulated exactly the expected number of bytes.

### 2. `SIGPIPE` killing the entire process under load

Load-testing at 100+ concurrent connections caused both the client and
server processes to die abruptly — exit code 141, with no error message and
truncated log output. **Root cause:** writing to a TCP socket after the
peer has already reset the connection raises `SIGPIPE`, and Linux's default
disposition for `SIGPIPE` is to **terminate the entire process**, not just
fail that one `write()` call. This is easy to trigger under realistic load
(many connections opening and closing in quick succession) and easy to miss
in low-concurrency manual testing, since it only shows up once connections
are actually being torn down mid-write. Fixed by calling
`signal(SIGPIPE, SIG_IGN)` at startup in every binary, so a broken pipe now
correctly surfaces as an ordinary `errno == EPIPE` failure return — which
the existing error-handling paths already treat as "connection closed" —
instead of an unrecoverable crash.

### 3. Undersized TCP backlog

`listen()`'s backlog was set to 3, later 16, in the original code — fine
for a manual demo, not for a burst of 100+ simultaneous connection
attempts, which caused spurious `ECONNREFUSED`s unrelated to the
concurrency model actually being benchmarked. Raised to 128 so the
benchmark measures the concurrency model's real behavior, not an
artificially small accept queue.

### 4. Unsynchronized global connection counter

`globalUserCnt` was a plain `int`, incremented from whatever thread called
`manageNewConnection()`. Currently safe only because connection acceptance
happens on a single thread — but a landmine for any future change that
accepts on multiple threads (two threads reading-modifying-writing the same
`int` concurrently is a textbook data race). Fixed with `std::atomic<int>`.

---

## Design notes / tradeoffs

- **Why length-prefixing instead of a delimiter?** A delimiter (e.g. `\n`)
  requires scanning byte-by-byte for the delimiter and doesn't work cleanly
  if the payload itself could contain that byte. A length prefix lets the
  receiver know exactly how many bytes to read with no scanning and no
  escaping concerns — at the cost of the sender needing to know its message
  length up front, which is trivial here since messages are fully buffered
  in memory before sending.
- **Why edge-triggered epoll instead of level-triggered?** Edge-triggered
  avoids the kernel re-checking "is this still readable?" on every
  `epoll_wait()` call for a socket that hasn't been fully drained yet,
  reducing syscall overhead at high connection counts — at the cost of the
  application needing strict "read until EAGAIN" discipline, since epoll
  will not remind you about data you didn't finish reading. Both the accept
  loop and the message-read loop in `ServerEpoll.cpp` follow this
  discipline explicitly.
- **Why is `select()` included at all if `epoll` is strictly better on
  Linux?** To make the comparison concrete rather than assumed — the
  benchmark shows the two performing near-identically at low-to-moderate
  connection counts and only diverging at higher concurrency, which is a
  more honest and more interesting result than "epoll wins, obviously."
- **Why a 1 MB frame size cap?** Without a ceiling, a malicious or buggy
  peer could send a length header claiming an arbitrarily large payload and
  force the receiver to allocate huge buffers. Never trust a length field
  read off the network without a sanity bound.
