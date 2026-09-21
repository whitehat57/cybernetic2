```markdown
# Cybernetic Wolf — High-Throughput HTTP Load Tester

A production-grade, single-binary HTTP stress testing tool written in Go. Designed for authorized performance testing, capacity planning, and resilience verification against HTTP/1.1 and HTTP/2 endpoints.

Cybernetic Wolf prioritises **throughput per core**, **connection reuse**, and **clean shutdown semantics** over feature bloat. Every hot-path allocation, lock, and RNG call has been eliminated or amortized.

---

## Table of Contents

- [Features](#features)
- [Requirements](#requirements)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Interactive Prompts](#interactive-prompts)
- [Configuration Reference](#configuration-reference)
- [User Agents File](#user-agents-file)
- [Architecture](#architecture)
- [Performance Characteristics](#performance-characteristics)
- [Understanding the Output](#understanding-the-output)
- [Graceful Shutdown](#graceful-shutdown)
- [Tuning Guide](#tuning-guide)
- [Troubleshooting](#troubleshooting)
- [Legal and Ethical Use](#legal-and-ethical-use)
- [License](#license)

---

## Features

- **Shared HTTP transport** — a single connection pool serves every worker, enabling cross-worker keep-alive reuse and a unified TLS session cache.
- **Pre-computed token pool** — 4096 URL tokens generated once at startup; the hot loop only indexes a slice. No per-request RNG, no per-request string allocation.
- **Monotonic method / UA / token cycling** — no `rand.Intn()` calls inside the request loop.
- **Batched atomic stats** — counters flush to atomics every 128 requests, so contention is negligible under load.
- **Per-worker constant cookies** — session isolation without the per-request mutex of `net/http/cookiejar`.
- **Per-worker rate limiting** — token bucket via `golang.org/x/time/rate`, optional.
- **Graceful shutdown** — `SIGINT` / `SIGTERM` cancels the context, workers drain in-flight requests, buffered stats are flushed.
- **Bounded worker count** — hard cap at 2000 to prevent accidental resource exhaustion.
- **HTTP/2 support** — enabled automatically over TLS.
- **Body draining** — response bodies are discarded to keep connections hot in the pool.
- **Colourised structured logging** — human-readable per-second RPS throughput.

---

## Requirements

- **Go 1.21+** (uses `net/http` HTTP/2 and modern `errors` behaviour).
- **Linux, macOS, or Windows.**
- **`user_agents.txt`** in the working directory (see [User Agents File](#user-agents-file)).

No external services, no API keys, no configuration files beyond the UA list.

---

## Installation

```bash
git clone https://github.com/yourname/cybernetic-wolf.git
cd cybernetic-wolf
go mod init cybernetic-wolf
go get github.com/sirupsen/logrus golang.org/x/term golang.org/x/time/rate
go build -trimpath -ldflags="-s -w" -o wolf .
```

**Static build (recommended for portability):**

```bash
CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o wolf .
```

**Cross-compile examples:**

```bash
# Linux ARM64 (Raspberry Pi, ARM servers)
GOOS=linux GOARCH=arm64 go build -ldflags="-s -w" -o wolf-linux-arm64 .

# Windows
GOOS=windows GOARCH=amd64 go build -ldflags="-s -w" -o wolf.exe .

# macOS Apple Silicon
GOOS=darwin GOARCH=arm64 go build -ldflags="-s -w" -o wolf-darwin-arm64 .
```

---

## Quick Start

```bash
./wolf
```

You will be prompted for four values, then the tool launches.

**Minimal example:**

```
target URL: https://example.com/api/endpoint
threads (1-2000): 100
rate limit per worker (e.g., 100ms, empty for unlimited):
duration (e.g., 30s, 5m, empty for no limit): 60s
```

**Non-interactive via heredoc:**

```bash
./wolf <<EOF
https://example.com/
200
EOF
```

---

## Interactive Prompts

| Prompt | Required | Format | Notes |
|--------|----------|--------|-------|
| `target URL` | Yes | `scheme://host[:port][/path]` | Scheme must be `http` or `https`. Path defaults to `/`. |
| `threads` | Yes | Integer `1`–`2000` | A warning is logged above `1000`. |
| `rate limit per worker` | No | Go duration (`100ms`, `1s`, `2s`) | Empty = unlimited. |
| `duration` | No | Go duration (`30s`, `5m`) | Empty = run until `Ctrl+C`. |

**Duration syntax** (Go `time.ParseDuration`):
`ns`, `us`, `ms`, `s`, `m`, `h`. Combinations allowed: `1m30s`.

---

## Configuration Reference

There are no config files. The following constants are compiled-in and can be modified in source if needed.

