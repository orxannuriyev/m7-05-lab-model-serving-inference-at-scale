# capacity-plan.md

# Vision Moderation Service — Capacity Plan

## 1. Latency Budget Breakdown

Target synchronous endpoint p95 latency: **250 ms**

### Assumptions

- Redis p95 ≈ 8 ms
- Pre/post-processing combined ≈ 15 ms
- Network in/out combined ≈ 10 ms
- GPU inference median ≈ 22 ms
- Budget is designed for p95, not median

| Stage | Budget (ms) | Notes |
|---------|---------:|---------|
| Network in | 5 | Client → LB → service |
| Auth + routing | 5 | API gateway and service routing |
| Payload parse | 10 | Request validation and image decode |
| Feature lookup (Redis) | 15 | Above observed 8 ms p95 |
| Pre-processing | 15 | Resize, normalization, tensor conversion |
| Model inference | 40 | GPU-backed inference with queueing allowance |
| Post-processing | 10 | Thresholding and label formatting |
| Serialization | 5 | JSON encoding |
| Network out | 5 | Service → client |
| **Headroom** | **140** | Burst absorption, jitter, retries |
| **Total** | **250** | |

### Headroom Calculation

Operational path latency:

- Network in: 5 ms
- Auth + routing: 5 ms
- Payload parse: 10 ms
- Redis lookup: 15 ms
- Pre-processing: 15 ms
- Inference: 40 ms
- Post-processing: 10 ms
- Serialization: 5 ms
- Network out: 5 ms

Total operational latency:

110 ms

Remaining latency budget:

250 ms − 110 ms = **140 ms headroom**

The latency budget therefore remains positive and provides sufficient protection against burst traffic and queueing delays.

---

## 2. CPU vs GPU Decision

### Decision: GPU Serving

The ONNX model requires approximately:

- CPU inference: ~75 ms per request on a single core
- T4 GPU inference: ~22 ms per request

A single CPU core can sustain approximately:

1 / 0.075 ≈ 13 RPS

A T4 GPU can process roughly:

1 / 0.022 ≈ 45 inferences/sec per execution stream

With concurrent execution streams and dynamic batching, practical throughput reaches several hundred RPS per GPU replica.

### Cost Comparison

Assumed on-demand pricing (rounded estimates):

| Instance Type | Hourly Cost | Monthly Cost |
|---------------|------------:|-------------:|
| 8 vCPU compute instance | ~$0.27/hr | ~$195/month |
| NVIDIA T4 instance | ~$0.55/hr | ~$400/month |

Sources:

- AWS EC2 pricing (m-class/c-class compute instances)
- AWS G4dn pricing (NVIDIA T4 GPU instances)

Although GPU instances cost more per node, they require significantly fewer replicas and provide much better latency characteristics. For a latency-sensitive moderation service, GPU serving offers the best balance between performance and cost.

---

## 3. Replica Sizing

### Throughput Assumptions

Conservative estimate:

- One T4 replica = 200 RPS
- Target utilization = 70%
- 30% spare capacity reserved for failover and latency protection

### Sustained Load (300 RPS)

| Metric | Value |
|---------|------:|
| Target RPS | 300 |
| Per-replica throughput | 200 |
| Raw replicas required | 2 |
| Replicas with 30% headroom | 3 |
| Monthly cost | ~$1,200 |

### Spike Load (500 RPS)

| Metric | Value |
|---------|------:|
| Target RPS | 500 |
| Per-replica throughput | 200 |
| Raw replicas required | 3 |
| Replicas with 30% headroom | 4 |
| Monthly cost if always-on | ~$1,600 |

### Scaling Strategy

The service will use **horizontal autoscaling with warm overprovisioning**.

- Maintain 3 GPU replicas during normal operation.
- Scale to 4 replicas when load approaches 400–450 RPS.
- Keep one additional replica warm to minimize scale-up latency.
- Use request queueing only as a short-term buffer during scaling events.

This approach supports both sustained and spike traffic while remaining well below the $4,000/month serving budget.

---

## 4. Batching Decision

### Synchronous Endpoint

Enable dynamic batching with:

- Maximum batch size: 4
- Maximum wait window: 5 ms

The synchronous API handles approximately 90% of traffic and must remain below a p95 latency of 250 ms. Large batches would improve GPU utilization but would also introduce queueing delays. A 5 ms batching window adds only a small amount of latency while allowing requests arriving in the same scheduling interval to be grouped efficiently.

A batch size of 4 improves GPU throughput during burst periods without significantly impacting response time. Even under peak conditions, the additional batching delay remains well within the available latency headroom.

### Batch Endpoint

Use a separate queue with more aggressive batching:

- Batch size: 16–32
- Wait window: 20–50 ms

Batch requests are throughput-oriented rather than latency-sensitive. Larger batches improve GPU utilization and reduce cost per inference. Isolating the batch queue from the synchronous path prevents partner uploads from affecting real-time moderation traffic.

This design maximizes hardware efficiency while protecting the latency SLA for synchronous requests.

---

## Summary

- **Serving choice:** NVIDIA T4 GPU
- **Sustained capacity:** 3 replicas
- **Spike capacity:** 4 replicas
- **Scaling method:** Autoscaling with warm overprovisioning
- **Synchronous batching:** Batch size 4, wait window 5 ms
- **Batch endpoint batching:** Batch size 16–32, wait window 20–50 ms
- **Estimated serving cost:** $1,200–$1,600/month
- **Budget limit:** $4,000/month
- **SLA target:** Meets 99.5% availability and p95 latency requirements