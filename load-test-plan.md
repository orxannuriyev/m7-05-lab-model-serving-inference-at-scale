# load-test-plan.md

# Vision Moderation Service — Load Test Plan

## 1. Tool Choice

### Selected Tool: k6

k6 is chosen because it provides deterministic load generation, supports HTTP APIs natively, and produces detailed latency percentile metrics (p50, p95, p99) required for SLO validation. It also integrates easily with Prometheus and Grafana for real-time monitoring during the test.

---

## 2. Test Phases

### Phase 1 — Warmup

**Duration:** 10 minutes

Purpose:

- Warm application caches
- Warm Redis connections
- Load model into GPU memory
- Eliminate cold-start effects

Traffic:

- 50 RPS → 150 RPS gradual ramp

---

### Phase 2 — Ramp-Up

**Duration:** 10 minutes

Purpose:

- Validate autoscaling behavior
- Observe latency growth under increasing load

Traffic:

- 150 RPS → 300 RPS
- Linear increase

---

### Phase 3 — Sustained Peak Load

**Duration:** 30 minutes

Purpose:

- Verify production operating conditions
- Validate p95 latency target

Traffic:

- Constant 300 RPS

---

### Phase 4 — Spike Test

**Duration:** 5 minutes

Purpose:

- Validate handling of documented traffic spikes

Traffic:

- Constant 500 RPS

Expected behavior:

- Autoscaler increases replicas from 3 to 4
- No SLA violations

---

### Phase 5 — Soak Test

**Duration:** 4 hours

Purpose:

- Detect memory leaks
- Detect GPU memory growth
- Detect connection pool exhaustion
- Validate long-term stability

Traffic:

- Constant 250 RPS

---

## 3. Traffic Shape

### Endpoint Mix

The production workload distribution will be reproduced.

| Endpoint | Traffic Share |
|-----------|--------------:|
| Synchronous moderation API | 90% |
| Batch moderation API | 10% |

---

### Payload Size Distribution

Representative image sizes:

| Payload Size | Traffic Share |
|-------------|--------------:|
| 100 KB | 20% |
| 500 KB | 50% |
| 1 MB | 25% |
| 3 MB | 5% |

This distribution approximates real-world image upload behavior.

---

### Concurrency Model

Closed-loop virtual-user model using k6.

Target concurrency:

| Load Level | Approximate Concurrent Requests |
|------------|-------------------------------:|
| 300 RPS | 75–100 |
| 500 RPS | 125–150 |

Requests will arrive continuously and independently to simulate production traffic.

---

## 4. Pass / Fail Criteria

### Availability

PASS:

- Success rate ≥ 99.9%

FAIL:

- Success rate < 99.9%

---

### Error Rate

PASS:

- Error rate ≤ 0.1%

FAIL:

- Error rate > 0.1%

Includes:

- HTTP 5xx
- Timeouts
- Connection failures

---

### Latency

#### Synchronous Endpoint

PASS:

- p50 ≤ 150 ms
- p95 ≤ 250 ms
- p99 ≤ 500 ms

FAIL:

- Any threshold exceeded

---

### Batch Endpoint

PASS:

- p95 ≤ 2 seconds

FAIL:

- p95 > 2 seconds

---

### Autoscaling

PASS:

- Scale-out completes before sustained SLA violation occurs

FAIL:

- Latency exceeds SLA because scaling cannot keep up

---

## 5. Bottleneck Checklist

During testing, inspect every serving replica and downstream dependency.

### Application Layer

Monitor:

- Request rate (RPS)
- Active connections
- Request queue depth
- Thread pool utilization
- Dynamic batching statistics

---

### CPU Resources

Monitor:

- CPU utilization (%)
- Load average
- Context switches
- CPU throttling

Target:

- CPU utilization < 80%

---

### Memory Resources

Monitor:

- Memory utilization
- Heap growth
- Memory leaks
- Garbage collection activity

Target:

- Memory utilization < 85%

---

### GPU Resources

Monitor:

- GPU utilization (%)
- GPU memory utilization
- GPU temperature
- Inference queue depth
- Batch size distribution

Target:

- GPU utilization 60–85%
- GPU memory < 90%

---

### Redis

Monitor:

- Redis p50 latency
- Redis p95 latency
- Connection count
- Cache hit rate
- Command throughput

Target:

- Redis p95 < 15 ms

---

### Network

Monitor:

- Inbound throughput
- Outbound throughput
- Packet loss
- Connection errors

Target:

- Packet loss = 0%

---

### Kubernetes / Infrastructure

Monitor:

- Replica count
- Pod restart count
- OOM kills
- Container CPU throttling
- Autoscaler events

Target:

- Zero pod crashes
- Zero OOM events

---

## Success Criteria Summary

The load test is considered successful if:

- Sustained 300 RPS is maintained for 30 minutes
- 500 RPS spikes are handled for 5 minutes
- Availability remains ≥ 99.9%
- Error rate remains ≤ 0.1%
- p95 latency remains ≤ 250 ms
- p99 latency remains ≤ 500 ms
- Redis p95 remains ≤ 15 ms
- No memory leaks, OOM events, or GPU saturation are observed