# Security Architecture & Authentication

## Overview

IdentityHub implements a comprehensive security model with **participant-scoped access control**, **API key authentication**, and **embedded STS (Secure Token Service)** integration. The security architecture ensures complete isolation between participants while supporting both regular users and administrative operations.

## Core Security Concepts

### Service Principal Model
**Definition**: In IdentityHub, a **Service Principal** represents a participant context, not individual users.

```java
// Service Principal = Participant Context ID
// Individual users are NOT first-level concepts in IdentityHub
ServicePrincipal.getName() == ParticipantContext.getParticipantContextId()
```

**Key Distinctions**:
- **Participant Context**: Organization/company-level entity (e.g., BMW, Bosch)
- **Service Principal**: The identity used for API authentication
- **Users**: Individual humans within an organization (not directly modeled)

### Authentication Flow

#### API Key Structure
**Format**: `base64(participantId) + "." + base64(randomString)`

```java
// Example API key structure
String participantId = "BPNL000000000001";
String randomString = generateSecureRandom();
String apiKey = base64(participantId) + "." + base64(randomString);
```

#### Request Authentication Process
1. **Extract API Key**: From `x-api-key` HTTP header
2. **Parse Service Principal ID**: Decode first part of API key
3. **Database Lookup**: Retrieve ParticipantContext by ID
4. **Credential Verification**: Match API key against stored credential
5. **Security Context**: Attach principal to request for controller injection

```java
@POST
@Path("/credentials")
public void addCredential(@Context SecurityContext securityContext) {
    var principal = securityContext.getUserPrincipal();
    // principal.getName() returns participant context ID
}
```

## Authorization Model

### Explicit Resource Authorization
**Pattern**: Authorization happens **after** controller method invocation

```java
@GET
@Path("/{credentialId}")
public VerifiableCredentialResource getCredential(
    @PathParam("credentialId") String id, 
    @Context SecurityContext securityContext) {
    
    // Explicit authorization check
    authorizationService.isAuthorized(securityContext, id, VerifiableCredentialResource.class)
        .orElseThrow(exceptionMapper(VerifiableCredentialResource.class, id));
    
    // Proceed with business logic
}
```

### Resource Lookup Functions
**Mechanism**: Each API module registers a lookup function for its resource type

```java
@Extension(value = NAME)
public class VerifiableCredentialApiExtension implements ServiceExtension {
    @Override
    public void initialize(ServiceExtensionContext context) {
        // Register lookup function for authorization
        authorizationService.addLookupFunction(VerifiableCredentialResource.class, 
            credentialId -> credentialStore.query(
                QuerySpec.Builder.newInstance()
                    .filter(new Criterion("id", "=", credentialId))
                    .build()
            ).orElseThrow().stream().findFirst().orElseThrow());
    }
}
```

### Authorization Logic
```java
public ServiceResult<Void> isAuthorized(Principal principal, String resourceId, Class<?> resourceClass) {
    var lookupFunction = resourceLookupFunctions.get(resourceClass);
    var resource = lookupFunction.apply(resourceId);
    
    // Check resource ownership
    boolean isAuthorized = Objects.equals(
        resource.getParticipantContextId(), 
        principal.getName()
    );
    
    return isAuthorized ? ServiceResult.success() : ServiceResult.unauthorized();
}
```

## Role-Based Access Control

### Built-in Roles
**Admin/Super-user Role**: Pre-defined role with elevated permissions

**Capabilities**:
- Create/delete participant contexts
- Access resources across multiple participants ("on behalf of" operations)
- Administrative operations requiring elevated privileges

**Implementation**: Super-user can perform operations even if they don't technically "own" the resource

### Role Assignment
Roles are assigned at the participant context level through the `roles` field:

```java
class ParticipantContext {
    List<String> roles;  // e.g., ["admin", "user"]
}
```

## API Key Management

### Key Generation
**Automatic Generation**: API keys created automatically when ParticipantContext is created

