# Repository Structure & Organization

## Overview

The IdentityHub repository follows a **modular, layered architecture** based on the Eclipse Dataspace Connector (EDC) extension system. The structure promotes clean separation of concerns, testability, and extensibility.

## Top-Level Directory Structure

```
IdentityHub/
├── core/                     # Core business logic & services
├── spi/                      # Service Provider Interfaces
├── extensions/               # Extension implementations  
├── protocols/                # Protocol implementations (DCP)
├── launcher/                 # Runtime launchers & configurations
├── e2e-tests/               # End-to-end testing suites
├── docs/                    # Architecture & developer documentation
├── dist/                    # Distribution BOMs (Bill of Materials)
├── resources/               # Shared resources (OpenAPI specs)
├── gradle/                  # Gradle build configuration
├── version-catalog/         # Dependency version management
└── build.gradle.kts         # Root build configuration
```

## Core Module Architecture

### `/core/` - Business Logic Layer

#### Common Infrastructure
- **`common-core/`**: Shared utilities, base classes, and common patterns
- **`lib/`**: Reusable libraries for specific concerns
  - `keypair-lib/`: Cryptographic key management utilities
  - `accesstoken-lib/`: Token handling and validation
  - `common-lib/`: Cross-cutting utilities
  - `issuerservice-common-lib/`: Issuer-specific shared code

#### Identity Hub Core Services
- **`identity-hub-core/`**: Main service implementations
  - Credential management services
  - Presentation generation services
  - Core business orchestration
- **`identity-hub-participants/`**: Participant context management
- **`identity-hub-keypairs/`**: Cryptographic key lifecycle
- **`identity-hub-did/`**: DID document management

#### Issuer Service Core
- **`issuerservice/`**: Credential issuance functionality
  - `issuerservice-core/`: Core issuance logic
  - `issuerservice-credentials/`: Credential management
  - `issuerservice-holders/`: Holder registration & management
  - `issuerservice-issuance/`: Issuance process orchestration

### `/spi/` - Service Provider Interfaces

**Purpose**: Define contracts between components, enabling loose coupling and extensibility

#### Core SPIs
- **`identity-hub-spi/`**: Main IdentityHub service interfaces
- **`participant-context-spi/`**: Multi-tenancy interfaces
- **`verifiable-credential-spi/`**: Credential management interfaces
- **`keypair-spi/`**: Key management interfaces
- **`did-spi/`**: DID resolution and management interfaces
- **`sts-spi/`**: Secure Token Service interfaces

#### Issuer Service SPIs
- **`issuerservice/`**: Issuer-specific service interfaces
  - `issuerservice-holder-spi/`: Holder management contracts
  - `issuerservice-credential-spi/`: Credential definition contracts
  - `issuerservice-issuance-spi/`: Issuance process contracts

#### Request Management SPIs
- **`holder-credential-request-spi/`**: Credential request lifecycle interfaces

## Extensions Layer

### `/extensions/` - Implementation Layer

#### Storage Extensions
**`store/sql/`**: SQL-based persistent storage implementations
- `identity-hub-credentials-store-sql/`: Credential storage
- `identity-hub-participantcontext-store-sql/`: Participant context storage
- `identity-hub-keypair-store-sql/`: Key pair storage
- `identity-hub-did-store-sql/`: DID resource storage
- `holder-credential-*-store-sql/`: Credential request storage
- `issuerservice-*-store-sql/`: Issuer service storage
- `sts-client-store-sql/`: STS client management storage

#### API Extensions
**`api/`**: REST API implementations

**Identity API** (`identity-api/`):
- `api-configuration/`: Base configuration
- `participant-context-api/`: Participant management endpoints
- `verifiable-credentials-api/`: Credential CRUD endpoints
- `did-api/`: DID management endpoints
- `keypair-api/`: Key management endpoints
- `validators/`: Input validation components

**Issuer Admin API** (`issuer-admin-api/`):
- `issuer-admin-api-configuration/`: Admin API configuration
- `holder-api/`: Holder management endpoints
- `credentials-api/`: Credential definition endpoints
- `attestation-api/`: Attestation management
- `issuance-process-api/`: Process management endpoints

**Authentication & Authorization**:
- `identityhub-api-authentication/`: Token validation
- `identityhub-api-authorization/`: Access control
- `lib/identityhub-api-authentication-lib/`: Authentication utilities

#### Feature Extensions
**`credentials/`**: Credential-specific functionality
- `credential-offer-handler/`: Processes incoming credential offers

**`did/`**: DID-specific functionality  
- `local-did-publisher/`: Publishes DID documents to local storage/CDN

**`issuance/`**: Issuance-specific functionality
- `issuerservice-presentation-attestations/`: Presentation-based attestations
- `issuerservice-database-attestations/`: Database-sourced attestations
- `issuerservice-issuance-rules/`: Business rule engine
- `local-statuslist-publisher/`: Status list publication

