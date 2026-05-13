# ThunderPropagator.Clients

ThunderPropagator Client Protocol Specification — shared wire format, protocols, options, and compliance checklist for all client libraries.

> **This is the authoritative specification.** Every client — regardless of language or platform — must implement exactly this protocol. Deviations require an approved RFC against this repository.

Full specification: [`README_Clients_Protocol.md`](README_Clients_Protocol.md)

---

## Documentation

This repository publishes generated documentation under [`/docs`](docs/README.md).

- [Overview](docs/README.md#overview)
- [Transport Layer](docs/README.md#transport-layer)
- [Connection & Handshake](docs/README.md#connection--handshake)
- [Authentication](docs/README.md#authentication)
- [Format Negotiation](docs/README.md#format-negotiation)
- [Subscriptions & Push Formats](docs/README.md#subscriptions--push-formats)
- [Record Status Codes](docs/README.md#record-status-codes)
- [Splitter Escaping](docs/README.md#splitter-escaping)
- [Heartbeat](docs/README.md#heartbeat)
- [Reconnection & Failover](docs/README.md#reconnection--failover)
- [Configuration Reference](docs/README.md#configuration-reference)
- [Mobile Lifecycle](docs/README.md#mobile-lifecycle)
- [Encryption Support](docs/README.md#encryption-support)
- [Error Handling](docs/README.md#error-handling)
- [Client Library Status](docs/README.md#client-library-status)
- [Diagrams](docs/README.md#diagrams)

### Docs Catalog

- Protocol Specification `Files:1` `Diagrams:✓`
  - [Transport & Negotiation](docs/README.md#transport-layer) `Diagrams:✓`
  - [Connection & Handshake](docs/README.md#connection--handshake) `Diagrams:✓`
  - [Authentication & Format Negotiation](docs/README.md#authentication) `Diagrams:✗`
  - [Subscriptions & Push Formats](docs/README.md#subscriptions--push-formats) `Diagrams:✓`
  - [Reconnection & Failover](docs/README.md#reconnection--failover) `Diagrams:✓`
  - [Mobile Lifecycle](docs/README.md#mobile-lifecycle) `Diagrams:✗`
  - [Error Handling](docs/README.md#error-handling) `Diagrams:✗`

**Last generated:** 2026-05-13

---

## Client Libraries

| Language | Package | Repository | Status |
|----------|---------|------------|--------|
| .NET (C#) | `ThunderPropagator.Clients.DotNet` | [Clients.DotNet](https://github.com/KiarashMinoo/ThunderPropagator.Clients.DotNet) | ✅ Reference implementation |
| JavaScript / TypeScript | `@thunderpropagator/client` | Clients.JS | 🔲 Planned |
| Python | `thunderpropagator-client` | Clients.Python | 🔲 Planned |
| Java | `com.thunderpropagator:client` | Clients.Java | 🔲 Planned |
| Go | `github.com/KiarashMinoo/thunderpropagator-go` | Clients.Go | 🔲 Planned |
| Rust | `thunderpropagator-client` | Clients.Rust | 🔲 Planned |
| C++ | `thunderpropagator-cpp` | Clients.Cpp | 🔲 Planned |
| Swift / Objective-C | SPM package | Clients.Swift | 🔲 Planned |
| Flutter / Dart | `thunderpropagator_client` | Clients.Flutter | 🔲 Planned |

---

## Contributing

To propose changes to this specification:

1. Open an issue describing the change and motivation.
2. Wire-format–breaking changes require a major version bump.
3. All client library maintainers must acknowledge before merging.
4. After merge, client libraries have 30 days to implement and release the change.

---

## License

[MIT](LICENSE) © 2026 Kiarash Minoo
