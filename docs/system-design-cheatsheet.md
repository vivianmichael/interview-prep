# System Design CheatSheet

## Numbers to Memorize

### Time and scale
| Item | Value |
|---|---|
| Seconds per day | 86,400 → round to 100,000 |
| 1M requests/day | ≈ 12 QPS |
| 100M requests/day | ≈ 1,200 QPS |
| 1B requests/day | ≈ 12K QPS |
| Peak vs average | 2-5× (flash sales / viral: 10×+) |
| Mental-math units | Use powers of 1000: KB 10³, MB 10⁶, GB 10⁹, TB 10¹², PB 10¹⁵ |
| 99.9% / 99.99% availability | 8.76 h/yr / ~52 min/yr |

### Latency
| Item | Value |
|---|---|
| Memory reference | ~100 ns |
| SSD random read | ~16-100 µs |
| Read 1 MB sequentially: memory / SSD / HDD | 0.25 ms / 1 ms / 20 ms |
| Round trip within an AZ | < 1 ms |
| Round trip across AZs (same region) | 1-2 ms |
| Round trip cross-region | 50-150 ms |
| DB read | 1-5 ms (cached), 5-30 ms (disk) |

### Modern single-node limits (don't over-shard)
| Component | Ballpark |
|---|---|
| App server | 100K+ concurrent connections; CPU-bound; 1K-10K+ req/s depending on work per request |
| Relational DB (single primary) | up to ~64 TiB; 10-20K write TPS; ~5-50K simple reads/s; ~20K connections |
| Redis / cache | 100K+ ops/s; up to ~1 TB RAM |
| Kafka broker | up to ~1M msgs/s; 1-5 ms latency; up to ~50 TB |
| Kafka partition | ~10 MB/s (benchmark it). Partitions = max(throughput ÷ 10 MB/s, consumer instances) |
| Chat server | 100K+ connections typical; ~1-2M per host is the WhatsApp-style ceiling |
| Datacenter network | 25 Gbps common; 50-100+ Gbps high-end |

### Sizes
| Item | Value |
|---|---|
| 2-hour movie | ~1 GB |
| Small plain-text book | ~1 MB |
| High-res photo | ~1 MB |
| Medium image | ~100 KB |
| Tweet / short text | ~300 B |
| Typical DB row | 100 B-1 KB |

# Back-of-the-Envelope Estimation Cheat Sheet

## Interview Scenario → What to Estimate

| Interview scenario | What to estimate | Key formula / hint |
|---|---|---|
| Design a URL shortener | Requests/sec, storage, DB writes, read/write ratio | Variant A: 100M writes/day, 100:1 reads. Variant B: 1B total URLs, 100M DAU, ~1000:1 reads (6-char base62 suffices; ~500 GB total, fits one DB). Cache = unique hot URLs × size |
| Design a payment API | Peak RPS, transaction throughput, database capacity, idempotency requirements | Peak RPS = avg × 3-10 (sales spikes); idempotency key store = txns/day × retention window × key size |
| Design a chat system | Concurrent connections, messages/sec, delivery fan-out | 1B users → ~200M concurrent; 200M active × 20 msgs/day = 4B/day ≈ 40K msgs/s; ~100K writes/s with inbox + groups; servers = connections ÷ 100K-1M |
| Design a file-storage service | Storage growth, upload throughput, download bandwidth | Storage = users × files × avg size × replication (3×); bandwidth = QPS × file size |
| Design a notification system | Events/sec, queue throughput, consumer capacity | Events/sec = notifications/day ÷ 86,400 (peak 5×); consumers = events/sec ÷ per-consumer rate |
| Design a video-streaming platform | Concurrent viewers, bandwidth, storage, CDN capacity | Egress = concurrent viewers × bitrate; storage = uploads/day × length × encodings (5-8 formats) |
| Design a news feed (Twitter/Instagram) | Read/write QPS, fan-out writes per post, feed cache size | Fan-out = posts/day × avg followers; celebrity accounts → hybrid push/pull |
| Design a rate limiter | Requests/sec checked, counter state in memory, Redis ops/sec | Memory = users × counters × bytes per counter; ops = 1-2 Redis calls per request |
| Design a web crawler | Pages/sec, total storage, bandwidth, dedup index size | Pages/sec = total pages ÷ crawl window; storage = pages × avg page size (~100 KB) |
| Design search / typeahead | Queries/sec, index size, memory for hot prefixes | Index = docs × avg size × 1.5-3×; typeahead QPS = searches × chars typed |
| Design a ride-hailing service | Location updates/sec, matching requests/sec, geo-index size | Updates/sec = active drivers ÷ update interval (e.g. 1M ÷ 4s = 250K/s) |
| Design a key-value store / cache | Dataset size, QPS, memory, number of nodes | Nodes = max(size ÷ RAM per node, QPS ÷ QPS per node); cache ≈ 20% of hot data |
| Design a logging / metrics system | Events/sec, MB/sec ingest, retention storage | Kafka partitions = max(MB/s ÷ 10, consumers); storage = MB/s × retention × replication |
| Design an e-commerce / flash sale | Peak concurrent users, orders/sec, inventory write contention | Peak = avg × 10-50 for flash sales; hot-item writes/sec per SKU |
| Design a ticket booking system | Concurrent seat selections, hold duration, DB lock contention | Active holds = bookings/sec × hold duration (e.g. 10 min) |
| Design a distributed ID generator | IDs/sec, ID bit layout, uniqueness horizon | Snowflake: 41-bit time + 10-bit machine + 12-bit sequence = 4096 IDs/ms/node |
| Design a collaborative editor | Concurrent editors per doc, ops/sec, document storage with history | Ops/sec = active editors × edits per second; storage includes version history |
| Design a leaderboard | Score updates/sec, sorted-set size, top-N read QPS | Redis sorted set: ~100 B per member × players |

## Reusable Estimation Template

Copy this block for each problem.

### 1. Assumptions
- MAU: ____
- DAU: ____ (MAU × 20-50%)
- Actions per user per day: ____
- Read : write ratio: ____
- Average payload size: ____
- Retention period: ____ years
- Replication factor: 3

### 2. Traffic
- Write QPS = (DAU × writes/user/day) ÷ 86,400 = ____
- Read QPS = Write QPS × ratio = ____
- Peak QPS = avg × (2-5) = ____

### 3. Storage
- Per day = writes/day × payload size = ____
- Per year = per day × 365 = ____
- Total = per year × years × replication = ____

### 4. Bandwidth
- Ingress = write QPS × payload size = ____
- Egress = read QPS × payload size = ____

### 5. Memory (cache)
- Cache size = daily read data × 0.2 = ____

### 6. Derived infrastructure
- App servers = peak QPS ÷ QPS per server = ____
- DB shards = total storage ÷ storage per node = ____
- Cache nodes = cache size ÷ RAM per node = ____

### 7. Sanity check
- [ ] Does any single node exceed its limit?
- [ ] Is the read or write path the bottleneck?
- [ ] Do the numbers look reasonable vs. real-world systems?
- 
### When sharding is actually justified
- Dataset beyond tens of TB, or
- Write throughput beyond ~10-20K TPS on one primary, or
- Operational pain (backup windows, maintenance), not just "it's big"

### Interview discipline
- Estimate only what changes a design decision (the crux, then bottleneck QPS and storage).
- Spend ~1-3 minutes; do the math just-in-time at the bottleneck.
- Round aggressively, say assumptions out loud, and tie every number to a decision.