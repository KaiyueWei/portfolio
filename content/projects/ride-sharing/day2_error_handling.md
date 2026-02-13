---
title: "Day 2: Error Handling in Go"
date: 2026-02-11
draft: false
---

In Go, errors are **values**, not exceptions. There is no `try/catch` — instead, functions return an `error` as part of their return values, and the caller is responsible for checking it. This makes error handling explicit and forces you to think about failure paths at every step.

## The `error` Interface

At its core, Go's `error` type is just an interface:

```go
type error interface {
	Error() string
}
```

Any type that implements the `Error() string` method satisfies this interface. The built-in `errors.New()` and `fmt.Errorf()` are the most common ways to create errors.

## Sentinel Errors

Sentinel errors are package-level variables that represent specific, well-known error conditions. By convention, they use the `Err` prefix.

```go
var (
	ErrNotImplemented = errors.New("not implemented")
	ErrNotFound       = errors.New("not found")
)
```

> **Note:** The short declaration operator `:=` is only allowed **inside functions** as a statement. The `var ()` block works at **both** the package level and inside functions.
> The key reason: **package-level variables can't use `:=`**.

## Returning and Wrapping Errors

Here's a practical example — a `Truck` type with a method that can fail:

```go
type Truck struct {
	id string
}

func (t *Truck) LoadCargo() error {
	// simulate a failure
	return ErrNotImplemented
}

func processTruck(truck Truck) error {
	fmt.Printf("Processing truck: %s\n", truck.id)

	if err := truck.LoadCargo(); err != nil {
		return fmt.Errorf("error loading cargo for %s: %w", truck.id, err)
	}
	return nil
}
```

The `%w` verb in `fmt.Errorf` **wraps** the original error, preserving the error chain so callers can unwrap and inspect it later with `errors.Is()` or `errors.As()`.

## Handling Errors

### Option 1: Check after the call

```go
err := processTruck(truck)
if err != nil {
	log.Fatalf("error processing truck: %v", err)
}
```

### Option 2: Inline short declaration (idiomatic Go)

```go
if err := processTruck(truck); err != nil {
	log.Fatalf("error processing truck: %v", err)
}
```

Option 2 is more idiomatic — it scopes `err` to the `if` block, keeping the surrounding scope clean.

## Matching Sentinel Errors

When a function can return different types of errors, you can branch on them. Use `errors.Is()` to match against sentinel errors — this works even if the error was wrapped with `fmt.Errorf("...: %w", err)`:

```go
func main() {
	trucks := []Truck{
		{id: "Truck-1"},
		{id: "Truck-2"},
		{id: "Truck-3"},
	}

	for _, truck := range trucks {
		fmt.Printf("Truck %s arrived.\n", truck.id)
		err := processTruck(truck)
		if err != nil {
			switch {
			case errors.Is(err, ErrNotImplemented):
				fmt.Printf("Skipping %s: not implemented yet\n", truck.id)
				continue
			case errors.Is(err, ErrNotFound):
				fmt.Printf("Skipping %s: not found\n", truck.id)
				continue
			default:
				log.Fatalf("unexpected error processing %s: %v", truck.id, err)
			}
		}
	}
}
```

> **Why `errors.Is()` over `==`?** A direct comparison like `err == ErrNotFound` fails if the error has been wrapped. `errors.Is()` unwraps the error chain and checks each layer, so it works with `fmt.Errorf("...: %w", err)`.

## Custom Error Types

For richer error information, you can define your own error type:

```go
type ValidationError struct {
	Field   string
	Message string
}

func (e *ValidationError) Error() string {
	return fmt.Sprintf("validation failed on %s: %s", e.Field, e.Message)
}
```

Use `errors.As()` to extract a custom error type from a wrapped chain:

```go
var validationErr *ValidationError
if errors.As(err, &validationErr) {
	fmt.Printf("Field: %s, Message: %s\n", validationErr.Field, validationErr.Message)
}
```

## Summary

| Concept | Use |
|---|---|
| `errors.New()` | Create simple sentinel errors |
| `fmt.Errorf("...: %w", err)` | Wrap errors with context |
| `errors.Is(err, target)` | Check if an error matches a sentinel |
| `errors.As(err, &target)` | Extract a specific error type from the chain |
| Custom error types | Carry structured data with errors |
