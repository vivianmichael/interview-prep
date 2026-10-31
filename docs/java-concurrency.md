# Java Concurrency

## Core Java

### What is the difference between HashMap and ConcurrentHashMap?

`HashMap` and `ConcurrentHashMap` both store data as key-value pairs, but they differ in how they handle concurrent access.

**HashMap**

- Not thread-safe.
- Concurrent modifications by multiple threads can cause race conditions and lost updates.
- Allows one `null` key and multiple `null` values.
- Suitable when the map is confined to one thread or protected externally by appropriate synchronization.

**ConcurrentHashMap**

- Supports safe concurrent access and updates.
- Uses a combination of CAS (Compare-And-Swap), synchronization on individual bins, and other internal mechanisms in Java 8+.
- Does not allow `null` keys or values.
- Provides atomic per-key operations such as `putIfAbsent()`, `computeIfAbsent()`, `compute()`, and `merge()`.

**Mental model:**

Think of `HashMap` as a shared notebook with no rules. If multiple people write in it simultaneously, they can interfere with each other.

`ConcurrentHashMap` coordinates access to the relevant part of the map. Threads working on different bins can often make progress concurrently, while conflicting updates are coordinated.

**Code example:**

```java
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class CacheExample {

    // Not safe for concurrent modifications without external synchronization.
    private final Map<String, Integer> unsafeCache = new HashMap<>();

    // Supports concurrent access.
    private final Map<String, Integer> safeCache =
            new ConcurrentHashMap<>();

    public void updateCache(String key, Integer value) {
        safeCache.put(key, value);
    }

    public Integer getFromCache(String key) {
        return safeCache.get(key);
    }
}
```

**Real-world production example: Payment processing**

Imagine a payment platform processing many transactions concurrently. It maintains a cache of merchant configuration, keyed by merchant ID:

```java
private final Map<String, MerchantConfig> merchantCache =
        new ConcurrentHashMap<>();

public MerchantConfig getMerchantConfig(String merchantId) {
    return merchantCache.computeIfAbsent(
        merchantId,
        id -> merchantRepository.findByMerchantId(id)
    );
}
```

Multiple request threads can safely access the cache. `computeIfAbsent()` coordinates computation for a missing key so concurrent callers do not independently install competing values for that key.

In production, be careful about doing slow database or network calls inside a map's mapping function: the computation can affect other updates to the relevant bin. Also consider cache expiration, eviction, stale data, and memory limits.

**Important distinction:**

Thread-safe individual map operations do not make a multi-step business operation atomic.

For example, this is not a safe atomic increment:

```java
map.put(key, map.get(key) + 1);
```

Use an atomic per-key operation instead:

```java
map.merge(key, 1, Integer::sum);
```

<details markdown="1">
<summary>Follow-up questions</summary>

- How does HashMap handle collisions?
- What happens during HashMap resize?
- What is the difference between HashMap and Hashtable?
- How does ConcurrentHashMap work internally in Java 8+?
- What is CAS (Compare-And-Swap)?
- Why does ConcurrentHashMap not allow null keys or values?
- Is ConcurrentHashMap completely lock-free?
- Is `computeIfAbsent()` thread-safe?
- Does ConcurrentHashMap make compound operations thread-safe?

</details>

---
