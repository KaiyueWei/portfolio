---
title: "Ride Sharing"
date: 2026-02-10
weight: 1
tags: ["Go", "TypeScript", "Docker", "Kubernetes", "Microservices", "RabbitMQ", "Next.js", "WebSocket"]
summary: "An Uber-style ride-sharing platform built with Go microservices, event-driven communication via RabbitMQ, and real-time updates with WebSockets."
description: "Building an Uber-style ride-sharing platform with microservices in Go and TypeScript, event-driven communication via RabbitMQ, and real-time updates with WebSockets."
repo: "https://github.com/KaiyueWei/ride-sharing"
cover:
    image: "images/ride-sharing/day0/hybrid.jpeg"
    alt: "Ride sharing architecture diagram"
    hidden: false
---
This project documents my journey building a microservices-based ride-sharing platform (Uber-style), applying what I've learnt in Operating System and Distributed System. Each part covers a concept I explored and the decisions I made along the way — from choosing between monolithic and microservices architectures, to designing event-driven communication with RabbitMQ, to handling errors idiomatically in Go.

<!--more-->

- Repo: https://github.com/KaiyueWei/ride-sharing

## Parts

1. [Day 0: Monolithic vs. Microservices]({{< relref "day0_microservices.md" >}})
2. [Day 1: The Business Problem]({{< relref "day1_business_problem.md" >}})
3. [Day 2: Error Handling in Go]({{< relref "day2_error_handling.md" >}})
4. [Day 3: Scaffolding the Trip Service]({{< relref "day3_trip_service.md" >}})
5. [Day 4: Tracing the GetRoute Call Flow]({{< relref "day4_get_route.md" >}})
