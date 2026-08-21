# ⚡ OpenTelemetry Collector Optimization Demo

A hands-on, reproducible benchmark comparing an **unoptimized** vs. **optimized** OpenTelemetry Collector pipeline under simulated load using [`telemetrygen`](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/cmd/telemetrygen/README.md).

This repository demonstrates how adding and correctly ordering key pipeline processors (`memory_limiter`, `transform`, `batch`, and `tail_sampling`) drastically cuts memory consumption, prevents OOM risks, reduces disk thrashing, and protects downstream telemetry backends.

---

## 📊 Benchmark Results Summary

| Metric | Before Optimization (`before-config.yaml`) | After Optimization (`after-config.yaml`) | Production Impact |
| :--- | :--- | :--- | :--- |
| **Memory Footprint** | **~149.3 MiB** | **~86.0 MiB** | **~42% Memory Reduction** |
| **Disk Block I/O** | **92.9 MB** | **8.65 MB** | **90.7% Reduction** in disk stress/thrashing |
| **Payload Batch Size** | 100 spans / payload | **2 spans / payload** | Reduced payload footprint via filtering |
| **CPU Behavior** | Flat (1.08% - 1.34%) | Controlled spikes (2.18% - 4.10%) | Efficient batch-flush processing |
| **OOM Safety** | None (Risk under traffic spikes) | Enforced by `memory_limiter` & `GOMEMLIMIT` | Collector stability under heavy load |
| **Data Privacy** | Raw telemetry transmitted | Static attributes scrubbed via `transform` | Enforced data compliance |

---

## 🛠️ Architecture Overview

```text
[ telemetrygen ]  --->  [ OTLP Receiver (gRPC :4317) ]
                                  │
                                  ▼
                      [ Processors Pipeline ]
                      ├── memory_limiter         (Always first to safeguard memory)
                      ├── transform              (Scrub static tags / scrub sensitive attributes)
                      ├── batch                  (Aggregate spans for payload efficiency)
                      └── tail_sampling          (Sample spans based on trace evaluation)
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
   * Notice exported spans drop significantly per payload: "spans": 2  
5. Capture resource stats:
   ```bash
   docker stats otel-collector --no-stream
   ```
   * Observe Improvements:

    * Memory drops to ~86 MiB (~42% reduction).

    * Block I/O drops to ~8.65 MB (90.7% drop in disk operations).

    * Note on Docker NET I/O: Docker's NET I/O metric displays cumulative network traffic transferred over the entire                container lifecycle. Because synthetic traffic flows continuously during tests, NET I/O total bytes continuously               increase. True network and transport optimization is demonstrated through reduced span payload counts and improved             compression ratios.

---

## 💡 Key Architectural Takeaways

1. **Always Place `memory_limiter` First:** Placing `memory_limiter` ahead of heavy allocation processors (`resourcedetection`, `tail_sampling`) prevents unnecessary memory consumption on data that might be dropped under backpressure.
2. **Reduced Disk Thrashing via Streamlined Buffering:** Correctly configuring `batch` and `memory_limiter` upstream minimizes disk buffer swaps, slashing Block I/O by over 90%.
3. **Payload Efficiency Over Raw Packet Counts:** Batching groups individual telemetry items into single HTTP/gRPC requests (`send_batch_size: 8192`), significantly increasing payload compression efficiency and cutting total connection handshakes.
