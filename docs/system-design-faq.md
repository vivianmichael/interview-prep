# System Design FAQs

## Kafka FAQs

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

Give each service its **own consumer group** (a different `group.id`).

- Kafka delivers each message to every subscribed consumer group.
- Different consumer groups receive messages independently and maintain their own offsets.
- Consumers sharing the same group split partitions among themselves for scalability.

**Architecture:**

```text
                         ┌──> Email Service
                         │    group.id=email-group
                         │
Producer ──> orders-topic├──> Inventory Service
                         │    group.id=inventory-group
                         │
                         └──> Analytics Service
                              group.id=analytics-group
```

**Producer Example:**

```java
@Service
public class OrderProducer {

    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public OrderProducer(KafkaTemplate<String, OrderEvent> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public void publish(OrderEvent event) {
        kafkaTemplate.send(
            "orders-topic",
            event.getOrderId(),
            event
        );
    }
}
```

**Configuration per service:**

```properties
# Email Service
spring.kafka.consumer.group-id=email-group

# Inventory Service
spring.kafka.consumer.group-id=inventory-group

# Analytics Service
spring.kafka.consumer.group-id=analytics-group
```

**Consumer Example:**

```java
@KafkaListener(
    topics = "orders-topic",
    groupId = "email-group"
)
public void consume(OrderEvent event) {
    System.out.println(
        "Email Service received: " + event.getOrderId()
    );
}
```

**Message Flow:**

```text
OrderCreated Event
        |
        v
   orders-topic
        |
        +--> email-group
        |      -> Email Service
        |
        +--> inventory-group
        |      -> Inventory Service
        |
        +--> analytics-group
               -> Analytics Service
```

All three services receive and process the same message because they belong to different consumer groups.

**Scaling each service:**

If a service runs multiple instances, all instances use the same `group.id`.

Kafka distributes partitions across those instances.

```text
orders-topic (3 partitions)

email-group
 ├── email-instance-1
 ├── email-instance-2
 └── email-instance-3
```

Maximum useful active consumers per group = number of partitions.

**Other important points:**

- Each consumer group tracks its own offsets independently.
- A slow or unavailable service does not block other consumer groups.
- Use appropriate `retention.ms` so services can catch up after downtime.
- A new consumer group can replay historical events using `auto.offset.reset=earliest`.
- If all consumers use the same `group.id`, Kafka load-balances messages instead of sending every message to every service.
- The number of partitions determines the maximum parallelism within a consumer group.

**Interview Sound Bite:**

> To have three services consume the same Kafka message, I create one topic and assign each service a different consumer group ID. Kafka guarantees that every consumer group receives every message independently, while consumers within the same group share the workload.

<details markdown="1">
<summary>Follow-up questions</summary>

- What happens if two services accidentally use the same `group.id`?
- How does a consumer group rebalance work?

</details>

---
