# Verifiable Credentials Management

## Overview

The **VerifiableCredentialResource** system manages the complete lifecycle of verifiable credentials, from storage to presentation, with support for multiple credential formats and automatic state management.

## Core Data Model

### VerifiableCredentialResource
**Location**: `spi/verifiable-credential-spi/src/main/java/org/eclipse/edc/identityhub/spi/verifiablecredentials/model/VerifiableCredentialResource.java`

```java
class VerifiableCredentialResource extends IdentityResource {
    // Core identity
    String id;                                    // Unique credential ID
    String participantContextId;                  // Owner participant
    String issuerId;                             // Credential issuer DID
    String holderId;                             // Credential holder DID
    
    // Lifecycle management
    int state;                                   // VcStatus enum value
    Instant timeOfLastStatusUpdate;              // State change timestamp
    Map<String, Object> metadata;               // Additional metadata
    
    // Credential content
    VerifiableCredentialContainer verifiableCredential;  // Actual VC + format
    
    // Policies (deprecated since 0.11.0)
    Policy issuancePolicy;                       // ODRL policy for issuance
    Policy reissuancePolicy;                     // ODRL policy for reissuance
}
```

### VerifiableCredentialContainer
**Wrapper Class**: Contains the actual credential in multiple representations
```java
class VerifiableCredentialContainer {
    Object rawVc;                               // Raw credential (JWT string, JSON object)
    CredentialFormat format;                    // VC1_0_JWT, VC1_0_LD, VC2_0_JOSE
    VerifiableCredential credential;            // Structured representation
}
```

## State Management

### VcStatus Lifecycle
```java
enum VcStatus {
    INITIAL,                // Created but not yet processed
    REQUESTING,             // Request initiated
    REQUESTED,              // Request sent to issuer
    ISSUING,               // Being issued
    ISSUED,                // Successfully issued and stored
    REISSUE_REQUESTING,    // Renewal request initiated
    REISSUE_REQUESTED,     // Renewal request sent
    TERMINATED,            // Lifecycle ended
    EXPIRED,               // Credential expired
    REVOKED,               // Credential revoked
    SUSPENDED,             // Credential suspended
    ERROR                  // Error state
}
```

### Status Operations
```java
// State transition methods
verifiableCredentialResource.suspend();      // Mark as suspended
verifiableCredentialResource.setExpired();   // Mark as expired  
verifiableCredentialResource.revoke();       // Mark as revoked

// Status checking
boolean isRevoked = vcr.isRevoked();
boolean isExpired = vcr.isExpired();
boolean isSuspended = vcr.isSuspended();
```

## Storage Architecture

### CredentialStore Interface
**Location**: `spi/verifiable-credential-spi/src/main/java/org/eclipse/edc/identityhub/spi/verifiablecredentials/store/CredentialStore.java`

```java
interface CredentialStore {
    StoreResult<Void> create(VerifiableCredentialResource credentialResource);
    StoreResult<Collection<VerifiableCredentialResource>> query(QuerySpec querySpec);
    StoreResult<Void> update(VerifiableCredentialResource credentialResource);
    StoreResult<Void> deleteById(String id);
    ServiceResult<VerifiableCredentialResource> findById(String credentialId);
}
```

### SQL Implementation
**Location**: `extensions/store/sql/identity-hub-credentials-store-sql/`

**Schema**: Stores credentials with format-specific handling
```sql
-- Key columns in credential store
id VARCHAR PRIMARY KEY,
participant_context_id VARCHAR NOT NULL,
issuer_id VARCHAR,
holder_id VARCHAR,
vc_state INTEGER,
raw_vc TEXT,                    -- Raw credential (JWT/JSON)
vc_format INTEGER,              -- Format enum index
verifiable_credential JSON,     -- Structured credential
metadata JSON,
issuance_policy JSON,
reissuance_policy JSON
```

## API Layer

### Identity API - Credential Management
**Location**: `extensions/api/identity-api/verifiable-credentials-api/`

