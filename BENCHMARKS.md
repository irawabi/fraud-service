# Benchmarks

Machine: MacBook (Intel x86_64), macOS 15.8, Python 3.12.8, `OMP_NUM_THREADS=1`.
Request: `POST /v1/predict` with `payloads/sample.json`, timed with `curl -w "%{time_total}s"`.

## Lab 2b: before warm-up

First three requests after a fresh server start:

| Request | Time |
|---|---|
| 1 | 0.049 s |
| 2 | 0.009 s |
| 3 | 0.012 s |

The first request pays one-time costs (scikit-learn/numpy code paths and memory are set up on the
first real prediction), so it is about 4-5x slower than the next ones.

## Lab 2b: after warm-up

First three requests after a fresh server start, with one warm-up prediction in the lifespan:

| Request | Time |
|---|---|
| 1 | 0.013 s |
| 2 | 0.013 s |
| 3 | 0.012 s |

The first customer no longer pays the start-up cost: it moved into server start.

## Lab 2b: load test

`python scripts/loadtest.py` (2,000 requests, 25 concurrent clients), 8 CPU cores, no `--reload`.

| Workers | p50 | p99 | Throughput | Errors |
|---|---|---|---|---|
| 1 | 188.0 ms | 264.4 ms | 133 req/s | 0 |
| 4 | 122.3 ms | 197.7 ms | 208 req/s | 0 |

Throughput rose about 56% and latency fell, because 4 processes score requests on 4 CPU cores in
parallel instead of one process sharing one interpreter (the GIL). The single-request time does
not change: each prediction still costs about the same; there are just more of them at once.
Not 4x, because the load generator and the 4 workers share the same 8-core laptop.

## Lab 2b: async trap (optional)

`time.sleep(0.2)` in an endpoint, 50 requests, 10 concurrent:

| Endpoint | p50 | p99 | Throughput |
|---|---|---|---|
| `async def` | 1431 ms | 7581 ms | 5 req/s |
| plain `def` | 215 ms | 243 ms | 45 req/s |

Blocking code inside `async def` freezes the single event loop, so requests run one at a time.
A plain `def` runs in a thread pool, so the requests overlap. That is why `/predict` is a plain `def`.

## Lab 3: image size and build time

| Image | Size | Cold build | User | Fake secret inside? |
|---|---|---|---|---|
| `fraud-service:naive` (`python:3.12`, `COPY . .`) | 3.26 GB | 390 s (446 MB context) | root | **yes** (`leaked.env`) |
| `fraud-service:dev` (multi-stage, `python:3.12-slim`) | 802 MB | 195 s (2 kB context) | appuser | no |
| `fraud-service:dev` after a one-line code change | | **4.6 s** (only `COPY src/` re-ran) | | |

The naive image also contained the Mac's `.venv` (470 MB of macOS binaries), `.git`, the data and the scored CSVs.

## Lab 3: inside the container (compose, 2 uvicorn workers, 512 MB limit)

`python scripts/loadtest.py` from the Mac to `localhost:8000` (2,000 requests, 25 concurrent):

| Setup | p50 | p99 | Throughput |
|---|---|---|---|
| Lab 2b, on the Mac, 1 worker | 188 ms | 264 ms | 133 req/s |
| Lab 3, container (as shipped) | 654 ms | 1032 ms | 38 req/s |
| Lab 3, same image, `-e OMP_NUM_THREADS=1` (experiment only) | 380 ms | 744 ms | 62 req/s |

p99 is about 4x worse in the container. Two causes on this laptop: Docker Desktop on macOS runs Linux in a
VM, so every request crosses the VM's network bridge; and without `OMP_NUM_THREADS=1` the math libraries in
both workers start one thread per CPU (8 each) and fight over the cores. On a Linux server, without the VM,
the gap is much smaller.

`docker stats --no-stream`: fraud-api 338 MiB / 512 MiB (66%), feature-cache 11 MiB.

`time docker compose stop fraud-api`: **1.8 s**, a graceful shutdown ("shutdown complete" in the log).
Because the CMD uses exec form, uvicorn is PID 1 and receives SIGTERM directly instead of being killed after 10 s.
