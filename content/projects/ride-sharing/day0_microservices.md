---
title: "Day 0: Monolithic vs Microservices"
date: 2026-02-09
draft: false
---

## References

- [Strangler Fig](https://martinfowler.com/bliki/StranglerFigApplication.html)
- [Microservice Architecture pattern](https://microservices.io/patterns/microservices.html)

## Service

- Micro
- Autonomous
- Loosely coupled
- Context bound

**Coupling** — describes how much one module relies on other modules.

**Cohesion** — how closely the responsibilities within a single module or component are related.

## Communication Styles

### Synchronous Request-Response (blocking)

**HTTP/REST (GraphQL or standard REST APIs)**
- a ket concept in REST is a resource, which represents a business domain(User, Post, Trip)
- REST uses the HTTP verbs (GET, POST, PUT, DELETE) for manipulating resouces, which are referenced using a URL
- No strict contract; loosely defined via documentation
- text-based (JSON, XML)

Excellent for public-facing APIs and scenerios where interoperabilty and ease of debugging are prioritized.

**gRPC/RPC (gRPC supports async communication with bi-directional streams)**
- High Performance Remote Procedure Call (RPC), framework that uses protocol buffers (a binary format)
- Define services and messages schemas in a `.proto` file, and code is generated for both the server and the client.
- Supports simple request/response and streaming RPCs, where servers can send a stream of messages to clients or vice versa.

Well-suited for high-performance, interbal service-to-service communication where strong contracts and efficiency are critical.

**Issues with Synchronous messaging pattern**
If sync communication leads to request/message being dependent on service health and response.
How can we fix it?

![Issue with Sync](/images/ride-sharing/day0/sync_issue.jpeg)

### Asynchronous (non-blocking)

Instead of direct communication, let's use a highly available middleman where we can send messages and it handles how to deliver them, to the right destination.

![Async](/images/ride-sharing/day0/async.jpeg)

- Message brokers (event-driven / RabbitMQ / Kafka)
- Pub/Sub

**Advantages**
Loosely coupled
 - Service A and Service B do not directly depend on each other's availability

Scalability
- Because messages queue up in the broker, Service B can scale up to handle bursts of messages

Reliability & fault-tolerant
- If Service B goes down temporarily, the broker holds the messages until B is back up

Durability
- Properly configured brokers(e.g., RabbitMQ, Kafka) support message persistence and acknowledgements to prevent message loss.

### Hybrid Approach

API Gateway with asynchronous backends:

- Gateway handles synchronous requests from clients while delegating to asynchronous backend services
- Service-to-service communication is async


![Hybrid Approach](/images/ride-sharing/day0/hybrid.jpeg)
