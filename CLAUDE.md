# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Vector is a high-performance, end-to-end observability data pipeline built in Rust. It collects, transforms, and routes logs, metrics, and traces to any vendor. The project emphasizes reliability, performance, and correctness.

## Build System and Common Commands

### Development with cargo vdev

Vector uses a custom development tool called `vdev` that wraps common operations. Most commands should use `cargo vdev` rather than plain `cargo`:

```bash
# Build for development
cargo build

# Build for release
make build  # or cargo build --release --no-default-features --features default

# Run unit tests (requires cargo-nextest)
cargo vdev test
# or cargo nextest run --workspace --no-default-features --features "default"

# Run specific component tests
cargo test --lib --no-default-features --features sinks-console sinks::console

# Run integration tests (requires Docker/Podman)
cargo vdev int test <integration-name>  # e.g., cargo vdev int test aws
make test-integration-<name>            # e.g., make test-integration-aws

# Format code
cargo vdev fmt
# or make fmt

# Run linter checks
cargo vdev check rust
# or make check-clippy

# Check that code is formatted
cargo vdev check fmt
# or make check-fmt

# Check component documentation is up to date
make check-component-docs

# Check licenses are up to date
cargo vdev check licenses
# or make check-licenses

# Generate component docs
make generate-component-docs
```

### Docker/Podman Environment

For contributors without a full local toolchain, Vector provides a containerized development environment:

```bash
# Enter the development container
make environment

# Run commands in the container (without entering shell)
make <target> ENVIRONMENT=true
# Examples:
# make test ENVIRONMENT=true
# make build-dev ENVIRONMENT=true
# make test-integration SCOPE="sources::file" ENVIRONMENT=true
```

### Running Specific Tests

```bash
# Run tests for a specific module
cargo test sources::file
cargo vdev test sources::file

# Run integration tests for a component
cargo vdev int test kafka

# Run tests with specific features only
cargo test --lib --no-default-features --features transforms-remap transforms::remap
```

## Architecture

### High-Level Structure

Vector follows a pipeline architecture with three core component types:
- **Sources**: Collect data from external systems (files, kafka, http, etc.)
- **Transforms**: Process and modify data (filter, remap, aggregate, etc.)
- **Sinks**: Send data to destinations (elasticsearch, s3, datadog, etc.)

### Directory Structure

- `/src` - Main Vector source code
  - `/src/sources` - Source implementations (data ingestion)
  - `/src/transforms` - Transform implementations (data processing)
  - `/src/sinks` - Sink implementations (data output)
  - `/src/config` - Configuration loading and validation
  - `/src/topology` - Pipeline graph construction and runtime
  - `/src/internal_events` - Instrumentation events for observability
  - `/src/api` - GraphQL API for introspection and control
  - `/src/kubernetes` - Kubernetes-specific integration code

- `/lib` - Workspace libraries (independent of main vector binary)
  - `/lib/vector-lib` - Core shared types and traits
  - `/lib/vector-core` - Core event types and processing
  - `/lib/vector-config` - Configuration schema and parsing
  - `/lib/vector-buffers` - Disk and memory buffering
  - `/lib/codecs` - Encoding/decoding (JSON, protobuf, etc.)
  - `/lib/file-source` - Shared file tailing implementation
  - `/lib/vector-vrl` - Vector Remap Language (VRL) integration

- `/benches` - Performance benchmarks
- `/tests` - Integration and end-to-end tests
- `/vdev` - Development tooling (custom cargo subcommand)

### Component Architecture

Components are organized by type and feature-gated:
- Each component must be behind a feature flag (e.g., `sources-kafka`, `sinks-elasticsearch`)
- Components follow a common trait pattern defined in `vector-lib`
- Configuration is defined using the `vector-config` derive macros
- Components emit instrumentation via the `internal_events` system

### Key Libraries

- **vector-lib**: Defines core traits (`SourceSender`, `StreamSink`, etc.) and event types
- **vector-core**: Event data model (`LogEvent`, `Metric`, `TraceEvent`) and internal telemetry
- **vector-config**: Configuration schema with automatic documentation generation
- **VRL (Vector Remap Language)**: External DSL for data transformation (in separate repo, used as dependency)

## Development Guidelines

### Feature Flags

All new components must be feature-gated:
```toml
[features]
sources-mycomponent = ["dep1", "dep2"]
```

The feature should follow naming: `<type>-<component_name>` (e.g., `sources-kafka`, `sinks-http`)

### Code Style

- Use `rustfmt` for formatting (run via `make fmt` or `cargo vdev fmt`)
- Follow Vector's logging style:
  - Use tracing crate's key/value style
  - Capitalize log messages and end with period
  - Use Display over Debug: `%error` not `?error`
  - Example: `warn!(message = "Failed to merge value.", %error);`
- Avoid panics except in truly impossible states (must document in function docs)

### Adding New Components

When adding a source, sink, or transform:
1. Place code in appropriate directory (`src/sources`, `src/sinks`, `src/transforms`)
2. Add feature flag in `Cargo.toml`
3. Add instrumentation using `internal_events` patterns
4. Write unit tests and integration tests
5. Update configuration documentation (auto-generated from code)
6. Add integration test workflow entries in `.github/workflows/integration.yml` and `.github/workflows/integration-comment.yml`
7. Update `.github/workflows/changes.yml` with filter for new files

### Testing

- Unit tests: Should not require external services
- Integration tests: Require Docker/Podman for test services
- Use `cargo-nextest` for running tests (installed separately)
- Integration tests are discovered via `cargo vdev int show`

### Configuration

- Configuration uses a custom derive system from `vector-config`
- Component configs are automatically documented from code
- Configs support TOML, YAML, and JSON formats
- Secret values can use secret backends (e.g., AWS Secrets Manager)

### VRL (Vector Remap Language)

VRL is the primary transformation language used in the `remap` transform:
- Maintained in separate repository (github.com/vectordotdev/vrl)
- Used as a git dependency
- Add custom VRL functions to `lib/vector-vrl/functions`
- Test VRL with `make test-vrl`

### Performance

- Vector uses jemalloc allocator on Unix systems
- Benchmarks in `/benches` use criterion
- Run benchmarks: `make bench` or `cargo bench --no-default-features --features benches`
- Test harness for end-to-end performance: https://github.com/vectordotdev/vector-test-harness

### Kubernetes Development

For working on Kubernetes integrations:
- Source: `src/sources/kubernetes_logs`
- E2E tests: `lib/k8s-e2e-tests`
- Use Tilt for live development: `tilt up` (requires minikube/k8s cluster)
- Helm charts in separate repo: github.com/vectordotdev/helm-charts

## Common Pitfalls

- Don't use bare `cargo` commands for tests - use `cargo vdev test` (requires `cargo-nextest`)
- Don't forget to add feature flags for new components and their dependencies
- Don't commit without running `make fmt`
- Integration tests need Docker/Podman running
- When modifying dependencies, run `make build-licenses` to update `LICENSE-3rdparty.csv`
- Component documentation is auto-generated; run `make generate-component-docs` after schema changes

## Minimum Supported Rust Version (MSRV)

MSRV is specified in `rust-toolchain.toml` and `Cargo.toml`. Can be bumped as needed for dependencies or language features.
