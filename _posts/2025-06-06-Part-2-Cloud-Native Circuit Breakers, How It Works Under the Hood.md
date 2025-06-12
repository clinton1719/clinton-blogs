---
title: "Part 2: Cloud-Native Circuit Breakers: How It Works Under the Hood"
date: 2025-06-06
categories: [Java, AWS, Resilience]
tags: [java, aws, circuit-breaker, dynamodb, annotations, resilience]
description: "Digging into the internals of cloud-circuitbreaker — how failure states are tracked, shared, and enforced across instances using DynamoDB and annotations."
---

In [Part 1](/clinton-blogs/Why-Java's-In-Memory-Circuit-Breakers-Fail-in-Distributed-Cloud-Systems/), I shared the motivation behind creating a distributed circuit breaker for cloud-native systems — something memoryless, stateless, and resilient enough to survive autoscaling chaos.

Now let’s look under the hood.

---

## 🧠 Design Goals Recap

I needed something that works seamlessly in distributed environments like **AWS Lambda**, **ECS**, or **Kubernetes** — where in-memory state is basically meaningless.

So the solution had to be:

- 🔁 **Shared** — one circuit state for all instances
- ☁️ **Stateless-compatible** — doesn't care where it runs
- ⚙️ **Configurable** — env vars or YAML, your call
- 🧩 **Pluggable** — default is DynamoDB, others possible later

---

## 💾 Why DynamoDB?

DynamoDB is serverless, fast, and battle-tested at scale. But most importantly:

- It supports **TTL (Time To Live)** out-of-the-box
- You can do **atomic updates and conditionals** per item
- It's a good fit for **key-value state tracking**, with minimal latency

We use one record per circuit breaker function, keyed by name (e.g., `getUserData`), which holds:

| Key (PK)      | Field               | Description                            |
|---------------|---------------------|----------------------------------------|
| `getUserData` | `failureCount`      | Number of failures observed            |
|               | `lastFailureTime`   | Timestamp of last failure              |
|               | `state`             | `CLOSED`, `OPEN`, or `HALF_OPEN`       |
|               | `ttl`               | Auto-expiry to reset breaker state     |

DynamoDB's TTL feature automatically cleans up old breaker states if left idle — meaning you never have to worry about stale entries.

---

## 🔄 Failure Tracking Logic

Here’s what happens step-by-step when a method fails:

1. The method is annotated with `@CloudCircuitBreaker`.
2. On exception, the handler increments the `failureCount` in DynamoDB **atomically**.
3. If the count exceeds threshold (say, 5), the breaker state flips to `OPEN`.
4. While `OPEN`, any call to that method will short-circuit — it skips execution and calls the fallback.
5. After a timeout (e.g., 60 seconds), it transitions to `HALF_OPEN` — allows one test request.
6. If the test passes → reset breaker (`CLOSED`).  
   If it fails → back to `OPEN`.

---

## 🧬 Configuration

You can configure each circuit via annotations (per method).

```java
@CloudCircuitBreaker(
    function = "getUserData",
    fallback = "fallbackFunction"
)
public Response getUserData(String userId) {
    return userService.fetchFromUpstream(userId);
}

public Response fallbackFunction(String userId) {
    return userService.fetchFromCache(userId);
}
```
Configure via YAML:
```yaml
cloudcb:
    failureThreshold: 5
    timeoutSeconds: 60
```

This lets you adapt quickly across environments — say you want a longer timeout in staging but more aggressive breakage in prod.

---

## 🗺️ Architecture Diagram

Here’s how it all fits together at runtime:

```plaintext
                ┌────────────────────────┐
                │  Java Method Call      │
                └─────────┬──────────────┘
                          │
                  Checks Annotation
                          │
                          ▼
            ┌────────────────────────────┐
            │ Circuit State Lookup (DDB)│
            └─────────┬──────────────────┘
                      │
        ┌─────────────▼──────────────┐
        │  Is State OPEN or TTL valid? │─────▶ Yes → Skip execution → Fallback
        └─────────────┬──────────────┘
                      │ No
                      ▼
        ┌────────────────────────────┐
        │   Execute Original Method  │
        └─────────┬──────────────────┘
                  │
         ┌────────▼────────┐
         │ Success?        │
         └──────┬──────────┘
                │
      ┌─────────▼─────────────┐
      │  Reset failure count  │
      └───────────────────────┘
```

This happens transparently via a runtime proxy that wraps all annotated methods — no boilerplate for developers.

---

## 🧪 Safe for Stateless Systems

What makes this durable:

- **TTL-managed keys** mean no cleanup jobs  
- **Atomic counters** prevent race conditions  
- **Centralized breaker state** shared across all containers/functions  

It doesn't matter if one instance dies, restarts, or scales out — the breaker state lives in DynamoDB and **survives independently** of your infrastructure.

---

## 🧰 Pluggable Design (Coming Soon)

While DynamoDB is default, the design allows swapping in Redis, S3, or even custom backends.

The circuit state store is defined behind an interface like:

```java
/**
 * Interface for persisting and retrieving the state of a circuit breaker.
 * <p>
 * Implementations of this interface are responsible for providing a durable store
 * (e.g., DynamoDB, Redis, in-memory) for circuit breaker states identified by a unique key.
 * </p>
 * <p>
 * This allows for distributed or clustered systems to share circuit breaker status across instances.
 * </p>
 *
 * @author Clinton Fernandes
 */
public interface CircuitBreakerStore {

    /**
     * Retrieves the current state of the circuit breaker for the given key.
     *
     * @param key A unique identifier representing a specific circuit breaker (e.g., service.method).
     * @return The current {@link CircuitBreakerState}, or {@code null} if no state exists.
     */
    CircuitBreakerState getState(String key);

    /**
     * Persists the circuit breaker state for the given key.
     *
     * @param key   A unique identifier representing a specific circuit breaker.
     * @param state The {@link CircuitBreakerState} to persist.
     */
    void saveState(String key, CircuitBreakerState state);

    /**
     * Resets (removes or reinitializes) the circuit breaker state for the given key.
     * This typically moves the circuit breaker back to the initial (closed) state.
     *
     * @param key A unique identifier representing a specific circuit breaker.
     */
    void reset(String key);
}
```

Swapping implementations is just a Spring bean or Lambda module away.

---

## 🔗 Try It Out

Ready to use it? Clone or check it out here:  
👉 [clinton1719/cloud-circuitbreaker](https://github.com/clinton1719/cloud-circuitbreaker)

---

## 👋 Up Next: Spring Boot Autoconfig & Lambda Support

In **Part 3**, I’ll walk through:

- Autowiring the annotation into Spring Boot  
- Supporting AWS Lambda and native Java functions  
- Packaging best practices for Maven-distributed libraries  

Follow along or subscribe to the blog feed. Feedback, ideas, PRs — always welcome.

Stay resilient 💪

