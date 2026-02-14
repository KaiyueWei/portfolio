---
title: "Day 1: The Business Problem"
date: 2026-02-10
draft: false
---

After exploring the theory behind microservices in Day 0, it's time to define what we're actually building and why.

## What Are We Building?

A ride-sharing platform — think Uber or Lyft. At its core, the problem is simple: **a rider wants to go somewhere, and a driver can take them there**. But the real complexity comes from making this work in real time.

The key user flows are:
- **Rider**: pick a location on a map, preview the trip/fare, request a ride, get matched with a driver, and pay
- **Driver**: register with a car package, see incoming trip requests, accept or decline, and navigate to the rider

What makes this interesting from a systems perspective is that nearly every interaction is **time-sensitive and event-driven**. When a rider requests a trip, we need to notify available drivers *immediately*. When a driver accepts, the rider needs to know *right away*. Polling or REST request-response alone won't cut it.

## Why Microservices?

This project is a natural fit for microservices because the domain breaks cleanly into independent responsibilities:

- **Trip Service** — manages the trip lifecycle (creation, driver assignment, completion)
- **Driver Service** — handles driver registration, availability, and location tracking
- **Payment Service** — processes payments via Stripe
- **API Gateway** — the single entry point that bridges HTTP/WebSocket clients to the backend

Each of these can be developed, deployed, and scaled independently. The Trip Service doesn't need to know how payments work — it just emits a `trip.event.completed` event, and the Payment Service reacts to it.

This is the hybrid approach from Day 0 in action: the API Gateway handles synchronous HTTP requests from the frontend, while backend services communicate asynchronously through RabbitMQ.

## Choosing the Communication Pattern

From what I covered in Day 0, the hybrid approach made the most sense here:

1. **Client-to-Gateway**: Synchronous HTTP for things like creating a trip or fetching driver info, plus **WebSockets** for real-time updates (driver location, trip status changes)
2. **Service-to-Service**: Asynchronous messaging via **RabbitMQ** using a topic exchange

I settled on a routing key convention to keep things organized:

| Pattern | Description |
|---------|-------------|
| `trip.event.*` | Trip lifecycle events (created, driver_assigned, no_drivers_found) |
| `driver.cmd.*` | Driver commands (trip_request, trip_accept, trip_decline, location) |
| `payment.event.*` | Payment events (session_created, success, failed, cancelled) |
| `payment.cmd.*` | Payment commands (create_session) |

The distinction between `event` and `cmd` is intentional — **events** describe something that happened (past tense, broadcast), while **commands** request an action (imperative, targeted). This keeps the message flow predictable.

## Structuring the Services

One thing I wanted to get right early was the internal structure of each service. I went with **Clean Architecture**, where each service follows this layout:

```
services/<name>-service/
├── cmd/                        # Entry point
├── internal/
│   ├── domain/                # Interfaces & business models
│   ├── service/               # Business logic
│   └── infrastructure/        # Adapters (events, gRPC, repository)
└── pkg/types/                 # Public types
```

The key idea: **domain logic has zero dependencies on infrastructure**. The domain layer defines interfaces (e.g., `TripRepository`), and the infrastructure layer provides concrete implementations (e.g., a MongoDB adapter). This makes it easy to swap out a database or message broker without touching business logic.

I also built a small scaffolding tool (`go run tools/create_service.go -name <service>`) so spinning up a new service with this structure takes seconds instead of manually creating directories and boilerplate.

## The Trip Scheduling Flow

Here's the full flow I designed for trip scheduling — from the rider requesting a ride to a driver being assigned:

