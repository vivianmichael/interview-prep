# Java & Spring Boot

## Core Java

### What is the difference between HashMap and ConcurrentHashMap?

**Answer:**

HashMap isn't thread-safe. ConcurrentHashMap allows concurrent access using
bucket-level synchronization (CAS plus synchronized in Java 8+) instead
of locking the entire map.

It also disallows null keys and values.

<details markdown="1">
<summary>Follow-up questions</summary>

- How does HashMap handle collisions?
- What happens during resize?

</details>

---

## Spring Boot

### How does Spring Boot auto-configuration work?

**Answer:**

`@SpringBootApplication` enables `@EnableAutoConfiguration`, which loads
configuration classes listed in:

`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`

Each configuration is guarded by conditions such as
`@ConditionalOnClass` and `@ConditionalOnMissingBean`.

<details markdown="1">
<summary>Follow-up questions</summary>

- How do you exclude a specific auto-configuration?
- What does `@ConditionalOnMissingBean` let you do?

</details>

---