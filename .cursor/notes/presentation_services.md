# Verifiable Presentation Services

## Overview

The **VerifiablePresentationService** is responsible for creating verifiable presentations (VPs) from stored verifiable credentials. It supports multiple credential formats and generates format-appropriate presentations with cryptographic proofs.

## Core Architecture

### VerifiablePresentationService Interface
**Location**: `spi/verifiable-credential-spi/src/main/java/org/eclipse/edc/identityhub/spi/verifiablecredentials/generator/VerifiablePresentationService.java`

```java
@FunctionalInterface
interface VerifiablePresentationService {
    Result<PresentationResponseMessage> createPresentation(
        String participantContextId,                    // Participant creating VP
        List<VerifiableCredentialContainer> credentials,// VCs to include
        @Nullable PresentationDefinition presentationDefinition, // Query constraints
        @Nullable String audience                       // VP recipient
    );
}
```

### Implementation
**Location**: `core/identity-hub-core/src/main/java/org/eclipse/edc/identityhub/core/services/verifiablepresentation/VerifiablePresentationServiceImpl.java`

**Key Features**:
- **Multi-format Support**: Groups credentials by format for optimal VP creation
- **Format Isolation**: Creates separate VPs for incompatible formats
- **Audience Integration**: Includes audience in VP proofs when provided

## Format-Specific Generators

### 1. JWT Presentation Generator
**Location**: `core/identity-hub-core/src/main/java/org/eclipse/edc/identityhub/core/services/verifiablepresentation/generators/JwtPresentationGenerator.java`

**Format**: JWT-based VPs for VC 1.0 JWT credentials
```java
// Generated VP structure (JWT claims)
{
  "iss": "did:example:issuer",      // Participant DID
  "aud": "did:example:verifier",    // Audience (if provided)
  "vp": {
    "type": ["VerifiablePresentation"],
    "verifiableCredential": [...],   // Array of JWT VCs
    "holder": "did:example:holder"
  }
}
```

**Requirements**:
- Must provide audience in additionalData
- Uses participant's signing key for VP proof
- Returns: `String` (signed JWT)

### 2. LDP Presentation Generator  
**Location**: `core/identity-hub-core/src/main/java/org/eclipse/edc/identityhub/core/services/verifiablepresentation/generators/LdpPresentationGenerator.java`

**Format**: JSON-LD VPs for VC 1.0 LD credentials
```json
{
  "@context": [...],
  "id": "urn:uuid:...",
  "type": ["VerifiablePresentation"],
  "holder": "did:example:holder",
  "verifiableCredential": [...],
  "proof": {
    "type": "JsonWebSignature2020",
    "created": "2024-...",
    "verificationMethod": "did:example:holder#key-1",
    "jws": "..."
  }
}
```

**Features**:
- Uses JsonWebSignature2020 by default
- Configurable signature suites
- Returns: `JsonObject`

### 3. JWT Enveloped Presentation Generator
**Location**: `core/identity-hub-core/src/main/java/org/eclipse/edc/identityhub/core/services/verifiablepresentation/generators/JwtEnvelopedPresentationGenerator.java`

**Format**: VC 2.0 JOSE format (newer standard)
- Supports "EnvelopedVerifiableCredential" and "EnvelopedVerifiablePresentation"
- Uses JOSE signing
- Returns: `String` (JOSE token)

## Presentation Creation Registry

### PresentationCreatorRegistry
**Interface**: `spi/verifiable-credential-spi/src/main/java/org/eclipse/edc/identityhub/spi/verifiablecredentials/generator/PresentationCreatorRegistry.java`

**Purpose**: Maps credential formats to appropriate presentation generators

```java
interface PresentationCreatorRegistry {
    void addCreator(PresentationGenerator<?> creator, CredentialFormat format);
    <T> T createPresentation(String participantContextId, 
                           List<VerifiableCredentialContainer> credentials,
                           CredentialFormat format, 
                           Map<String, Object> additionalData);
}
```

### Registration Configuration
**Location**: `core/identity-hub-core/src/main/java/org/eclipse/edc/identityhub/core/CoreServicesExtension.java`

```java
// Generator registration by format
presentationCreatorRegistry.addCreator(new JwtPresentationGenerator(...), CredentialFormat.VC1_0_JWT);
presentationCreatorRegistry.addCreator(new LdpPresentationGenerator(...), CredentialFormat.VC1_0_LD);
presentationCreatorRegistry.addCreator(new JwtEnvelopedPresentationGenerator(...), CredentialFormat.VC2_0_JOSE);
```

## Key Management Integration

### Signing Key Resolution
**Process**:
1. Resolve participant's active signing key pair
2. Use private key alias for signing operations
3. Include public key ID in VP proof
4. Recipient uses public key ID to verify VP