**`sts/`**: Secure Token Service
- `sts-account-provisioner/`: Account lifecycle management
- `sts-account-service-local/`: Local account service
- `sts-core/`: STS core functionality
- `sts-api/`: STS REST endpoints

**`common/`**: Cross-cutting concerns
- `credential-watchdog/`: Monitors credential lifecycle and expiration

## Protocol Implementation

### `/protocols/dcp/` - Decentralized Claims Protocol

#### Core Protocol
- **`dcp-spi/`**: DCP service interfaces
- **`dcp-core/`**: Core protocol implementation
- `dcp-transform-lib/`: Message transformation utilities
- `dcp-validation-lib/`: Protocol validation logic

#### IdentityHub Integration
**`dcp-identityhub/`**: DCP integration for credential holders
- `credentials-api-configuration/`: API configuration for DCP
- `presentation-api/`: VP generation endpoints
- `storage-api/`: Credential storage integration
- `credential-offer-api/`: Credential offer processing
- `dcp-identityhub-core/`: Core integration logic

#### Issuer Integration
**`dcp-issuer/`**: DCP integration for credential issuers
- `dcp-issuer-spi/`: Issuer-specific SPI
- `dcp-issuer-api/`: Issuer API endpoints
- `dcp-issuer-core/`: Issuer core functionality

## Runtime & Distribution

### `/launcher/` - Executable Runtimes
- **`identityhub/`**: Main IdentityHub runtime launcher
- **`issuer-service/`**: Dedicated issuer service runtime

### `/dist/` - Distribution Management
**Bill of Materials (BOM) modules** for dependency management:
- `identityhub-base-bom/`: Base IdentityHub dependencies
- `identityhub-bom/`: Complete IdentityHub distribution
- `identityhub-feature-sql-bom/`: SQL feature set
- `issuerservice-base-bom/`: Base issuer service dependencies
- `issuerservice-bom/`: Complete issuer service distribution
- `issuerservice-feature-sql-bom/`: Issuer SQL feature set

## Testing Strategy

### `/e2e-tests/` - End-to-End Testing
- **`identity-api-tests/`**: Identity API integration tests
- **`admin-api-tests/`**: Admin API integration tests
- **`sts-api-tests/`**: STS API integration tests
- **`dcp-issuance-tests/`**: DCP protocol issuance tests
- **`bom-tests/`**: BOM dependency validation tests
- **`identityhub-test-fixtures/`**: Shared test utilities
- **`runtimes/sts/`**: STS runtime for testing
- **`tck-tests/`**: Technology Compatibility Kit tests

## Documentation

### `/docs/` - Technical Documentation
- **`developer/`**: Developer-focused documentation
  - `architecture/`: System architecture diagrams and descriptions
  - `decision-records/`: Architectural Decision Records (ADRs)
- **OpenAPI specifications**: Generated API documentation

### `/resources/` - Shared Resources
- **`openapi/`**: OpenAPI specification files
- **Configuration files**: Checkstyle, build configurations

## Build System

### Gradle Configuration
- **`build.gradle.kts`**: Root build script
- **`settings.gradle.kts`**: Module inclusion and configuration
- **`gradle.properties`**: Build properties and versions
- **`version-catalog/`**: Centralized dependency version management
- **`gradle/wrapper/`**: Gradle wrapper for consistent builds

## Module Naming Conventions

### Patterns
- **Core modules**: `{domain}-{type}` (e.g., `identity-hub-core`)
- **SPI modules**: `{domain}-spi` (e.g., `verifiable-credential-spi`)
- **Extension modules**: `{feature}-{type}` (e.g., `credential-offer-handler`)
- **Store modules**: `{resource}-store-{technology}` (e.g., `identity-hub-credentials-store-sql`)
- **API modules**: `{resource}-api` (e.g., `participant-context-api`)

### Grouping Strategy
- **By concern**: Related functionality grouped together
- **By layer**: Clear separation between SPI, core, and extensions
- **By technology**: Storage implementations grouped by technology
- **By protocol**: Protocol implementations isolated

## Architectural Benefits

### Modularity
- **Independent deployability**: Modules can be deployed separately
- **Selective functionality**: Include only needed features
- **Technology flexibility**: Swap implementations (e.g., different databases)

### Extensibility
- **SPI-based design**: Clean extension points
- **Plugin architecture**: Add new functionality without core changes
- **Protocol agnostic**: Support multiple protocols and versions

### Testability
- **Unit testing**: Each module independently testable
- **Integration testing**: API and protocol-level test suites
- **E2E testing**: Complete system validation

### Maintainability
- **Clear boundaries**: Well-defined module responsibilities
- **Dependency management**: Centralized version control
- **Documentation**: Comprehensive architecture documentation 