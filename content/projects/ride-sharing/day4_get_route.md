---
title: "Day 4: Tracing the GetRoute Call Flow"
date: 2026-02-22
draft: false
tags: ["v0.1.0"]
---

Today I traced the full call flow of `GetRoute` — the function that sits at the core of trip previewing. Walking through each layer made the architecture concrete in a way that reading the files in isolation didn't.

## Request Flow

When a client wants to preview a trip, the request passes through three distinct layers before hitting the external routing API:

```
Client
  POST /trip/preview  {userID, pickup, destination}
  │
  ▼
api-gateway/http.go : handleTripPreview
  - validates userID is present
  - re-serializes body as JSON
  - POST http://trip-service:8083/preview  ──────────────────────┐
  - wraps response in contracts.APIResponse{Data: ...}           │
  - returns 201                                                   │
                                                                  ▼
                              trip-service/infrastructure/http/http.go : HandleTripPreview
                                - decodes previewTripRequest {userID, pickup, destination}
                                - calls s.Service.GetRoute(ctx, pickup, destination)
                                │
                                ▼
                              trip-service/service/service.go : GetRoute
                                - builds OSRM URL with lon/lat coords
                                - GET http://router.project-osrm.org/route/v1/driving/...
                                - unmarshals JSON into OsrmApiResponse
                                - returns routes {distance, duration, geometry{coordinates}}
                                │
                                ▼
                              back to HandleTripPreview
                                - writeJSON 200 with OsrmApiResponse
```

## Layer Responsibilities

Each layer has a clear, single responsibility:

| Layer | File | Responsibility |
|---|---|---|
| Edge | `api-gateway/http.go` | Input validation, forwarding, unified response envelope |
| HTTP adapter | `trip-service/infrastructure/http/http.go` | Translate HTTP → domain call, serialize result |
| Service | `trip-service/service/service.go` | Business logic, calls external OSRM API |
| Domain | `trip-service/domain/trip.go` | Defines the `TripService` interface |
| Types | `shared/types/types.go` | Shared data shapes (`OsrmApiResponse`, `Coordinate`) |

## What I Learned

- **The gateway and the service have different jobs.** The API gateway handles the user-facing contract (validation, envelope), while the trip service handles the domain logic. Keeping these separate means either can change without breaking the other.
- **The HTTP adapter is just translation.** `HandleTripPreview` doesn't make decisions — it decodes a request, calls the service, and serializes the result. All actual logic lives in `GetRoute`.
- **External calls belong in the service layer.** The OSRM API call lives in `service.go`, not in the HTTP handler. This keeps the infrastructure layer thin and makes the service testable with a mock HTTP client.