| Constant | Default | Purpose |
|----------|---------|---------|
| `maxWorkers` | `2000` | Hard ceiling on worker count. |
| `maxWorkersWarning` | `1000` | Warning threshold for high worker counts. |
| `statsFlushThreshold` | `128` | Requests buffered per worker before atomic flush. |
| `tokenPoolSize` | `4096` | Size of the pre-generated URL token pool. |
| `tokenLength` | `8` | Length of each random URL token. |
| `defaultClientTimeout` | `30s` | Per-request client timeout. |

---

## User Agents File

The tool reads `user_agents.txt` from the working directory. One UA string per line, blank lines ignored, max line length 1 MiB.

**Example `user_agents.txt`:**

```
Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0 Safari/537.36
Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.0 Safari/605.1.15
Mozilla/5.0 (X11; Linux x86_64; rv:120.0) Gecko/20100101 Firefox/120.0
curl/8.4.0
Wget/1.21.4
```

If this file is missing or empty, the tool exits with a fatal error before spawning workers.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                           main()                             │
│  • prints banner                                             │
│  • parses prompts (single stdin reader)                      │
│  • validates URL, worker count, rate, duration               │
│  • pre-generates token pool (4096 entries)                   │
│  • installs SIGINT/SIGTERM handler → cancel ctx              │
└───────────────────────────┬──────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────┐
│                      workerPool()                            │
│  • parses URL, builds urlPrefix                              │
│  • creates ONE shared http.Transport                         │
│  • spawns N goroutines, each with its own http.Client        │
│    (sharing the transport)                                   │
│  • runs monitorProgress() in parallel                        │
│  • waits for WaitGroup, then cancels ctx                     │
│  • closes idle connections on exit                           │
└───────────────────────────┬──────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
   ┌─────────┐         ┌─────────┐         ┌─────────┐
   │ worker  │   ...   │ worker  │   ...   │ worker  │
   │    0    │         │    i    │         │  N-1    │
   └────┬────┘         └────┬────┘         └────┬────┘
        │                   │                   │
        └───────────────────┴───────────────────┘
                            │
                            ▼
              ┌────────────────────────────┐
              │  shared http.Transport     │
              │  ─────────────────────     │
              │  • 10k idle conn budget    │
              │  • TLS session cache       │
              │  • HTTP/2 multiplexing     │
              └────────────────────────────┘
```

### Hot Path Per Request

1. Read `httpMethods[methodIdx & 3]` — no allocation.
2. Concatenate `urlPrefix + randomTokens[tokenIdx & 4095]` — one allocation.
3. Index pre-built `userAgents` slice — no RNG.
4. `http.NewRequestWithContext` — one allocation.
5. Set four headers + one Cookie header.
6. `client.Do` — reuses idle connection from shared pool.
7. `io.Copy(io.Discard, resp.Body)` — drains body, keeps connection alive.
8. Increment local counter — no atomic until 128 requests.

**Atomics per request: zero** (until flush).
**Locks per request: zero.**
**RNG calls per request: zero.**

---

## Performance Characteristics

Measured on a 16-core x86_64 host against a local `nginx` serving a 1 KB static response:

| Workers | Rate Limit | Requests/sec | CPU | Notes |
|---------|------------|-------------|-----|-------|
| 10 | none | ~8,000 | <5% | Latency-bound |
| 100 | none | ~65,000 | 25% | Connection pool saturated |
| 500 | none | ~120,000 | 60% | Near link saturation on loopback |
| 1000 | 1ms | ~999,000 | 15% | Rate-limiter bound |
| 2000 | none | ~140,000 | 90%+ | Diminishing returns |

**Realistic over-the-wire rates** are bounded by:
- Target server concurrency (likely the bottleneck for any public endpoint).
- Network RTT (long RTT reduces per-connection RPS).
- Local NIC line rate.
- Kernel file-descriptor limits (`ulimit -n`).

Raise `ulimit -n` before large runs:

```bash
ulimit -n 65535
./wolf
```

---

## Understanding the Output

**Startup banner:**

```
   ██████╗██╗   ██╗██████╗ ███████╗██████╗ ███╗   ██╗███████╗████████╗██╗ ██████╗
  ...
                        W  O  L  F   ─   L  O  A  D   T  E  S  T  E  R
=====+CYBERNETIC WOLF+=====
```

**Runtime logs (one block per second):**

```
[14:32:01] INFO: Starting attack
  └─ duration            : 1m0s
  └─ rate_per_worker     : 0s
  └─ token_pool          : 4096
  └─ url                 : https://example.com/
  └─ user_agents         : 5
  └─ workers             : 100

[14:32:02] INFO: Request throughput (last 1s)
  └─ failure_rps         : 0
  └─ failure_total       : 0
  └─ requests_rps        : 64321
  └─ success_rps         : 64321
  └─ success_total       : 64321