**Endpoints**:
```
GET    /api/identity/v1/participants/{id}/credentials/{credentialId}  # Get credential
POST   /api/identity/v1/participants/{id}/credentials                # Add credential
PUT    /api/identity/v1/participants/{id}/credentials                # Update credential
DELETE /api/identity/v1/participants/{id}/credentials/{credentialId} # Delete credential
GET    /api/identity/v1/participants/{id}/credentials?type={type}    # Query by type
```

### Credential Writing Service
**Location**: `core/identity-hub-core/src/main/java/org/eclipse/edc/identityhub/core/services/verifiablecredential/CredentialWriterImpl.java`

**Supported Formats**:
- `VC1_0_JWT`: JWT-based VCs
- `VC1_0_LD`: JSON-LD VCs
- `VC2_0_JOSE`: JOSE-based VCs (newer format)

## Credential Request Management

### HolderCredentialRequest
**Purpose**: Manages credential requests from holders to issuers

**States**:
```java
enum HolderRequestState {
    INITIAL,
    REQUESTED,
    ISSUED,
    ERROR
}
```

### Request Processing
1. **Credential Offer Received** → Create request
2. **Request Sent to Issuer** → Update state
3. **Credential Received** → Store as VerifiableCredentialResource
4. **Status Updates** → Automatic state transitions

## Query & Search

### CredentialQueryResolver
**Purpose**: Resolves credential queries from presentation requests

**Capabilities**:
- Filter by credential type
- Filter by participant context
- Support for presentation definitions
- Scope-based access control

### Query Examples
```java
// Find credentials by type
QuerySpec.Builder.newInstance()
    .filter(new Criterion("verifiableCredential.credential.type", "contains", "DriverLicense"))
    .build();

// Find credentials by participant
QuerySpec.Builder.newInstance()
    .filter(new Criterion("participantContextId", "=", participantId))
    .build();
```

## Credential Verification

### Status Checking
**Service**: `CredentialStatusCheckService`
- Validates credential status lists
- Checks revocation status
- Handles suspension states

### Credential Lookup
**Class**: `CredentialResourceLookup`
- Provides reflection-based property access
- Handles raw JSON credential properties
- Supports complex nested queries

## Integration Points

### Presentation Generation
**Flow**: Credentials → Presentation Service → Verifiable Presentation
- Filters credentials by query
- Groups by format for VP generation
- Applies presentation definitions

### Credential Offers
**Flow**: External Issuer → Credential Offer → Holder Request → Stored Credential
- Automatic processing of credential offers
- Support for multiple credential formats
- Integration with DCP protocol

### Status List Management
**Features**:
- Bitstring-based status lists
- Revocation list management
- Suspension list management
- Automatic status updates

## Key Files

| Component | Location |
|-----------|----------|
| **Core Model** | `spi/verifiable-credential-spi/src/main/java/org/eclipse/edc/identityhub/spi/verifiablecredentials/model/` |
| **Store SPI** | `spi/verifiable-credential-spi/src/main/java/org/eclipse/edc/identityhub/spi/verifiablecredentials/store/` |
| **SQL Store** | `extensions/store/sql/identity-hub-credentials-store-sql/` |
| **API Controllers** | `extensions/api/identity-api/verifiable-credentials-api/` |
| **Core Services** | `core/identity-hub-core/src/main/java/org/eclipse/edc/identityhub/core/services/verifiablecredential/` |
| **Query Resolution** | `spi/verifiable-credential-spi/src/main/java/org/eclipse/edc/identityhub/spi/verifiablecredentials/resolution/` |

## Implementation Notes

### Format Support
- **JWT VCs**: Stored as JWT strings, parsed for querying
- **JSON-LD VCs**: Stored as JSON objects, full LD processing
- **Envelope VCs**: VC 2.0 JOSE format support

### Security Considerations
- All credentials scoped to participant context
- Access control on all operations
- Secure storage of credential metadata
- No private key storage in credential data

### Performance Optimizations
- Indexed queries on common fields
- Lazy loading of credential content
- Efficient status checking
- Batch operations support 