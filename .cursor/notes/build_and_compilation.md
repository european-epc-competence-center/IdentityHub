# IdentityHub Build Guide

## Prerequisites

- **Java 17** (verified with Amazon Corretto 17.0.15)
- **Gradle wrapper** included (`./gradlew`)

## Setup Java 17

```bash
# Install and use Java 17 via SDK Manager
sdk install java 17.0.15-amzn
sdk use java 17.0.15-amzn
java -version  # Should show OpenJDK 17.0.15
```

## Build Commands

```bash
# Full build (151 modules, ~4-5 minutes)
./gradlew build

# Build JAR files
./gradlew :launcher:identityhub:shadowJar       # → launcher/identityhub/build/libs/identity-hub.jar
./gradlew :launcher:issuer-service:shadowJar    # → launcher/issuer-service/build/libs/issuer-service.jar
```

## Other Useful Tasks

```bash
./gradlew clean          # Clean build artifacts
./gradlew test           # Run tests only
./gradlew assemble       # Build without tests
./gradlew javadoc        # Generate documentation
launcher:identityhub:run # Run IdentityHub directly
```

## Project Structure (151 modules)

- `core/` - Business logic and services
- `spi/` - Service Provider Interfaces  
- `extensions/` - Plugin implementations
- `protocols/` - DCP protocol support
- `launcher/` - Executable applications

## Troubleshooting

**Java Version Issues:**
```bash
java -version                    # Verify Java 17 is active
sdk use java 17.0.15-amzn      # Switch to Java 17
./gradlew clean build           # Clean rebuild
```

**Memory Issues:**
```bash
export GRADLE_OPTS="-Xmx4g -XX:MaxMetaspaceSize=1g"
./gradlew build
```

**Note:** JavaDoc warnings are non-fatal and don't affect functionality. 