# Protocols and APIs

## Overview

IdentityHub implements the **Decentralized Claims Protocol (DCP)** and exposes multiple REST APIs for credential management, presentation, and administration. The architecture supports multiple protocol versions and provides both public and administrative interfaces.

**Official API Categorization** (per documentation):
- **Hub API**: Public-facing DCP protocol endpoints (exposable to internet)
- **Identity API**: Internal management endpoints (require network protection)
- **Issuer Admin API**: Administrative endpoints (highest security requirements)
- **Observability API**: Health check endpoints (container-internal only)

## Decentralized Claims Protocol (DCP)

### Protocol Implementation
**Location**: `protocols/dcp/`

**Purpose**: Implements DCP specification for credential exchange
- **Version Support**: DCP v0.8 and v1.0
- **Message Types**: PresentationQuery, PresentationResponse, CredentialOffer
- **Transport**: HTTP REST with JSON-LD

### DCP Core Components

#### 1. DCP Core
**Location**: `protocols/dcp/dcp-core/`
- Base DCP message types and transformers
- Protocol validation and processing
- Version-agnostic core functionality

#### 2. DCP IdentityHub Integration
**Location**: `protocols/dcp/dcp-identityhub/`

**Modules**:
- **`presentation-api/`**: VP generation endpoint
- **`credential-offer-api/`**: Credential offer processing
- **`storage-api/`**: Credential storage interface
- **`dcp-identityhub-core/`**: Core DCP integration logic

#### 3. DCP Issuer Components
**Location**: `protocols/dcp/dcp-issuer/`
- Issuer-side DCP protocol implementation
- Credential issuance workflows
- Integration with issuer services

### Protocol Namespaces
```java
// Supported DCP versions with different namespaces
DSPACE_DCP_NAMESPACE_V_0_8 -> DCP_SCOPE_V_0_8
DSPACE_DCP_NAMESPACE_V_1_0 -> DCP_SCOPE_V_1_0

// Context URLs for JSON-LD processing
DCP_CONTEXT_URL = "https://identity.foundation/..."
```

## API Architecture

### 1. Hub API (Public - DCP Protocol)

#### Presentation API
**Location**: `protocols/dcp/dcp-identityhub/presentation-api/`
```
POST /v1/participants/{participantContextId}/presentations
```

**Purpose**: Generate verifiable presentations for external requesters per DCP specification

#### Request Flow
1. **Authentication**: Validate self-issued token from requester
2. **Query Processing**: Parse PresentationQueryMessage
3. **Credential Resolution**: Find matching credentials
4. **VP Generation**: Create verifiable presentation
5. **Protocol Transformation**: Convert to DCP format
6. **Response**: Return PresentationResponseMessage

#### Storage API
**Status**: ⚠️ **Documented as "not yet implemented"** in official docs
**Reality**: Found implementation traces in codebase - may be partially implemented

```
POST /v1/participants/{participantContextId}/credentials  # (Storage API endpoint)
```

**Purpose**: Allow credential issuers to store newly issued credentials

#### Security Model
- **Self-Issued Tokens**: Requesters must provide valid authentication
- **Scope Validation**: Issuer scopes control credential access
- **Participant Isolation**: Only access to authorized participant's credentials
- **Internet Exposure**: Can be exposed to internet (with proper infrastructure)

### 2. Identity API (Management Interface)

#### Base Configuration
**Location**: `extensions/api/identity-api/api-configuration/`
```
Base Path: /api/identity/v1/
Context: IdentityHubApiContext.IDENTITY
Security: Internal only - requires network protection
```

#### Participant Management
**Endpoints**: `extensions/api/identity-api/participant-context-api/`
```
POST   /participants                    # Create participant (admin-only)
GET    /participants/{id}               # Get participant details  
DELETE /participants/{id}               # Delete participant (admin-only)
PUT    /participants/{id}/regenerate-token  # Regenerate API token
```

