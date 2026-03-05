# AGENTS.md - Centrifugo Development Guide

This document provides guidelines for agentic coding assistants working on the Centrifugo codebase.

## Project Overview

Centrifugo is a real-time messaging server written in Go. It's a high-performance, scalable WebSocket/HTTP streaming server with support for various transports and backends.

## Build Commands

### Basic Commands
```bash
# Run all tests (excluding misc/)
make test

# Run integration tests (requires Docker services)
make test-integration

# Build the binary
make build

# Generate code (protobuf, API docs, etc.)
make generate

# Update web assets
make web

# Update Swagger UI
make swagger-web

# Manage dependencies
make deps           # Tidy go.mod
make local-deps     # Tidy, download, and vendor
```

### Testing Specific Files/Packages
```bash
# Run tests for a specific package
go test ./internal/middleware -v -race

# Run a single test
go test ./internal/middleware -v -race -run TestConnLimit

# Run tests with coverage
go test ./internal/middleware -cover -race

# Run tests with integration tag
go test ./internal/middleware -tags=integration -v -race
```

### Linting and Code Quality
```bash
# Run golangci-lint
golangci-lint run --timeout 3m0s

# Run specific linters
golangci-lint run -E errcheck,govet,staticcheck

# Check for typos
typos
```

## Code Style Guidelines

### Imports
- Group imports: standard library, third-party, internal
- Use `goimports` for automatic formatting
- Example:
```go
import (
    "context"
    "fmt"
    "net/http"

    "github.com/centrifugal/centrifuge"
    "github.com/rs/zerolog/log"
    "github.com/stretchr/testify/require"

    "github.com/centrifugal/centrifugo/v6/internal/config"
    "github.com/centrifugal/centrifugo/v6/internal/tools"
)
```

### Naming Conventions
- **Packages**: lowercase, single word, descriptive
- **Interfaces**: `-er` suffix (e.g., `Handler`, `Publisher`)
- **Variables**: camelCase, descriptive names
- **Constants**: PascalCase or ALL_CAPS for exported constants
- **Test files**: `_test.go` suffix
- **Test functions**: `TestXxx` where Xxx starts with uppercase

### Error Handling
- Always check errors, never ignore them
- Use `errors.New()` or `fmt.Errorf()` for simple errors
- Wrap errors with context using `fmt.Errorf("...: %w", err)`
- Return errors from functions, don't log them internally unless necessary
- Use `require.NoError(t, err)` in tests

### Testing Patterns
- Use `testify/require` for assertions in tests
- Table-driven tests for multiple test cases
- Use `t.Run()` for subtests
- Mock external dependencies using test helpers in `internal/tools/`
- Integration tests use `//go:build integration` tag

### File Organization
- Keep files focused and small (under 500 lines ideally)
- One public type per file when possible
- Group related functionality in the same package
- Use `internal/` for private implementation details

### Types and Interfaces
- Use strong typing, avoid `interface{}` when possible
- Define interfaces close to where they're used
- Prefer composition over inheritance
- Use `type alias` for clarity when needed

### Logging
- Use `github.com/rs/zerolog` for structured logging
- Log at appropriate levels: Debug, Info, Warn, Error
- Include context in log messages (request IDs, user IDs, etc.)
- Use structured logging fields:
```go
log.Info().Str("channel", channel).Int("clients", count).Msg("channel stats")
```

### Configuration
- Configuration is managed via `internal/config/`
- Use environment variables with `OTEL_` prefix for OpenTelemetry
- Configuration validation happens at startup

### API Design
- Use context.Context as first parameter in functions that make network calls
- Return concrete types, not interfaces (except for dependency injection)
- Use options structs for functions with many parameters
- Follow Go's convention of returning `(result, error)`

### Concurrency
- Use `sync.Mutex` for simple synchronization
- Use `context.Context` for cancellation and timeouts
- Be careful with goroutine leaks - always provide exit paths
- Use `errgroup` for coordinating multiple goroutines

### Generated Code
- Generated code lives in `internal/gen/`
- Don't edit generated files directly
- Run `make generate` after changing protobuf definitions
- Generated files should be committed to repository

## Development Workflow

1. **Before making changes**:
   - Run `make test` to ensure tests pass
   - Run `golangci-lint run` to check code quality

2. **Making changes**:
   - Follow existing patterns in the codebase
   - Write tests for new functionality
   - Update existing tests if behavior changes

3. **After making changes**:
   - Run `make test` to verify tests still pass
   - Run `make test-integration` if changes affect integration points
   - Run `golangci-lint run` to ensure code quality
   - Run `make generate` if you modified protobuf definitions

4. **Commit messages**:
   - Use descriptive commit messages
   - Reference issue numbers if applicable
   - Keep changes focused and atomic

## Common Patterns

### Middleware Pattern
```go
type Middleware interface {
    Middleware(http.Handler) http.Handler
}
```

### Handler Pattern
```go
type Handler struct {
    // dependencies
}

func (h *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // implementation
}
```

### Test Helpers
- Use `internal/tools/test_helpers.go` for common test utilities
- Mock transports and connections using `TestTransport`
- Use `NodeWithMemoryEngineNoHandlers()` for testing without full node setup

### Error Types
Define custom error types when needed:
```go
type ConfigError struct {
    Field string
    Err   error
}

func (e *ConfigError) Error() string {
    return fmt.Sprintf("config error in field %s: %v", e.Field, e.Err)
}

func (e *ConfigError) Unwrap() error {
    return e.Err
}
```

## CI/CD Pipeline

- Docker image builds and pushes to GitHub Container Registry (ghcr.io) on tag pushes
- Multi-architecture support (linux/amd64, linux/arm64)
- Automatic tagging based on semantic versioning
- GitHub Actions cache for faster builds
- Uses GitHub Token for authentication to ghcr.io

## Tools and Dependencies

- **Go version**: 1.25.0 (minimum)
- **Linter**: golangci-lint with custom config (see `.golangci.yml`)
- **Testing**: testify, built-in testing package
- **Logging**: zerolog
- **Configuration**: viper, cobra
- **Protocol**: gRPC, WebSocket, HTTP/2, QUIC

## Notes for Agents

- Always check for existing patterns before implementing new functionality
- The codebase uses many external dependencies - check `go.mod` before adding new ones
- Performance is critical - avoid allocations in hot paths
- Memory safety is important - use proper synchronization
- Backward compatibility matters - don't break existing APIs without good reason
- Security is paramount - validate all inputs, sanitize outputs