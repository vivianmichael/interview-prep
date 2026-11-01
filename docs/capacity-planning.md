# Capacity Planning: Turning Demand into a Server Count

One of the most common system design interview questions is:

> "Our application receives 10,000 requests per second. How many servers do we need?"

A senior backend engineer should be able to answer this systematically rather than guessing.

The process is straightforward:

1. Understand the incoming traffic.
2. Measure the capacity of a single server.
3. Convert demand into required server count.
4. Account for failures and production headroom.
5. Validate downstream dependencies such as databases and external services.

---
## 1. Understand the Incoming Traffic

Before estimating server count, you must understand the workload.

### Requests Per Second (RPS)

**RPS (Requests Per Second)** measures how many requests arrive at your system each second.

| Traffic | Meaning |
|----------|---------|
| 10 RPS | 10 requests/second |
| 100 RPS | 100 requests/second |
| 1,000 RPS | 1,000 requests/second |
| 10,000 RPS | 10,000 requests/second |

Example:

```text
1,000 RPS
```

Requests per minute:

```text
1,000 × 60 = 60,000
```

Requests per hour:

```text
1,000 × 3,600 = 3,600,000
```

---
### Average Traffic vs Peak Traffic

Suppose your application receives:
```text
8,640,000 requests/day
```
Average RPS:
```text
8,640,000 / 86,400 (seconds in a day) = 100 RPS
```

However:
```text
Average RPS = 100
Peak RPS = 800
```
If you size your infrastructure for 100 RPS, your application may fail during business-hour spikes.

### Capacity Planning Focus

Always plan for:

- Peak RPS
- Peak duration
- Traffic growth
- Traffic spike patterns
- Read-heavy vs write-heavy workloads

Peak traffic determines the final infrastructure sizing, not daily averages.**

---
## 2. Understand Server Capacity

The next question is:

> How many requests can one server handle?

There is no universal answer.

Capacity depends on:

- CPU cores and RAM
- Database latency
- External API latency
- Thread pools & Connection pools
- Garbage collection
- Payload size
- Business logic complexity

---

### Example: Two Endpoints, Different Cost

### Read Endpoint

```java
@GetMapping("/payments/{id}")
public Payment getPayment(@PathVariable String id) {
    return paymentService.findById(id);
}
```

### Write Endpoint

```java
@PostMapping("/payments")
public Payment createPayment(
        @RequestBody PaymentRequest request) {

    return paymentService.createPayment(request);
}
```
The second endpoint might:

- Validate input +  Write to DB + Publish Kafka event + Call payment gateway + Write audit logs

Both endpoints are requests, but they consume very different amounts of resources.

---

### Measured Capacity Matters

Suppose load testing produces:

| Metric | Value |
|----------|---------|
| Instance Size | 4 vCPU, 8GB RAM |
| Throughput | 800 RPS |
| Average Latency | 40 ms |
| p95 Latency | 90 ms |
| CPU Usage | 65% |

Assume:

```text
800 RPS
```

is the maximum sustainable throughput while still meeting latency and error-rate requirements.

### Important

Capacity should come from:

- Load testing
- Benchmarking
- Production monitoring
Not from CPU count alone.

---

## 3. Converting Demand into Server Count

Assume:

| Input | Value |
|---------|---------|
| Peak Traffic | 10,000 RPS |
| Measured Server Capacity | 800 RPS |
| Target Utilization | 70% |

---

### Step 1: Calculate Usable Capacity

Formula:

```text
Usable Capacity  =  Measured Capacity × Target Utilization
```

Example:

```text
800 × 0.7 = 560 RPS
```

Each server contributes:

```text
560 RPS
```

of planned capacity.

Why not use 100%?

Because:

- Traffic fluctuates
- Databases slow down
- Background jobs consume resources
- Retries happen
- Latency increases near capacity

---

### Step 2: Calculate Required Servers

Formula:

```text
Servers Required = Peak RPS / Usable Capacity
```

```text
10,000 / 560 = 17.86
```

Round up:

```text
18 servers
```

### Verification

```text
18 × 560 = 10,080 RPS
```

The system can handle the target load.

---

## Core Capacity Planning Formula

```text
Servers Required = ceil( Peak RPS / (Measured RPS per Server × Utilization Target)
)
```

Example:

```text
ceil(10,000 / (800 × 0.70)) = 18
```

---

## 4. Plan for Failure (N+1 Capacity)

A production system must survive failures.

Suppose:

```text
18 servers are deployed.
```

If one server dies:

```text
17 × 560 = 9,520 RPS
```

Remaining capacity:

```text
9,520 RPS
```

Required demand:

```text
10,000 RPS
```

Shortfall:

```text
480 RPS
```

The system is under-provisioned.

---

### Add Redundancy

Requirement:

```text
(N - 1) × 560 >= 10,000
```

Calculation:

```text
N >= 18.86
```

Round up:

```text
19 servers
```

---

### Verification

| Scenario | Servers Available | Capacity | Can Handle 10k RPS? |
|-----------|------------------|-----------|---------------------|
| Normal | 19 | 10,640 RPS | ✅ |
| One Failure | 18 | 10,080 RPS | ✅ |
| Two Failures | 17 | 9,520 RPS | ❌ |

---

### N+1 Capacity Planning