#### Credential Management
**Endpoints**: `extensions/api/identity-api/verifiable-credentials-api/`
```
GET    /participants/{id}/credentials/{credentialId}  # Get specific credential
POST   /participants/{id}/credentials                # Add new credential (✅ as of 2024-08-23)
PUT    /participants/{id}/credentials                # Update credential (✅ as of 2024-08-23)
DELETE /participants/{id}/credentials/{credentialId} # Delete credential
GET    /participants/{id}/credentials?type={type}    # Query credentials by type
```

**Important Note**: Write endpoints added in 2024-08-23 decision record for:
- Adding VCs not issued through DCP Issuance
- Populating VC storage when migrating to new IdentityHub instance
- **NOT for use as issuance APIs** - must not be exposed publicly

#### DID Management
**Endpoints**: `extensions/api/identity-api/did-api/`
```
GET    /participants/{id}/dids/{didId}   # Get DID resource
POST   /participants/{id}/dids           # Create DID
PUT    /participants/{id}/dids/{didId}   # Update DID
DELETE /participants/{id}/dids/{didId}   # Delete DID
POST   /participants/{id}/dids/{didId}/publish    # Publish DID document
DELETE /participants/{id}/dids/{didId}/publish    # Unpublish DID document
```

#### KeyPair Management
**Endpoints**: `extensions/api/identity-api/keypair-api/`
```
GET    /participants/{id}/keypairs                   # List keypairs
POST   /participants/{id}/keypairs                   # Create keypair
GET    /participants/{id}/keypairs/{keypairId}       # Get keypair
PUT    /participants/{id}/keypairs/{keypairId}       # Update keypair
DELETE /participants/{id}/keypairs/{keypairId}       # Delete keypair
POST   /participants/{id}/keypairs/{keypairId}/rotate  # Rotate keypair
POST   /participants/{id}/keypairs/{keypairId}/revoke # Revoke keypair
```

### 3. Issuer Admin API

#### Configuration
**Location**: `extensions/api/issuer-admin-api/issuer-admin-api-configuration/`
```
Base Path: /api/issuer/v1/
Purpose: Administrative interface for issuer services
Security: Administrative access only - highest security requirements
```

#### Credential Definition Management
**Endpoints**: `extensions/api/issuer-admin-api/credential-definition-api/`
```
GET    /credential-definitions           # List credential definitions
POST   /credential-definitions           # Create credential definition
GET    /credential-definitions/{id}      # Get credential definition
PUT    /credential-definitions/{id}      # Update credential definition
DELETE /credential-definitions/{id}      # Delete credential definition
```

#### Holder Management
**Endpoints**: `extensions/api/issuer-admin-api/holder-api/`
```
GET    /holders                          # List holders
POST   /holders                          # Register holder
GET    /holders/{id}                     # Get holder details
PUT    /holders/{id}                     # Update holder
DELETE /holders/{id}                     # Remove holder
```

#### Issuance Process Management
**Endpoints**: `extensions/api/issuer-admin-api/issuance-process-api/`
```
GET    /issuance-processes               # List issuance processes
POST   /issuance-processes               # Create issuance process
GET    /issuance-processes/{id}          # Get process details
PUT    /issuance-processes/{id}          # Update process
DELETE /issuance-processes/{id}          # Delete process
POST   /issuance-processes/{id}/execute  # Execute issuance
```

### 4. Observability API

**Purpose**: Application health monitoring for container orchestration
**Security**: No access control - internal container use only
**Endpoints**:
- Docker health checks
- Kubernetes readiness probes
- Application status monitoring

## Authentication & Authorization

### 1. API Authentication
**Location**: `extensions/api/identityhub-api-authentication/`

**Method**: API Key in `x-api-key` header
**Format**: `base64(participantId) + "." + base64(randomString)`

**Token Validation**:
- STS-issued access tokens
- Participant-scoped access
- JWT signature verification
- Expiration and revocation checking

