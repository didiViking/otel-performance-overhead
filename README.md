# ⚡ OpenTelemetry Collector Optimization Demo

A hands-on, reproducible benchmark comparing an **unoptimized** vs. **optimized** OpenTelemetry Collector pipeline under simulated load using [`telemetrygen`](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/cmd/telemetrygen/README.md) (accompanying the blog post [**Is Your OpenTelemetry Pipeline Becoming a Performance Bottleneck?**](https://your-blog-link-here)).

This repository demonstrates how configuring and correctly ordering key pipeline components (`memory_limiter`, `resource_detection`, `transform`, `tail_sampling`, and exporter-level `sending_queue.batch`) drastically cuts outbound network traffic, enforces OOM safety, and protects downstream telemetry backends.

---

## 📊 Benchmark Results Summary

Results from running ~54,000 traces over 30 seconds (`4` workers) on OpenTelemetry Collector `v0.158.0` (`512 MiB` container memory limit).

| Metric | Before Optimization (`before-config.yaml`) | After Optimization (`after-config.yaml`) | Production Impact |
| :--- | :--- | :--- | :--- |
| **Outbound Network I/O** | **3.33 MB exported** (`14 MB` in) | **431 kB exported** (`14.8 MB` in) | **~87% Reduction** in egress bandwidth & backend ingestion |
| **Peak Memory Footprint** | **82.88 MiB** | **168.0 MiB** | Expected buffering trade-off for 10s `tail_sampling` + `100k`-item queue |
| **CPU Behavior** | 2.93% – 8.42% | 7.79% – 25.51% | Active trace-ID evaluation, attribute scrubbing & batch compression |
| **OOM Safety** | None (Risk under traffic spikes) | Enforced by `memory_limiter` (`350 MiB`) & `GOMEMLIMIT` | Collector stability under heavy load |
| **Batching Architecture** | Unbatched frequent exports | Exporter-level `sending_queue.batch` (`min_size: 8192`) | Replaces deprecated `batch` processor |
| **Data Privacy & Metadata** | All host attributes + raw headers sent | High-detail host attributes trimmed & sensitive headers scrubbed | Reduced payload size & enforced compliance |

---

## 🛠️ Architecture Overview

```text
[ telemetrygen ]  --->  [ OTLP Receiver (gRPC :4317) ]
                                  │
                                  ▼
                      [ Processors Pipeline ]
                      ├── memory_limiter         (Always first to safeguard memory)
                      ├── resource_detection     (Trim unnecessary host metadata)
                      ├── transform              (Scrub sensitive span attributes)
                      └── tail_sampling          (Retain errors + 5% probabilistic sample)
                                  │
                                  ▼
                      [ OTLP gRPC Exporter ]
                      └── sending_queue.batch    (Group & compress surviving spans)
                                  │
                                  ▼
                      [ Mock OTLP Backend ]
```

---

## 🚀 Step-by-Step Reproduction Guide

### **Prerequisites**
* [Docker](https://docs.docker.com/get-docker/) & [Docker Compose](https://docs.docker.com/compose/) installed.

---

### **Step 1: Run the Unoptimized Pipeline ("Before")**

1. Ensure `docker-compose.yaml` mounts `./before-config.yaml`.
2. Start the Collector and mock backend containers:
   ```bash
   docker-compose down && docker-compose up -d
   ```
3. Generate load using `telemetrygen` (runs for 30s at 1,000 traces/sec per worker):
   ```bash
   docker run --rm --network otel-live-demo_default ghcr.io/open-telemetry/opentelemetry-collector-contrib/telemetrygen:latest traces \
     --otlp-endpoint=otel-collector:4317 \
     --otlp-insecure \
     --rate=1000 \
     --duration=30s \
     --workers=4
   ```
4. Observe downstream exports on the mock backend:
   ```bash
   docker logs --tail 20 otel-backend
   ```
5. Capture resource stats during and immediately after the run:
   ```bash
   docker stats otel-collector --no-stream
   ```
   * **Note `NET I/O`:** With no sampling or attribute trimming, `14 MB` inbound produces **`3.33 MB` outbound** across frequent unbatched exports.

---

### **Step 2: Run the Optimized Pipeline ("After")**

1. Edit `docker-compose.yaml` to point the volume mount to `./after-config.yaml`:
   ```yaml
   volumes:
     - ./after-config.yaml:/etc/otelcol-contrib/config.yaml
   ```
2. Restart the stack:
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
4. Observe batched exports arriving at the mock backend:
   ```bash
   docker logs --tail 20 otel-backend
   ```
5. Capture resource stats:
   ```bash
   docker stats otel-collector --no-stream
   ```
   * **Observe Key Results & Trade-offs:**
     * **Outbound `NET I/O` drops by ~87%:** `14.8 MB` inbound results in only **`431 kB` outbound**, dramatically reducing downstream network and backend ingestion costs.
     * **Memory & CPU Trade-off:** Peak memory rises from `82.88 MiB` to `168.0 MiB` because `tail_sampling` holds traces in memory for the `10s` `decision_wait` window before forwarding surviving spans into the `100,000`-item `sending_queue`. `memory_limiter` (`350 MiB`) ensures this buffering stays safely below the `512 MiB` container limit.

---

## 💡 Key Architectural Takeaways

1. **Always Place `memory_limiter` First:** Placing `memory_limiter` ahead of heavy allocation processors (`resource_detection`, `tail_sampling`) prevents unnecessary memory consumption on data that might be refused under backpressure.
2. **Use Exporter-Level Batching (`sending_queue.batch`):** Because the standalone `batch` processor is deprecated, configuring `batch` inside the `otlp_grpc` exporter's `sending_queue` (`min_size: 8192`, `flush_timeout: 200ms`) groups surviving spans after sampling and improves delivery durability.
3. **Observability Optimization Is a Resource Trade-off:** Stateful optimizations like `tail_sampling` and large sending queues intentionally trade modest local Collector RAM and CPU (`168 MiB` peak) for massive downstream bandwidth and backend storage savings (~87% egress reduction).