[![Trip Scheduling Flow](https://mermaid.ink/img/pako:eNqNVt9v2jAQ_lcsP21qGvGjZSEPlSpaTX1YxWDVpAmpMvZBIkicOQ6UVf3fd4mdEgcKzQOK47v7vjt_d-aVcimAhnSW5vC3gJTDXcyWiiWzlOCTMaVjHmcs1eQpB3X49Xb88J1p2LLd4d4vFWdTUJuYw-HmnYo3oM5sH34fs10Cqf7Qb6oRFL-bnZLz5c3NnmRIRgrwteJGJmXOuTa2eyP0aFAPyXIyHtWO5YaxN7-PEoNJpNrM1nOSCw3Y_QuPWLq0nBvWl4h30fIos_Bhg5n6vMIVDTgVLyNN5II4LO9L65A8wrbyJimAyAkjwlbSBHBwENisQ2vl80T4pfexapbGRW1RHckkYaloILMNi9dsvgYXtHUQv2E-lXwF-hCccQ5ZE3sNiwZ0A_O2sjSw6oPTbB_nWbT3TB3dGEiykGrLlABBtCQTNp_H-sdP8sUwK3EmkGcS2-lnAQV8rUtwVCchGSvJIc8tJ2KoMGxDjxSZKIVapZZrpovc9_2j4nF7whGPifvM8jxepp8XkW2SzAQmOVKMZXoyF6_Nwq5bunetkLxp2HfIUQR8JQts5Bqz9DJGx3K1ZtZdHAVpCc9mZStkc3s-0WZtTFukOsGnB5QeE7u6PM4gKScQsozktmF_TmuN1qidUHZJa6q9V04m2RowlrV1SnbQc5GUq33YacFL_R2faHtPzxXt0ZN1O-6iXbRW1Zu4HxeiqjTJivk6ziPTcp-U1aXD-NPgZ46a21KL061Qj6kzc38_facarzCyv1tOdGgVk7OUpCipOSxjZEI9moBKWCzwKn8tQ8yojiCBGQ3xVTC1muEV_4Z2rNByuks5DbUqwKNKFsuIhgu2znFlZo79C1Cb4L36R8rmkoav9IWGvW_-1XVn0O_1-kE3GAyHgUd3-Lnb8fu9frc_xKfbvQ6CN4_-qyJ0_KDX7Q86QTDoDAfD66ve23_1IPGQ?type=png)](https://mermaid.live/edit#pako:eNqNVt9v2jAQ_lcsP21qGvGjZSEPlSpaTX1YxWDVpAmpMvZBIkicOQ6UVf3fd4mdEgcKzQOK47v7vjt_d-aVcimAhnSW5vC3gJTDXcyWiiWzlOCTMaVjHmcs1eQpB3X49Xb88J1p2LJd4d4vFWdTUJuYw-HmnYo3oM5sH34fs10Cqf7Qb6oRFL-bnZLz5c3NnmRIRgrwteJGJmXOuTa2eyP0aFAPyXIyHtWO5YaxN7-PEoNJpNrM1nOSCw3Y_QuPWLq0nBvWl4h30fIos_Bhg5n6vMIVDTgVLyNN5II4LO9L65A8wrbyJimAyAkjwlbSBHBwENisQ2vl80T4pfexapbGRW1RHckkYaloILMNi9dsvgYXtHUQv2E-lXwF-hCccQ5ZE3sNiwZ0A_O2sjSw6oPTbB_nWbT3TB3dGEiykGrLlABBtCQTNp_H-sdP8sUwK3EmkGcS2-lnAQV8rUtwVCchGSvJIc8tJ2KoMGxDjxSZKIVapZZrpovc9_2j4nF7whGPifvM8jxepp8XkW2SzAQmOVKMZXoyF6_Nwq5bunetkLxp2HfIUQR8JQts5Bqz9DJGx3K1ZtZdHAVpCc9mZStkc3s-0WZtTFukOsGnB5QeE7u6PM4gKScQsozktmF_TmuN1qidUHZJa6q9V04m2RowlrV1SnbQc5GUq33YacFL_R2faHtPzxXt0ZN1O-6iXbRW1Zu4HxeiqjTJivk6ziPTcp-U1aXD-NPgZ46a21KL061Qj6kzc38_facarzCyv1tOdGgVk7OUpCipOSxjZEI9moBKWCzwKn8tQ8yojiCBGQ3xVTC1muEV_4Z2rNByuks5DbUqwKNKFsuIhgu2znFlZo79C1Cb4L36R8rmkoav9IWGvW_-1XVn0O_1-kE3GAyHgUd3-Lnb8fu9frc_xKfbvQ6CN4_-qyJ0_KDX7Q86QTDoDAfD66ve23_1IPGQ)

## What I Learned

- **Start with the domain, not the tech.** It was tempting to jump straight into setting up RabbitMQ and Kubernetes, but defining the business flows first made every technical decision easier — the architecture fell out naturally from the problem.
- **Events vs. commands is a useful mental model.** Separating "something happened" from "please do something" made the message flow much cleaner and easier to reason about.
- **Clean Architecture pays off early.** Even in a small project, having clear boundaries between domain logic and infrastructure made it easy to iterate without breaking things.