### 2. Authorization Service
**Location**: `extensions/api/identityhub-api-authorization/`

**Features**:
- Role-based access control (RBAC)
- Resource-level permissions
- Participant context isolation
- Admin operation protection

### 3. Issuer Admin Authentication
**Location**: `extensions/api/issuer-admin-api/issuer-admin-api-authentication/`

**Enhanced Security**:
- Admin-level token validation
- Elevated privilege requirements
- Audit logging integration

## API Validation

### Input Validation
**Location**: `extensions/api/identity-api/validators/`

**Validators**:
- **KeyPair Validators**: Key format and algorithm validation
- **Participant Context Validators**: Manifest and context validation
- **Verifiable Credential Validators**: VC format and content validation

### JSON Schema Validation
- Request/response schema validation
- Protocol-specific message validation
- Extensible validator registry

## Protocol Transformations

### JSON-LD Processing
**Features**:
- Context expansion and compaction
- Namespace handling for different DCP versions
- Type transformation between internal and protocol formats

### Format Conversion
**Capabilities**:
- Internal model ↔ Protocol message transformation
- Multi-version protocol support
- Backward compatibility handling

## Configuration & Extension

### Web Service Registration
**Pattern**:
```java
// Service registration for different contexts
webService.registerResource(IdentityHubApiContext.IDENTITY, controller);
webService.registerResource(IdentityHubApiContext.CREDENTIALS, controller);
webService.registerResource(IssuerApiContext.ADMIN, controller);
```

### API Versioning
**Strategy**:
- Path-based versioning (`/v1/`, `/v2/`)
- Namespace-based protocol versioning
- Backward compatibility maintenance

### Extension Points
**Customization**:
- Custom validators for new resource types
- Additional authentication mechanisms
- Protocol-specific transformers
- Custom API endpoints

## Documentation Discrepancies Found

### ⚠️ Storage API Implementation Status
**Official Documentation**: "not yet implemented"
**Code Reality**: Found implementation traces in:
- `protocols/dcp/dcp-identityhub/storage-api/`
- Various storage-related controllers and services

**Resolution Needed**: Clarify actual implementation status

### ✅ Credential Write API
**Recent Addition**: 2024-08-23 decision record added write endpoints
**Status**: Implemented and working
**Usage**: For non-DCP credential imports only

### 📋 API Documentation Links
**Official Links**:
- [Presentation API Documentation](https://eclipse-edc.github.io/IdentityHub/openapi/presentation-api)
- [Identity API Documentation](https://eclipse-edc.github.io/IdentityHub/openapi/identity-api)

## Key Files

| Component | Location |
|-----------|----------|
| **DCP Core** | `protocols/dcp/dcp-core/` |
| **Presentation API** | `protocols/dcp/dcp-identityhub/presentation-api/` |
| **Identity API** | `extensions/api/identity-api/` |
| **Issuer Admin API** | `extensions/api/issuer-admin-api/` |
| **Authentication** | `extensions/api/identityhub-api-authentication/` |
| **Authorization** | `extensions/api/identityhub-api-authorization/` |
| **Validators** | `extensions/api/identity-api/validators/` |
| **DCP Transform** | `protocols/dcp/dcp-transform-lib/` |
| **DCP Validation** | `protocols/dcp/dcp-validation-lib/` |

## Implementation Notes

### Error Handling
- Standardized error response formats
- HTTP status code mapping
- Detailed error messages with failure reasons
- Security-conscious error disclosure

### Performance Considerations
- Efficient query processing for large credential sets
- Lazy loading of credential content
- Caching of frequently accessed resources
- Batch operation support where applicable

### Security Best Practices
- Input sanitization and validation
- Rate limiting and DoS protection
- Secure token handling and storage
- Comprehensive audit logging

### Protocol Compliance
- Full DCP specification implementation
- W3C VC/VP format support
- OpenAPI specification generation
- Interoperability testing coverage 