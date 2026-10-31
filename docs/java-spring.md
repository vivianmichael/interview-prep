# Java & Spring Boot

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

## Spring Boot

### How does Spring Boot auto-configuration work?

Spring Boot auto-configuration configures application components based on the dependencies available on the classpath, the beans already defined by the application, and configuration properties.

The goal is to reduce the amount of manual configuration developers need to write.

**Mental model:**

Imagine you add Spring Data JPA and a database driver to your project. Spring Boot detects the relevant classes and, when the required conditions are met, configures infrastructure such as a `DataSource`, an `EntityManagerFactory`, and a transaction manager.

**How does it work?**

`@SpringBootApplication` combines three annotations:

```java
@SpringBootApplication
public class PaymentApplication {

    public static void main(String[] args) {
        SpringApplication.run(PaymentApplication.class, args);
    }
}
```

It is effectively equivalent to:

```java
@Configuration
@EnableAutoConfiguration
@ComponentScan
public class PaymentApplication {
}
```

The key annotation is `@EnableAutoConfiguration`, which enables Spring Boot's auto-configuration mechanism.

In modern Spring Boot versions, auto-configuration candidates are listed in:

```text
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

Auto-configuration classes use conditional annotations to decide whether their beans should be registered.

| Annotation | Purpose |
|---|---|
| `@ConditionalOnClass` | Applies configuration when a particular class is present on the classpath |
| `@ConditionalOnMissingBean` | Applies configuration when a matching bean has not already been defined |
| `@ConditionalOnBean` | Applies configuration when a matching bean exists |
| `@ConditionalOnProperty` | Applies configuration when a property matches the specified condition |
| `@ConditionalOnMissingClass` | Applies configuration when a particular class is absent |

**Code example: Creating your own auto-configuration**

Imagine you are building a reusable payment-processing library for other Spring Boot applications. You want to provide a default `PaymentService`, but allow an application to supply its own implementation.

```java
@AutoConfiguration
@ConditionalOnClass(PaymentService.class)
public class PaymentAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public PaymentService paymentService() {
        return new PaymentService();
    }
}
```

Register the configuration class in:

```text
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

Add this fully qualified class name as a line in that file:

```text
com.example.payment.PaymentAutoConfiguration
```

When an application includes the library, Spring Boot can discover the auto-configuration. If the application has already defined a matching `PaymentService` bean, the conditional default bean is not created.

**Real-world production example: Shared payment library**

Consider a company with several payment microservices:

- Payment authorization
- Settlement
- Refunds
- Merchant management

They all use a shared internal payment library. The library can provide auto-configuration for common components such as payment clients, authentication components, default timeout settings, retry configuration, and metrics.

Each service can use the defaults or provide its own beans and properties where its requirements differ.

**Senior-level considerations:**

- Auto-configuration is conditional; it does not force an application to use every default.
- `@ConditionalOnMissingBean` is a common way to provide a default that applications can override.
- Configuration ordering matters when one auto-configuration depends on beans or conditions from another.
- `@ConditionalOnProperty` is useful for optional features.
- Spring Boot's condition evaluation report and the Actuator `conditions` endpoint can help diagnose why an auto-configuration matched or did not match.

<details markdown="1">
<summary>Follow-up questions</summary>

- What is the difference between `@Configuration` and `@AutoConfiguration`?
- What does `@EnableAutoConfiguration` do?
- What is the difference between `@ComponentScan` and auto-configuration?
- How do you exclude a specific auto-configuration?
- What does `@ConditionalOnMissingBean` let you do?
- What happens if two auto-configurations define the same bean?
- How does Spring Boot discover auto-configuration classes?
- How would you build a reusable Spring Boot starter?
- How would you debug why an expected bean was not created?

</details>

---
