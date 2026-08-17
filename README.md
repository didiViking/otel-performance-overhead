# ⚡ OpenTelemetry Collector Optimization Demo

A hands-on, reproducible benchmark comparing an **unoptimized** vs. **optimized** OpenTelemetry Collector pipeline under simulated load using `telemetrygen`, see https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/cmd/telemetrygen/README.md

This repository demonstrates how adding key pipeline processors (`batch`, `memory_limiter`, `transform`, and `tail_sampling`) drastically reduces network bandwidth, prevents OOM risks, and protects downstream telemetry storage.

---

## 📊 Benchmark Results Summary

| Metric | Before Optimization (`before-config.yaml`) | After Optimization (`after-config.yaml`) | Production Impact |
| :--- | :--- | :--- | :--- |
| **Network Egress** | ~20.5 MB | **~401 kB** | **~98% reduction** in network bandwidth |
| **I/O & Flushes** | High frequency (every ~20ms) | Low frequency (aggregated batches) | Reduced network socket and CPU churn |
| **OOM Safety** | None (Risk under traffic spikes) | Enforced by `memory_limiter` & `GOMEMLIMIT` | Collector stability under heavy load |
| **Data Privacy** | Raw telemetry transmitted | Static attributes scrubbed via `transform` | Enforced data compliance |

---

## 🛠️ Architecture Overview

```text
[ telemetrygen ]  --->  [ OTLP Receiver (gRPC :4317) ]
                                 │
                                 ▼
                     [ Processors Pipeline ]
                     ├── memory_limiter
                     ├── transform (scrub static tags)
                     ├── batch
                     └── tail_sampling (10% sample)
                                 │
                                 ▼
                     [ Debug Exporter (stdout) ]
```

---

## 🚀 Step-by-Step Reproduction Guide

### **Prerequisites**
* [Docker](https://docs.docker.com/get-docker/) & [Docker Compose](https://docs.docker.com/compose/) installed.

---

### **Step 1: Run the Unoptimized Pipeline ("Before")**

1. Ensure `docker-compose.yaml` mounts `./before-config.yaml`.
2. Start the Collector container:
   ```bash
   docker-compose down && docker-compose up -d
   ```
3. Generate load using `telemetrygen` (runs for 30s at 1,000 spans/sec):
   ```bash
   docker run --rm --network otel-live-demo_default ghcr.io/open-telemetry/opentelemetry-collector-contrib/telemetrygen:latest traces \
     --otlp-endpoint=otel-collector:4317 \
     --otlp-insecure \
     --rate=1000 \
     --duration=30s \
     --workers=4
   ```
4. Observe high-frequency log flushes:
   ```bash
   docker logs --tail 20 otel-collector
   ```
5. Capture resource stats immediately after the run:
   ```bash
   docker stats otel-collector --no-stream
   ```
   * **Note `NET I/O`:** Notice the **In / Out** ratio is nearly 1:1 (~20MB received / ~20MB exported).

---

### **Step 2: Run the Optimized Pipeline ("After")**

1. Edit `docker-compose.yaml` to point the volume mount to `./after-config.yaml`:
   ```yaml
   volumes:
     - ./after-config.yaml:/etc/otelcol-contrib/config.yaml
   ```
2. Restart the Collector:
   ```bash
   docker-compose down && docker-compose up -d
   ```
3. Run the exact same `telemetrygen` benchmark:
   ```bash
   docker run --rm --network otel-live-demo_default ghcr.io/open-telemetry/opentelemetry-collector-contrib/telemetrygen:latest traces \
     --otlp-endpoint=otel-collector:4317 \
     --otlp-insecure \
     --rate=1000 \
     --duration=30s \
     --workers=4
   ```
4. Observe aggregated flushes:
   ```bash
   docker logs --tail 20 otel-collector
   ```
5. Capture resource stats:
   ```bash
   docker stats otel-collector --no-stream
   ```
   * **Observe `NET I/O`:** Output is reduced down to **~400kB** (~98% network bandwidth saved).

---

## 💡 Key Architectural Takeaways

1. **Trade RAM for Downstream Savings:** `tail_sampling` uses a small memory buffer (~120 MiB) to evaluate traces over a window before deciding to drop or forward them. This minor RAM trade-off prevents gigabytes of unnecessary trace storage downstream.
2. **Always Place `memory_limiter` First:** In the processor execution chain, `memory_limiter` must precede `batch` or `tail_sampling` to prevent out-of-memory errors during traffic bursts.
3. **Batching Reduces Network Overhead:** Grouping telemetry items into large payloads (`send_batch_size: 8192`) dramatically improves TCP efficiency compared to sending individual spans.