```java
// Key generation during participant creation
ParticipantContext.Builder.newInstance()
    .participantContextId(manifest.getParticipantId())
    .apiTokenAlias("%s-%s".formatted(participantId, "api-key"))
    .build();
```

### Key Storage
**Vault Integration**: API keys stored securely in vault, never in database

```java
// API key storage pattern
String tokenAlias = participantId + "-api-key";
vault.storeSecret(tokenAlias, generatedApiKey);

// Database only stores the alias reference
participantContext.setApiTokenAlias(tokenAlias);
```

### Key Regeneration
**Endpoint**: `PUT /participants/{id}/regenerate-token`

**Security**: Requires current valid API key to regenerate

```bash
curl -X PUT \
  -H "x-api-key: <current-api-key>" \
  "http://identityhub/api/identity/v1/participants/<participant-id>/regenerate-token"
```

## Secure Token Service (STS) Integration

### Embedded Deployment Model
**Decision**: As of 2025-02-24, STS **must always be embedded** in IdentityHub runtime

**Rationale**:
- STS and IdentityHub are tightly coupled (token format knowledge)
- IdentityHub manages all security material (key pairs, participant contexts)
- Eliminates need for remote STS calls and account synchronization

### STS Account Provisioning
**Automatic Provisioning**: STS accounts created automatically with participant contexts

```java
// STS account creation is internal implementation detail
class StsAccountProvisioner {
    // Creates OAuth2 client credentials for each participant
    void provision(ParticipantContext context) {
        var clientId = context.getParticipantContextId();
        var clientSecret = generateClientSecret();
        // Store in vault, not database
    }
}
```

### Token Issuance
**OAuth2 Client Credentials Flow**: Each participant has dedicated OAuth2 credentials

**Token Structure**: DCP-compliant Self-Issued tokens
- Includes participant identity claims
- Cryptographically signed with participant's key
- Scoped to specific operations and resources

## Security Best Practices

### API Key Security
- **Never expose in logs**: API keys are sensitive secrets
- **Regular rotation**: Automatic regeneration capabilities
- **Secure transmission**: HTTPS-only for API communication
- **Vault storage**: Never store API keys in database

### Access Control
- **Principle of least privilege**: Default deny, explicit grant
- **Resource isolation**: Strict participant context boundaries
- **Admin operations**: Require elevated privileges and explicit authorization

### Network Security
**API Exposure Guidelines**:
- **Hub API (Presentation)**: Can be exposed to internet (with proper network infrastructure)
- **Identity API**: **Internal only** - requires API gateways and network protection
- **Issuer Admin API**: **Administrative access only** - highest security requirements

### Key Management
- **Vault integration**: All secrets stored in secure vault
- **Key rotation**: Automated key lifecycle management
- **Algorithm support**: Configurable cryptographic algorithms
- **Hardware Security Module (HSM)**: Support for hardware-backed keys

## Key Components

| Component | Location | Purpose |
|-----------|----------|---------|
| **API Authentication** | `extensions/api/identityhub-api-authentication/` | Token validation and principal extraction |
| **Authorization Service** | `extensions/api/identityhub-api-authorization/` | Resource-level access control |
| **STS Core** | `extensions/sts/sts-core/` | Token generation and validation |
| **STS Account Provisioner** | `extensions/sts/sts-account-provisioner/` | Automatic account lifecycle |
| **Participant Context Service** | `core/identity-hub-participants/` | Principal management and API key generation |

## Implementation Notes

### Error Handling
- **HTTP 401**: Authentication failures (invalid/missing API key)
- **HTTP 403**: Authorization failures (insufficient permissions)
- **HTTP 404**: Resource not found or access denied (security through obscurity)

### Performance Considerations
- **Cached principals**: Avoid repeated database lookups
- **Efficient authorization**: Optimized resource lookup functions
- **Connection pooling**: Vault and database connection management

### Future Enhancements
- **Multi-factor authentication**: Additional security layers
- **Fine-grained permissions**: More granular RBAC model
- **Audit logging**: Comprehensive security event tracking
- **Zero-trust networking**: Network-level security enhancements 