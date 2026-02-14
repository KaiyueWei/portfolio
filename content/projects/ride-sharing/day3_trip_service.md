---
title: "Day 3: Scaffolding the Trip Service"
date: 2026-02-12
draft: false
tags: ["v0.1.0"]
---

With the architecture designed in Day 1 and Go error patterns from Day 2, it's time to build the first real service. The Trip Service is the heart of the platform — it owns the trip lifecycle. I started by scaffolding it using the generator tool:

```bash
go run tools/create_service.go -name trip
```

This gives us the Clean Architecture layout from Day 1:

```
services/trip-service/
├── cmd/                        # Application entry point
├── internal/
│   ├── domain/                # Business domain models and interfaces
│   │   └── trip.go
│   ├── service/               # Business logic implementation
│   │   └── service.go
│   └── infrastructure/        # External dependency implementations
│       ├── events/            # Event handling (RabbitMQ)
│       ├── grpc/              # gRPC server handlers
│       └── repository/        # Data persistence
├── pkg/
│   └── types/                 # Shared types and models
└── README.md
```

## Domain Layer

The domain layer defines the core models and interfaces — no implementation details, no imports from infrastructure packages.

```go
type TripModel struct {
    ID       primitive.ObjectID
    UserID   string
    Status   string
    RideFare RideFareModel
}

type RideFareModel struct {
    PackageType string // sedan, SUV, van, luxury
    TotalPrice  int    // price in cents
}
```

And the contracts that the rest of the application depends on:

```go
type TripRepository interface {
    CreateTrip(trip TripModel) (*TripModel, error)
}

type TripService interface {
    CreateTrip(fare RideFareModel) (*TripModel, error)
}
```

This is the key principle of Clean Architecture at work: everything depends *inward* on these interfaces. The service layer implements `TripService`, the infrastructure layer implements `TripRepository`, but neither knows about the other's concrete type.

## Service Layer

The service layer implements the business logic. It only depends on the `TripRepository` interface — not on any specific database or storage implementation.

```go
type service struct {
    repo domain.TripRepository
}

func NewService(repo domain.TripRepository) domain.TripService {
    return &service{repo: repo}
}

func (s *service) CreateTrip(fare domain.RideFareModel) (*domain.TripModel, error) {
    trip := domain.TripModel{
        ID:       primitive.NewObjectID(),
        UserID:   "", // TODO: pull from fare.UserID
        Status:   "pending",
        RideFare: fare,
    }
    return s.repo.CreateTrip(trip)
}
```

The constructor `NewService` takes a `TripRepository` — this is dependency injection in its simplest form. At startup, we decide which repository implementation to plug in. The service doesn't care if it's an in-memory map, MongoDB, or a mock for testing.

One thing I noticed: `UserID` is hardcoded to `""` right now. It should be pulled from the request context or the fare data. Something to fix in the next iteration.

## Infrastructure Layer

For now, I went with an in-memory repository as a placeholder:

```go
type InmemRepository struct {
    trips map[string]domain.TripModel
}

func NewInmemRepository() *InmemRepository {
    return &InmemRepository{
        trips: make(map[string]domain.TripModel),
    }
}

func (r *InmemRepository) CreateTrip(trip domain.TripModel) (*domain.TripModel, error) {
    r.trips[trip.ID.Hex()] = trip
    return &trip, nil
}
```

It satisfies the `TripRepository` interface, which is all that matters. Later, I'll swap this for a real MongoDB implementation — the service layer won't need to change at all.

## Entry Point

The `cmd/main.go` is a minimal test harness to verify everything wires up:

```go
func main() {
    repo := repository.NewInmemRepository()
    svc := service.NewService(repo)

    trip, err := svc.CreateTrip(domain.RideFareModel{
        PackageType: "sedan",
        TotalPrice:  2500,
    })
    if err != nil {
        log.Fatal(err)
    }
    log.Printf("Created trip: %+v", trip)

    select {} // keep alive
}
```

It creates the dependency chain (repo -> service), runs a test trip, and loops forever to keep the container alive in Kubernetes.

## What I Learned

- **The scaffolding tool was worth building.** Having `create_service.go` meant I could focus on domain modeling instead of creating directories and boilerplate. Every new service starts with the same clean structure.
- **Start with an in-memory adapter.** It let me validate the domain model and service logic without setting up MongoDB. The interface boundary means swapping it out later is a one-line change.
- **Dependency injection in Go is just constructors.** No framework needed — pass interfaces through constructors and you get testability and swappability for free.
- **Leave TODOs visible.** The hardcoded `UserID` is a known gap. Marking it explicitly keeps it from becoming a hidden bug later.