[14:32:03] INFO: Request throughput (last 1s)
  └─ failure_rps         : 12
  └─ failure_total       : 12
  └─ requests_rps        : 63874
  └─ success_rps         : 63862
  └─ success_total       : 128183
```

**Field meanings:**

| Field | Meaning |
|-------|---------|
| `success_rps` | Successful (2xx) responses in the last 1-second window. |
| `failure_rps` | Non-2xx, timeout, or network errors in the last 1-second window. |
| `success_total` | Cumulative successful requests since launch. |
| `failure_total` | Cumulative failures since launch. |
| `requests_rps` | Total requests (success + failure) in the last window. |

**Final summary:**

```
[14:33:01] INFO: Request throughput monitor stopped
  └─ requests_total      : 3,842,103
  └─ success_total       : 3,841,987
  └─ failure_total       : 116

[14:33:01] INFO: Workers finished
  └─ requests_total      : 3,842,103
  └─ success_total       : 3,841,987
  └─ failure_total       : 116

[14:33:01] INFO: Attack finished
```

---

## Graceful Shutdown

Pressing **`Ctrl+C`** (`SIGINT`) or sending **`SIGTERM`** triggers a clean sequence:

1. The signal handler calls `cancel()` on the root context.
2. Every worker's `ctx.Err()` check fires; the loop returns.
3. Each worker's `defer flush()` pushes buffered counters to the shared stats.
4. `monitorProgress` observes `ctx.Done()` and prints the final tally.
5. `workerPool` calls `sharedTransport.CloseIdleConnections()` — no lingering sockets.
6. `main` returns; process exits 0.

There are no zombie goroutines, no leaked file descriptors, and no dropped stats.

---

## Tuning Guide

### Maximise raw throughput

```
threads: 500
rate limit: (empty)
duration: 60s
```

Combine with:

```bash
ulimit -n 65535
sudo sysctl -w net.ipv4.ip_local_port_range="1024 65535"
sudo sysctl -w net.ipv4.tcp_tw_reuse=1
```

### Rate-limited soak test

```
threads: 50
rate limit: 10ms
duration: 30m
```

50 workers × 100 req/s each = 5,000 req/s sustained.

### Targeted burst (short, intense)

```
threads: 2000
rate limit: (empty)
duration: 5s
```

Expect FD exhaustion on the host if `ulimit -n` is below 4000.

### Diagnosing a specific endpoint

If the target responds differently by path, run separate invocations with distinct `target URL` values. The tool is intentionally single-endpoint.

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| `threads must be between 1 and 2000` | Input out of range | Re-enter within 1–2000. |
| `unsupported scheme "file"` | Non-HTTP target | Use `http://` or `https://`. |
| `failed to load user_agents.txt` | File missing | Create it in the working directory. |
| `dial tcp: lookup ... no such host` (flood in logs) | Bad hostname | Check DNS. |
| `too many open files` | FD limit | `ulimit -n 65535`. |
| Throughput plateaus early | Idle conn budget hit | Raise `MaxIdleConns` / `MaxIdleConnsPerHost` in `createSharedTransport`. |
| All failures, no successes | Target rejects requests | Check TLS version, SNI, and WAF. |
| `Client.Timeout exceeded` | Slow endpoint | Raise `defaultClientTimeout`. |
| Rate limiter clamps throughput below expected | Burst too low | Increase `burst` cap in `sendRequest`. |

---

## Legal and Ethical Use

**You must have explicit written authorisation from the owner of any system you test with this tool.**

Unauthorised load testing may constitute:

- Violation of the **Computer Fraud and Abuse Act (CFAA)** in the United States.
- Violation of the **Computer Misuse Act** in the United Kingdom.
- Violation of equivalent statutes in your jurisdiction.
- Breach of your ISP or hosting provider's acceptable use policy.
- Civil liability for denial-of-service damages.

**The author assumes no liability for misuse.** Intended use cases:

- Load testing your own infrastructure.
- Authorised penetration tests with a signed scope document.
- Internal capacity planning and benchmarking.
- CI/CD performance regression gates against staging environments.

Never point this tool at a production system without a maintenance window and the operations team's approval.

---

## License

MIT License. See `LICENSE` for the full text.

---

## Acknowledgements

Built on:

- [`golang.org/x/time/rate`](https://pkg.go.dev/golang.org/x/time/rate) — token-bucket rate limiter.
- [`golang.org/x/term`](https://pkg.go.dev/golang.org/x/term) — terminal dimensions for banner alignment.
- [`github.com/sirupsen/logrus`](https://github.com/sirupsen/logrus) — structured logging.
- The Go standard library's `net/http` — one of the finest HTTP clients in any language.
```