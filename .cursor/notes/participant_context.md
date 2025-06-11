# Participant Context - Multi-Tenancy Architecture

## Overview

**ParticipantContext** is the foundational multi-tenancy mechanism in IdentityHub. It serves as the **unit of management and control** for all identity resources, ensuring complete isolation between different participants in the system.

## Core Concept

```java
// Key class: ParticipantContext
Location: spi/participant-context-spi/src/main/java/org/eclipse/edc/identityhub/spi/participantcontext/model/ParticipantContext.java

- participantContextId: String    // Unique participant identifier (e.g., BPN)
- did: String                     // Associated DID for the participant
- state: ParticipantContextState  // CREATED, ACTIVATED, DEACTIVATED
- roles: List<String>             // Participant roles
- apiTokenAlias: String           // Vault reference for API token
- properties: Map<String, Object> // Additional metadata
- createdAt/lastModified: long    // Timestamps
```

## Resource Isolation

### Identity Resources Scoping
All identity resources are **scoped to a ParticipantContext**:

- **VerifiableCredentialResource**: Credentials owned by participant
- **KeyPairResource**: Cryptographic keys for participant
- **DidResource**: DID documents managed by participant

### Access Control Pattern
```java
// Every API call must validate participant context access
authorizationService.isAuthorized(securityContext, participantId, ParticipantContext.class)

// All queries include participant context filter  
QuerySpec.filter(new Criterion("participantContextId", "=", participantId))
```

## State Management

### Lifecycle States
```java
enum ParticipantContextState {
    CREATED,     // Initial state, not yet operational
    ACTIVATED,   // Fully operational, DID published
    DEACTIVATED  // Disabled, cannot be used
}
```

### Event-Driven Coordination
**Location**: `core/identity-hub-participants/src/main/java/org/eclipse/edc/identityhub/participantcontext/ParticipantContextEventCoordinator.java`

Events trigger automatic resource management:
- **ParticipantContextCreated** → Generate default keypairs, create DID
- **ParticipantContextDeleting** → Cleanup all associated resources

## Key Components

### 1. ParticipantContextService
**Interface**: `spi/participant-context-spi/src/main/java/org/eclipse/edc/identityhub/spi/participantcontext/ParticipantContextService.java`

```java
// Core operations
ServiceResult<CreateParticipantContextResponse> createParticipantContext(ParticipantManifest manifest)
ServiceResult<ParticipantContext> getParticipantContext(String participantContextId)
ServiceResult<Void> deleteParticipantContext(String participantContextId)
ServiceResult<String> regenerateApiToken(String participantContextId)
```

### 2. ParticipantManifest
**Creation Request**: Describes new participant requirements
- participantId, DID, key material, service endpoints
- Roles and additional properties
- Whether to immediately activate

### 3. Storage Layer
**SQL Schema**: `extensions/store/sql/identity-hub-participantcontext-store-sql/src/main/resources/participant-context-schema.sql`

```sql
CREATE TABLE participant_context (
    participant_context_id VARCHAR PRIMARY KEY,
    created_date BIGINT NOT NULL,
    last_modified_date BIGINT,
    state INTEGER NOT NULL,
    api_token_alias VARCHAR NOT NULL,
    did VARCHAR,
    roles JSON,
    properties JSON DEFAULT '{}'
);
```

## API Integration

### Identity API Endpoints
**Location**: `extensions/api/identity-api/participant-context-api/`

```
POST   /api/identity/v1/participants       # Create participant (admin-only)
GET    /api/identity/v1/participants/{id}  # Get participant details
DELETE /api/identity/v1/participants/{id}  # Delete participant (admin-only)
PUT    /api/identity/v1/participants/{id}/regenerate-token  # Regenerate API token
```

### Base64 URL Encoding
Participant IDs are **base64-URL encoded** in API paths for security:
```java
// Helper: ParticipantContextId.onEncoded(encodedId)
// Decodes base64-URL encoded participant ID to human-readable form
```

## Security Model

### Authentication Tokens
- Each participant has a **unique API token** stored in Vault
- Token alias: `{participantId}-sts-client-secret`
- Tokens are **never stored in database**, only in secure Vault

### Authorization Scoping
- API access tokens are **scoped to specific participant context**
- Cannot access resources of other participants
- Admin operations require elevated permissions

## Integration Points

### STS (Secure Token Service)
**Location**: `extensions/sts/`
- Provisions STS accounts for each participant
- Manages OAuth2 client credentials
- Issues access tokens scoped to participant context

### Event Publishing
**Publisher**: `core/identity-hub-participants/src/main/java/org/eclipse/edc/identityhub/participantcontext/ParticipantContextEventPublisher.java`

Events published for external systems:
- `participantcontext.created`
- `participantcontext.updated` 
- `participantcontext.deleting`
- `participantcontext.deleted`

## Implementation Notes

### Resource Cleanup
When deleting a ParticipantContext:
1. **ParticipantContextDeleting** event fired first
2. Event coordinator triggers cleanup of dependent resources
3. Actual deletion from database occurs last
4. **ParticipantContextDeleted** event signals completion

### Transaction Management
All participant operations are **transactional** to ensure consistency across:
- ParticipantContext database record
- Associated DID resources
- KeyPair resources
- Vault token storage

## Key Files

| Component | Location |
|-----------|----------|
| **Core Model** | `spi/participant-context-spi/src/main/java/org/eclipse/edc/identityhub/spi/participantcontext/model/` |
| **Service Interface** | `spi/participant-context-spi/src/main/java/org/eclipse/edc/identityhub/spi/participantcontext/ParticipantContextService.java` |
| **Implementation** | `core/identity-hub-participants/src/main/java/org/eclipse/edc/identityhub/participantcontext/` |
| **Storage** | `extensions/store/sql/identity-hub-participantcontext-store-sql/` |
| **API** | `extensions/api/identity-api/participant-context-api/` |
| **Events** | `spi/participant-context-spi/src/main/java/org/eclipse/edc/identityhub/spi/participantcontext/events/` | 