
# Java & Spring Boot

## Core Java

### What is the difference between HashMap and ConcurrentHashMap?

**Answer:**

HashMap isn't thread-safe. ConcurrentHashMap allows concurrent access using
bucket-level synchronization (CAS plus synchronized in Java 8+) instead
of locking the entire map.

It also disallows null keys and values.

**Follow-up questions:**

- How does HashMap handle collisions?
- What happens during resize?

---

## Spring Boot

### How does Spring Boot auto-configuration work?

**Answer:**

`@SpringBootApplication` enables `@EnableAutoConfiguration`, which loads
configuration classes listed in:

`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`

Each configuration is guarded by conditions such as
`@ConditionalOnClass` and `@ConditionalOnMissingBean`.

**Follow-up questions:**

- How do you exclude a specific auto-configuration?
- What does `@ConditionalOnMissingBean` let you do?
