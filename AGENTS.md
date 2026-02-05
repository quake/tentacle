# AGENTS.md - Tentacle P2P Framework

> AI coding agent instructions for the Tentacle codebase.

## Project Overview

Tentacle is a multiplexed P2P network framework written in Rust, based on yamux. It supports mounting custom protocols for peer-to-peer communication.

**Structure:** Cargo workspace with multiple crates:
- `tentacle/` - Main P2P framework library
- `yamux/` - Yamux multiplexing protocol (tokio-yamux)
- `secio/` - Secure IO encryption (tentacle-secio)
- `multiaddr/` - Multi-address handling (tentacle-multiaddr)
- `bench/` - Benchmarks
- `protocols/` - Reference implementations (unmaintained)

**Rust Edition:** 2024 | **MSRV:** 1.85.0

## Build, Test, and Lint Commands

### Building

```bash
cd tentacle && cargo build --features ws,unstable,tls  # Standard build
make build    # Build all feature combinations (CI check)
make examples # Build examples
```

### Testing

```bash
make test  # Run all tests
cd tentacle && cargo test --all --features ws,unstable,tls,upnp  # Direct

# Run a single test by name
cd tentacle && cargo test test_dial_with_secio --features ws,unstable,tls -- --nocapture

# Run a specific test file
cd tentacle && cargo test --test test_dial --features ws,unstable,tls
```

### Linting and Formatting

```bash
make fmt     # Check formatting (cargo fmt --all -- --check)
make clippy  # Run clippy with strict warnings
make ci      # Full CI check
```

### Other Commands

```bash
make fuzz    # Fuzz tests (requires nightly)
make cov     # Code coverage
make gen-mol # Regenerate molecule files
```

## Feature Flags

- **Default:** `tokio-runtime,tokio-timer`
- **Full testing:** `ws,unstable,tls,upnp`
- **WebAssembly:** `wasm-timer,unstable --no-default-features`
- **Key flags:** `ws`, `tls`, `upnp`, `unstable`, `secio-async`, `metrics`, `parking_lot`

## Code Style Guidelines

### Formatting (rustfmt.toml)

Max width: 100 | Tab: 4 spaces | Imports: alphabetical | Use `?` operator

### Import Order

1. Standard library (`std::`) | 2. External crates | 3. Internal (`crate::`)

```rust
use std::{io, sync::Arc, time::Duration};

use bytes::Bytes;
use tokio_util::codec::LengthDelimitedCodec;

use crate::{ProtocolId, service::ProtocolMeta};
```

### Naming Conventions

| Element | Convention | Example |
|---------|------------|---------|
| Files | snake_case | `protocol_handle_stream.rs` |
| Functions | snake_case | `create_meta`, `handle_error` |
| Types/Structs | PascalCase | `ServiceBuilder`, `ProtocolMeta` |
| Constants | SCREAMING_SNAKE_CASE | `MAX_BUF_SIZE`, `HEADER_SIZE` |
| Acronyms in types | Preserved | `IOError`, `TLS` |
| Type aliases | PascalCase | `type Result<T> = std::result::Result<T, Error>` |

### Type Patterns

- **Newtype wrappers** for IDs: `pub struct ProtocolId(usize)`
- **Builder pattern** for complex configs: `ServiceBuilder::default().insert_protocol(meta).build(handle)`
- **Arc** for shared state across async boundaries
- **Feature gating**: `#[cfg(feature = "tls")]`

### Error Handling

Use `thiserror` for error definitions:

```rust
#[derive(Error, Debug)]
pub enum TransportErrorKind {
    #[error("transport io error: `{0:?}`")]
    Io(#[from] IOError),
    #[error("multiaddr `{0:?}` is not supported")]
    NotSupported(Multiaddr),
}
```

### Async Patterns

Use `async_trait` for async trait definitions, Tokio as default runtime, channel-based communication (mpsc, oneshot):

```rust
#[async_trait::async_trait]
pub trait ServiceHandle: Send {
    async fn handle_error(&mut self, _control: &mut ServiceContext, _error: ServiceError) {}
}
```

### Documentation

- Doc comments (`///`) required on all public items - enforced by `#![deny(missing_docs)]`
- Module-level docs use `//!`
- Implementation comments use `//`

### Testing Patterns

- Tests in `tentacle/tests/` directory
- Helper functions for service setup (e.g., `create()`)
- Test both with and without secio (encryption)
- Use `crossbeam-channel` for test synchronization
- Create runtime per test: `tokio::runtime::Runtime::new()`

## Clippy Rules

- `-D clippy::let_underscore_must_use` - Deny unused must-use values
- All warnings treated as errors (`RUSTFLAGS='-W warnings'`)

## Key Architectural Concepts

### Protocol Traits

- `ServiceProtocol` - Global protocol handler (lives for service lifetime)
- `SessionProtocol` - Per-session handler (lives for session lifetime)  
- `ProtocolSpawn` - Stream-based protocol with full async control
- `ServiceHandle` - Service-level event handler

### Building a Service

1. Define protocol behavior with traits
2. Build `ProtocolMeta` from `MetaBuilder`
3. Register protocols in `ServiceBuilder`
4. Build and run `Service` on async runtime