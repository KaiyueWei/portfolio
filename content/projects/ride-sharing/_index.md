---
title: "Ride Sharing"
date: 2026-02-10
tags: ["Go", "TypeScript", "Docker", "Kubernetes", "Microservices", "RabbitMQ", "Next.js", "WebSocket"]
summary: "A microservices-based ride-sharing platform built with Go, Docker, and Kubernetes, using an event-driven architecture with RabbitMQ and WebSockets for real-time updates"
---
This project documents my journey building a microservices-based ride-sharing platform (Uber-style), applying what I've learnt in Operating System and Distributed System. Each part covers a concept I explored and the decisions I made along the way — from choosing between monolithic and microservices architectures, to designing event-driven communication with RabbitMQ, to handling errors idiomatically in Go.

<!--more-->

- Repo: https://github.com/KaiyueWei/ride-sharing

## Parts

1. [Day 0: Monolithic vs. Microservices]({{< relref "day0_microservices.md" >}})
2. [Day 1: The Business Problem]({{< relref "day1_business_problem.md" >}})
3. [Day 2: Error Handling in Go]({{< relref "day2_error_handling.md" >}})
4. [Day 3: Scaffolding the Trip Service]({{< relref "day3_trip_service.md" >}})
