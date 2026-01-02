---
title: "Data Engineering (Part 1): Container Networking"
date: 2026-01-01
tags: ["Docker", "Docker Compose", "Networking", "Postgres"]
summary: "Practical rules for connecting services between containers and the host."
---

When you containerize a pipeline, the most common failure isn’t your code — it’s usually how services address each other.
<!--more-->

## The rule
Inside a container, `localhost` means *that container*, not your laptop and not another container.

## In Docker Compose: use service names
If Postgres is started as a Compose service (for example `pgdatabase`), other containers in the same Compose project should connect using the service name:

- Host: `pgdatabase`
- Port: `5432`

Example connection string:

```text
postgresql://root:root@pgdatabase:5432/ny_taxi
```

## From your laptop (host)
From your host OS, you connect via the published port:

- Host: `localhost`
- Port: `5432`

Example:

```bash
pgcli -h localhost -p 5432 -U root -d ny_taxi
```

## Notes
- Docker Desktop (Mac/Windows): `host.docker.internal` is commonly used for reaching services running on your host from inside a container.
- For Postgres, make sure your volume mount path matches the image/version you’re using.
