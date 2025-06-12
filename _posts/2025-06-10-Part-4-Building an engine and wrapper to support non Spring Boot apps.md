---
title: "Part 4: Building an engine and wrapper to support non Spring Boot apps"
date: 2025-06-10
categories: [Java, AWS, Resilience]
tags: [Java, Spring Boot, AWS Lambda, Distributed Systems, DynamoDB, Circuit Breaker]
description: Building an engine to support non spring boot apps
---

In [Part 1](/clinton-blogs/Why-Java's-In-Memory-Circuit-Breakers-Fail-in-Distributed-Cloud-Systems/), I introduced the idea behind the `cloud-circuitbreaker` — a distributed, annotation-driven circuit breaker backed by DynamoDB.  
In [Part 2](/clinton-blogs/Part-2-Cloud-Native-Circuit-Breakers,-How-It-Works-Under-the-Hood/), we walked through how failure detection, TTLs, and state tracking work.  
In [Part 3](/clinton-blogs/Part-3-Spring-Boot-&-Lambda-Integration-with-Cloud-Circuit-Breaker/), we walked through how to integrate cloud circuit breaker with Lambda and Spring Boot.

Not all apps are Spring Boot right? Since this works only with annotations, I wanted something more. 

So, I built something new.

---

## 💡 Introducing `cloud-circuitbreaker-engine`

This is a cloud-native, distributed circuit breaker written in Java, backed by **DynamoDB**, and designed to be:

- **Stateless on the instance**  
- **Shared in behavior**  
- **Easy to use** with or without `@CloudCircuitBreaker` annotation  
- **Framework-agnostic**, with support for Spring Boot, plain Java, and AWS Lambda

👉 [View the source on GitHub](https://github.com/clinton1719/cloud-circuitbreaker/blob/main/cloud-circuitbreaker-core/src/main/java/com/cloudcb/core/CloudCircuitBreakerEngine.java)

---

## ✨ Example Usage

Here’s how you'd use it inside your service:

### 2. Initialize the Engine

```java
CloudCircuitBreakerEngine engine = CloudCircuitBreakerEngine
    .builder()
    .withDynamoClient(AmazonDynamoDBClientBuilder.defaultClient())
    .withBreakerTable("circuit-breaker-table")
    .withTimeoutSeconds(10)
    .withFailureThreshold(3)
    .withTTLSeconds(30)
    .build();
```
You can customize all circuit breaker parameters: timeouts, thresholds, TTL, and more.

---

### 3. Wrap Your Function

Now wrap your function using the engine:

```java
String result = engine.execute(
    "user-service-call",
    () -> externalService.callUserService(userId),
    () -> cachedUserService.readFromCache(userId)
);
```
This works just like the annotation — but under your full control.

---

## 💡 Why Use This Over Annotations?

If you're in a:

- CLI app  
- Lightweight microservice with manual wiring  
- Kotlin/Scala or non-Spring ecosystem  
- Testing environment or framework plugin  

…then this gives you the same power as `@CloudCircuitBreaker`, but without dependency on Spring or reflection.

---

## 🧪 Testing It

Here’s a full testable example:

```java
CloudCircuitBreakerEngine engine = CloudCircuitBreakerEngine
    .builder()
    .withDynamoClient(dynamoClient)
    .withBreakerTable("cb-table")
    .build();

String result = engine.execute(
    "external-fetch",
    () -> unstableAPI.getUser("123"),
    () -> "fallback-user"
);
```

Failures are automatically recorded in DynamoDB. If threshold is breached, fallback is called until TTL expires.

---

## 🏁 That’s a Wrap

That concludes the series on building and using a **distributed circuit breaker for cloud-native apps**.

Whether you're building resilient APIs in Spring Boot, handling retries in Lambdas, or writing lightweight CLI tools, `cloud-circuitbreaker` gives you:

- A shared view of failures across all instances  
- TTL-based breaker state stored in DynamoDB  
- A clean, ergonomic API — with or without Spring

Thanks for following along this far!  
This was a problem I personally faced at scale — and building this library made it dramatically easier to handle edge cases, retries, and chaos in production.

---

## 📬 Got Ideas?

I'm planning Redis support, metrics emitters, and maybe a dashboard.

Open an [issue](https://github.com/clinton1719/cloud-circuitbreaker/issues) or [connect on LinkedIn](https://www.linkedin.com/in/clinton-fernandes-45932915a/) — I’d love to hear how you’d like to use this.

Until next time,  
**Happy breaking (responsibly).**