Definition:

```text
Minimum Required Capacity + One Additional Server
```
This ensures that losing one instance does not impact customers.

---

## Availability Zone Failure

Consider:

```text
19 Servers
```

distributed across:

```text
Zone A = 7
Zone B = 6
Zone C = 6
```

If Zone A fails:

```text
12 Servers Remain
```

Capacity:

```text
12 × 560 = 6,720 RPS
```

Required:
```text
10,000 RPS
```
Not enough.

### Lesson

Always design capacity for the failure scenario you care about:

- Single instance failure
- Rack failure
- Availability zone failure
- Region failure

---

## 5. Understanding Concurrency

Capacity planning is not only about throughput.

It is also about:

```text
How many requests are active simultaneously?
```

This is called:

**Concurrency**

---

### Little's Law

Formula:

```text
L = λ × W
```

Where:

| Variable | Meaning |
|-----------|----------|
| L | Requests in flight |
| λ | Arrival Rate (RPS) |
| W | Response Time (seconds) |

---

### Example A: Fast Service

Traffic:

```text
1,000 RPS
```

Response Time:

```text
100 ms = 0.1 sec
```

Calculation:

```text
1,000 × 0.1 = 100
```

Concurrency:

```text
100 requests in flight
```

---

### Example B: Slow Service

Traffic:

```text
1,000 RPS
```

Response Time:

```text
500 ms = 0.5 sec
```

Calculation:

```text
1,000 × 0.5 = 500
```

Concurrency:

```text
500 requests in flight
```

---

### Key Observation

| Metric | Fast | Slow |
|----------|------|------|
| Traffic | 1,000 RPS | 1,000 RPS |
| Response Time | 100 ms | 500 ms |
| Concurrent Requests | 100 | 500 |

Traffic did not change.

Concurrency increased **5x**.

---

## Why This Matters in Java

Consider a Spring Boot application using JDBC.

Slow database queries:

1. Hold connections longer.
2. Increase connection pool usage.
3. Cause requests to wait.
4. Increase response times.
5. Increase concurrency.
6. Create even more pressure on the database.

Feedback loop:

```text
Slow Query
      ↓
Connections Held Longer
      ↓
Pool Saturation
      ↓
Requests Wait
      ↓
Higher Latency
      ↓
More Requests In Flight
      ↓
Additional System Pressure
```

Adding application servers alone may not solve the problem.

The bottleneck may be:

- Database
- Cache
- Kafka
- External provider
- Network

---

## End-to-End Example: Payment API

Requirements:

| Input | Value |
|---------|---------|
| Peak Traffic | 12,000 RPS |
| Measured Capacity | 1,000 RPS per instance |
| Utilization Target | 70% |
| Failure Requirement | Survive 1 instance failure |

---

### Step 1: Calculate Usable Capacity

```text
1,000 × 0.70 = 700 RPS
```

---

### Step 2: Calculate Minimum Instances

```text
12,000 / 700 = 17.14
```

Round up:

```text
18 instances
```

---

### Step 3: Add Failure Capacity

To survive one instance loss:

```text
18 + 1 = 19 instances
```

---

### Verification

Normal:

```text
19 × 700 = 13,300 RPS
```

One instance down:

```text
18 × 700 = 12,600 RPS
```

Still above target:

```text
12,000 RPS
```

✅ Requirement met

---

## The Hidden Bottleneck: Database Capacity

Suppose:

```text
API Traffic = 12,000 RPS
```

Each request generates:

```text
1 SQL Query
```

Database capacity:

```text
8,000 QPS
```

The application layer can handle:

```text
13,300 RPS
```

But the database can only process:

```text
8,000 QPS
```

The database becomes the bottleneck.

---

### Potential Solutions

- Query optimization
- Better indexes
- Caching
- Read replicas
- Reduced database calls
- Batch processing
- Asynchronous workflows
- Event-driven architecture

---

## Capacity Planning Cheat Sheet

| Step | Question |
|--------|-----------|
| 1 | What is peak traffic? |
| 2 | How much can one instance handle? |
| 3 | What utilization target should we use? |
| 4 | How many instances are required? |
| 5 | What happens if an instance fails? |
| 6 | What happens if an AZ fails? |
| 7 | What is the system bottleneck? |
| 8 | Can the system handle spikes and retries? |

---

## Interview Answer (30-Second Version)

> Capacity planning starts with peak traffic and measured instance capacity. Determine the sustainable RPS per server through load testing, apply a utilization target (typically leaving headroom for spikes), and divide peak demand by usable capacity. Then add redundancy for failures using N+1 planning and validate that downstream systems such as databases, caches, Kafka, and external services can also sustain the required throughput. Application servers are often not the real bottleneck; the slowest component in the request path determines overall system capacity.

---

## Formula Summary

### Server Capacity

```text
Usable Capacity =
Measured Capacity × Utilization Target
```

### Server Count

```text
Servers Required = ceil( Peak RPS / Usable Capacity)
```

### Concurrency (Little's Law)

```text
Concurrency = RPS × Response Time (seconds)
```

### N+1 Planning

```text
(N - 1) × Usable Capacity >= Peak Demand
```

Always verify that the entire request path (application, database, cache, queues, and downstream services) can sustain the calculated load.
