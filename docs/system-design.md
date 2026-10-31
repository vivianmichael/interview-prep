# System Design

## Kafka

??? question "How do you guarantee message ordering in Kafka? `Medium`"
**Key point:** Kafka guarantees ordering only **within a single partition**, never across partitions of a topic.

    **Approaches:**

    1. **Use a message key (preferred).** Messages with the same key always go to the
       same partition (`hash(key) % numPartitions`), so ordering is preserved per key
       (e.g. per `orderId` or `userId`) while the topic still scales across many partitions.
    2. **Use a single partition.** This guarantees total ordering for the topic, but only
       one consumer in a group can read it, so throughput and scalability are limited.
       It's only suitable for low-volume topics.

    **Producer settings to avoid reordering on retries:**

    ```properties
    enable.idempotence=true
    acks=all
    max.in.flight.requests.per.connection=5   # max allowed with idempotence
    ```

    Without idempotence, a failed batch that is retried can land *after* a later batch, which breaks ordering.

    **Consumer side:**

    - Within a consumer group, one partition is read by exactly one consumer, so order is preserved.
    - If you process messages with multiple threads or async calls, you can break the order again.
      Process sequentially per partition, or hand off by key.
    - Changing the partition count later changes `hash(key) % n`, so the same key may move to a different partition.

    **Follow-ups:**

    - What happens to ordering if you add partitions to an existing topic?
    - What is an idempotent producer and how does it prevent duplicates?
    - How do you get ordering and high throughput together?

??? question "One producer, three services consuming the same message: how do you set it up? `Medium`"
**Answer:** Give each service its **own consumer group** (a different `group.id`).

    - Kafka delivers each message **once per consumer group**.
    - Different groups each get **every** message, independently, with their own offsets.
    - Consumers sharing the **same** group split the partitions between them, so each message
      goes to only one of them (load balancing, not broadcasting).

    ```
                              ┌─> group: billing-service       (gets all messages)
    Producer ──> Topic ───────┼─> group: notification-service  (gets all messages)
                              └─> group: analytics-service     (gets all messages)
    ```

    **Config per service:**

    ```properties
    # billing-service
    group.id=billing-service

    # notification-service
    group.id=notification-service

    # analytics-service
    group.id=analytics-service
    ```

    **Scaling each service:** if a service runs multiple instances, all instances use the
    **same** `group.id`. Kafka spreads the partitions across them. Max useful instances per
    service = number of partitions.

    **Other points:**

    - Each group tracks its own offsets, so a slow or down service doesn't affect the others.
    - Use `retention.ms` so a service that was down can catch up when it returns.
    - A new group can replay from the beginning with `auto.offset.reset=earliest`.

    **Follow-ups:**

    - What happens if two services accidentally use the same `group.id`?
    - How does a consumer group rebalance work?
    - Kafka vs RabbitMQ: how does each handle this fan-out pattern?