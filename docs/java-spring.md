# Java & Spring Boot

## Core Java

??? question "What is the difference between HashMap and ConcurrentHashMap? `Medium`"
    HashMap isn't thread-safe. ConcurrentHashMap allows concurrent access by
    locking at the bucket/node level (CAS plus synchronized in Java 8+) instead
    of locking the whole map, and it disallows null keys and values.

    **Follow-ups:**

    - How does HashMap handle collisions?
    - What happens during resize?

## Spring Boot

??? question "How does Spring Boot auto-configuration work? `Medium`"
`   @SpringBootApplication` enables `@EnableAutoConfiguration`, which loads
    configuration classes listed in
    `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`.
    Each is guarded by conditions such as `@ConditionalOnClass` and
   `@ConditionalOnMissingBean`.

    **Follow-ups:**

    - How do you exclude a specific auto-configuration?
    - What does `@ConditionalOnMissingBean` let you do?