
# System Design

## FAQ's

### How do you guarantee message ordering in Kafka?

Kafka guarantees ordering only **within a single partition**, never across
partitions of a topic.

**Approaches:**

**Use a message key (preferred).**

   Messages with the same key always go to the same partition
   (`hash(key) % numPartitions`), preserving ordering per key
   (e.g. `orderId` or `userId`) while allowing the topic to scale.

**Use a single partition.**

   This guarantees total ordering for the topic, but only one consumer
   in a consumer group can read it. Throughput and scalability are limited.

**Producer settings to avoid reordering on retries:**

```properties
enable.idempotence=true
acks=all
max.in.flight.requests.per.connection=5
```

Without idempotence, retries can cause a later batch to be written
before an earlier failed batch.

**Consumer side:**

- Within a consumer group, one consumer owns a partition at a time.
- Multiple threads or asynchronous processing can break ordering again.
- Process sequentially per partition, or hand off processing by key.
- Increasing the partition count can change key-to-partition mapping.

<details markdown="1">
<summary>Follow-up questions</summary>

- What happens if you add partitions to an existing topic?
- What is an idempotent producer and how does it prevent duplicates?
- How do you get ordering and high throughput together?

</details>

---

### One producer, three services consuming the same message: how do you set it up?

**Answer:**

Give each service its **own consumer group** (a different `group.id`).

- Kafka delivers each message to every subscribed consumer group.
- Different groups receive messages independently and maintain their own offsets.
- Consumers sharing the same group split partitions between themselves.

**Architecture:**

```text
                         ┌──> billing-service
                         │    group.id=billing-service
                         │
Producer ──> Kafka Topic ├──> notification-service
                         │    group.id=notification-service
                         │
                         └──> analytics-service
                              group.id=analytics-service
```

**Configuration per service:**

```properties
# billing-service
group.id=billing-service

# notification-service
group.id=notification-service

# analytics-service
group.id=analytics-service
```

**Scaling each service:**

If a service runs multiple instances, all instances use the same `group.id`.
Kafka distributes partitions across those instances.

Maximum useful active consumers per group = number of partitions.

**Other important points:**

- Each group tracks its own offsets.
- A slow or unavailable service doesn't block other groups.
- Use appropriate `retention.ms` so a service can catch up after downtime.
- A new group can replay from the beginning using `auto.offset.reset=earliest`.

<details markdown="1">
<summary>Follow-up questions</summary>

- What happens if two services accidentally use the same `group.id`?
- How does a consumer group rebalance work?

</details>