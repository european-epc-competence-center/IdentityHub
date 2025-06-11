# EDC Identity Hub - Project Notes Index

## Overview

The EDC Identity Hub is a comprehensive implementation of the **Decentralized Claims Protocol (DCP)** specification, functioning as a "wallet" for Verifiable Credentials. It provides secure storage, management, and presentation of verifiable credentials in a multi-tenant environment.

**Eclipse EDC Context**: IdentityHub is part of the [Eclipse Dataspace Connector (EDC)](https://eclipse-edc.github.io/) ecosystem, which is an open source framework for building secure, globally-scalable data-sharing services. EDC provides components for control planes, data planes, decentralized identity systems, and federated data catalogs. IdentityHub specifically implements the **decentralized identity systems** component of this ecosystem.

**Current Status**: As of 2023-09-27, IdentityHub has migrated from a partial DWN (Decentralized Web Node) implementation to fully adopt the **Identity and Trust Protocols**, marking a significant architectural evolution toward DCP compliance.

## Project Structure

```
IdentityHub/
├── core/                    # Core business logic modules
├── spi/                     # Service Provider Interfaces
├── extensions/              # Extension implementations
├── protocols/               # Protocol implementations (DCP)
├── launcher/                # Runtime launchers
├── e2e-tests/              # End-to-end tests
├── docs/                   # Architecture documentation
└── dist/                   # Distribution BOMs
```

## Core Components

### 1. Multi-Tenancy & Participant Management
- **Participant Context**: Core unit of management for identity resources
- **File**: [participant_context.md](participant_context.md)

### 2. Verifiable Credentials Management
- **Credential Resources**: Storage and lifecycle management of VCs
- **File**: [verifiable_credentials.md](verifiable_credentials.md)

### 3. Identity Services
- **DID Management**: DID creation, resolution, and publishing
- **KeyPair Management**: Cryptographic key lifecycle
- **File**: [identity_services.md](identity_services.md)

### 4. Presentation Services
- **VP Generation**: Creates verifiable presentations from credentials
- **File**: [presentation_services.md](presentation_services.md)

### 5. Protocol & API Layer
- **DCP Protocol**: Implementation of decentralized claims protocol
- **REST APIs**: Identity API, Presentation API, Admin APIs
- **File**: [protocols_and_apis.md](protocols_and_apis.md)

### 6. Storage & Persistence
- **SQL Extensions**: Database implementations for all stores
- **File**: [storage_architecture.md](storage_architecture.md)

### 7. Security & Authorization
- **STS Integration**: Secure Token Service for authentication
- **Authorization**: Role-based access control
- **File**: [security_architecture.md](security_architecture.md)

## Key Architecture Patterns

- **EDC Extension System**: Modular architecture based on Eclipse Dataspace Connector
- **SPI-based Design**: Clean separation between interfaces and implementations
- **Event-Driven Architecture**: Participant lifecycle events, credential state changes
- **Multi-format Support**: JWT, JSON-LD, VC 1.0, VC 2.0 support
- **Embedded STS Deployment**: STS is always embedded in IdentityHub runtime (as of 2025-02-24)

## API Architecture

### Hub API (Public - DCP Protocol)
- **Presentation API**: VP generation for external requesters (`/v1/participants/{id}/presentations`)
- **Storage API**: Credential storage for issuers *(not yet implemented)*

### Identity API (Management - Internal)
- **Participant Management**: Participant context CRUD operations
- **Credential Management**: VC storage and retrieval (including write endpoints as of 2024-08-23)
- **DID Management**: DID document lifecycle
- **KeyPair Management**: Cryptographic key operations

### Issuer Admin API (Administrative)
- **Credential Definition Management**: Define credential schemas
- **Holder Management**: Register and manage credential holders
- **Issuance Process Management**: Orchestrate credential issuance workflows

## Development Areas

### Issuance Services
- **Credential Issuance**: Automated credential generation and management
- **Status Management**: Revocation and suspension capabilities
- **File**: [issuer_services.md](issuer_services.md)

### Testing & Quality
- **E2E Tests**: Comprehensive testing across all components
- **API Tests**: Identity API, Admin API, STS API testing
- **File**: [testing_strategy.md](testing_strategy.md)

## Documentation Verification Status

### ✅ Verified Against Official Documentation
- **API Structure**: Confirmed Hub API vs Identity API distinction
- **Security Model**: API key authentication with `x-api-key` header format verified
- **STS Deployment**: Embedded-only deployment model confirmed (2025-02-24 decision)
- **Multi-tenancy**: Participant context isolation model verified
- **Protocol Implementation**: DCP v0.8 and v1.0 support confirmed

### ⚠️ Documentation Gaps Found
- **Storage API**: Documented as "not yet implemented" in official docs, but found implementation traces in codebase
- **Credential Write API**: Recent addition (2024-08-23) may not be reflected in all documentation
- **Identity Trust Protocol Migration**: Architecture documents may lag behind actual implementation

### 📝 Additional Findings
- **Project Evolution**: Significant architectural shift in 2023 from DWN to Identity and Trust Protocols
- **EDC Integration**: Deep integration with EDC ecosystem not fully apparent from repository structure alone
- **Deployment Model**: Recent decision (2025-02-24) mandates embedded STS deployment only

## Quick Navigation

| Component | Description | Key Files |
|-----------|-------------|-----------|
| **Core Services** | Business logic implementations | `core/identity-hub-core/`, `core/identity-hub-*` |
| **SPIs** | Service interfaces | `spi/verifiable-credential-spi/`, `spi/participant-context-spi/` |
| **APIs** | REST endpoints | `extensions/api/identity-api/`, `protocols/dcp/` |
| **Storage** | SQL implementations | `extensions/store/sql/` |
| **Security** | STS and authorization | `extensions/sts/`, `extensions/api/*-authentication` |

## References

- [Eclipse EDC Ecosystem](https://eclipse-edc.github.io/) - Main EDC framework documentation
- [IdentityHub API Documentation](https://eclipse-edc.github.io/IdentityHub/openapi/) - Generated API specs
- [Architecture Documentation](../docs/developer/architecture/) - Technical architecture details
- [Decision Records](../docs/developer/decision-records/) - Architectural evolution decisions
- [DCP Specification](https://projects.eclipse.org/projects/technology.dataspace-dcp) - Protocol specification
- [Main README](../README.md) - Project overview and quick start 