**Key Sources**:
- **Default KeyPair**: Participant's default signing key
- **Group-specific Keys**: Keys for specific use cases
- **DID Document**: Public keys published in DID document

### Proof Generation
**JWT VPs**: 
- JWT signature using participant's private key
- Key ID in JWT header for verification

**LDP VPs**:
- JsonWebSignature2020 proof
- verificationMethod points to DID document key
- JWS signature over canonical JSON-LD

## API Integration

### Presentation API Controller
**Location**: `protocols/dcp/dcp-identityhub/presentation-api/src/main/java/org/eclipse/edc/identityhub/api/verifiablecredential/PresentationApiController.java`

**Endpoint**: `POST /v1/participants/{participantContextId}/presentations`

**Flow**:
1. **Receive PresentationQueryMessage** (DCP protocol)
2. **Validate and authenticate** request token
3. **Resolve credentials** matching query criteria
4. **Generate VP** using VerifiablePresentationService
5. **Return VP** in protocol-specific format

### Query Resolution
**Component**: `CredentialQueryResolver`
- Filters credentials by participant context
- Applies presentation definition constraints
- Enforces scope-based access control
- Returns matching credentials for VP generation

## Multi-Format Handling

### Format Grouping Strategy
**Issue**: Different credential formats cannot be mixed in single VP
**Solution**: Create separate VPs for each format

```java
// Credentials grouped by format
var groups = credentials.stream().collect(Collectors.groupingBy(VerifiableCredentialContainer::format));

// Separate VPs created
if (!jwt11Vcs.isEmpty()) {
    String jwt11Vp = registry.createPresentation(participantContextId, jwt11Vcs, VC1_0_JWT, additionalData);
    vpToken.add(jwt11Vp);
}

if (!ldp11Vcs.isEmpty()) {
    JsonObject ld11Vp = registry.createPresentation(participantContextId, ldp11Vcs, VC1_0_LD, additionalData);
    vpToken.add(ld11Vp);
}
```

### Response Format
**PresentationResponseMessage**:
```java
{
  "presentation": [        // Array of VPs (different formats)
    "eyJ...",             // JWT VP
    { "@context": ... }   // JSON-LD VP  
  ]
}
```

## Security Considerations

### Authentication Requirements
- **Self-Issued Tokens**: Presentation requests must include valid authentication
- **Participant Validation**: Verify participant context exists and is active
- **Scope Enforcement**: Apply issuer scopes to credential filtering

### Cryptographic Proofs
- **Private Key Security**: Keys resolved from secure storage
- **Proof Uniqueness**: Each VP includes unique identifiers and timestamps
- **Algorithm Support**: Configurable signature algorithms per format

### Audience Binding
- **JWT VPs**: Audience claim binds VP to specific verifier
- **Replay Protection**: Timestamps and nonces prevent replay attacks

## Configuration & Extensibility

### Signature Suite Registry
**Component**: `SignatureSuiteRegistry`
- Configurable cryptographic suites for LDP VPs
- Default: JsonWebSignature2020
- Extensible for new signature types

### Token Generation Service
**Component**: `TokenGenerationService`
- JWT creation and signing
- Configurable key resolution
- Support for multiple algorithms

## Key Files

| Component | Location |
|-----------|----------|
| **Core Service** | `core/identity-hub-core/src/main/java/org/eclipse/edc/identityhub/core/services/verifiablepresentation/` |
| **SPI Interface** | `spi/verifiable-credential-spi/src/main/java/org/eclipse/edc/identityhub/spi/verifiablecredentials/generator/` |
| **JWT Generator** | `core/identity-hub-core/.../generators/JwtPresentationGenerator.java` |
| **LDP Generator** | `core/identity-hub-core/.../generators/LdpPresentationGenerator.java` |
| **JOSE Generator** | `core/identity-hub-core/.../generators/JwtEnvelopedPresentationGenerator.java` |
| **Registry Impl** | `core/identity-hub-core/.../PresentationCreatorRegistryImpl.java` |
| **API Controller** | `protocols/dcp/dcp-identityhub/presentation-api/.../PresentationApiController.java` |

## Implementation Notes

### Performance Optimizations
- **Format Grouping**: Avoids incompatible format mixing
- **Lazy Evaluation**: Credentials loaded only when needed
- **Concurrent Processing**: Multiple VPs can be generated in parallel

### Error Handling
- **Invalid Credentials**: Graceful handling of malformed VCs
- **Key Resolution Failures**: Clear error messages for missing keys
- **Format Incompatibility**: Explicit error for unsupported combinations

### Future Enhancements
- **Presentation Definition Support**: Full PD processing (currently logged but ignored)
- **Zero-Knowledge Proofs**: Support for selective disclosure
- **Batch VP Generation**: Optimizations for large credential sets 