# IdentityHub Build and Compilation Guide

## Java Version Requirements

**Required Java Version: 17**

The IdentityHub project requires Java 17, as evidenced by:
- Docker files use `openjdk:17-slim-buster` base image
- Successful compilation achieved with Java 17.0.15 (Amazon Corretto)
- Gradle 8.10 is used, which supports Java 17

## Prerequisites

1. **Java 17**: Install using SDK Manager or your preferred method
2. **Gradle**: Project includes Gradle wrapper (`./gradlew`), so no separate installation needed

## Setting Up Java Version

### Using SDK Manager (Recommended)

```bash
# List available Java versions
sdk list java

# Install Java 17 (Amazon Corretto)
sdk install java 17.0.15-amzn

# Use Java 17 for current session
sdk use java 17.0.15-amzn

# Verify Java version
java -version
```

Expected output:
```
openjdk version "17.0.15" 2025-04-15 LTS
OpenJDK Runtime Environment Corretto-17.0.15.6.1 (build 17.0.15+6-LTS)
OpenJDK 64-Bit Server VM Corretto-17.0.15.6.1 (build 17.0.15+6-LTS, mixed mode, sharing)
```

## Compilation Instructions

### 1. Full Project Build

```bash
./gradlew build
```

This command:
- Compiles all 151 modules in the project
- Runs all tests
- Generates documentation (JavaDoc)
- Performs code quality checks (Checkstyle)
- Build time: ~4-5 minutes on modern hardware

### 2. Building the Main JAR

```bash
./gradlew :launcher:identityhub:shadowJar
```

This creates a fat JAR at:
`launcher/identityhub/build/libs/identity-hub.jar`

### 3. Building the Issuer Service JAR

```bash
./gradlew :launcher:issuer-service:shadowJar
```

This creates a fat JAR at:
`launcher/issuer-service/build/libs/issuer-service.jar`

## Available Gradle Tasks

### Common Build Tasks
- `./gradlew build` - Full build with tests
- `./gradlew assemble` - Build without tests
- `./gradlew test` - Run all tests
- `./gradlew clean` - Clean build artifacts

### Application Tasks
- `launcher:identityhub:run` - Run IdentityHub directly
- `launcher:issuer-service:run` - Run Issuer Service directly
- `launcher:identityhub:runShadow` - Run using shadow JAR
- `launcher:issuer-service:runShadow` - Run Issuer Service using shadow JAR

### Documentation Tasks
- `./gradlew javadoc` - Generate API documentation
- `./gradlew autodoc` - Generate project documentation

## Project Structure

The project uses a modular architecture with 151 modules organized into:

- **Core modules** (`core/`): Business logic and core services
- **SPI modules** (`spi/`): Service Provider Interfaces
- **Extensions** (`extensions/`): Plugin implementations
- **Protocols** (`protocols/`): DCP protocol implementations
- **Launchers** (`launcher/`): Executable applications
- **E2E Tests** (`e2e-tests/`): Integration test suites

## Build Configuration

- **Gradle Version**: 8.10
- **Build Plugin**: Eclipse EDC build plugin
- **Code Quality**: Checkstyle enabled
- **Testing**: JUnit with comprehensive test coverage
- **Packaging**: Shadow plugin for fat JARs

## Troubleshooting

### Java Version Issues
If you encounter compilation errors:
1. Verify Java 17 is active: `java -version`
2. Switch to Java 17: `sdk use java 17.0.15-amzn`
3. Clean and rebuild: `./gradlew clean build`

### Memory Issues
For large builds, you may need to increase Gradle memory:
```bash
export GRADLE_OPTS="-Xmx4g -XX:MaxMetaspaceSize=1g"
./gradlew build
```

### Build Warnings
The build may show JavaDoc warnings for missing comments. These are non-fatal and don't affect functionality.

## Verification

After successful build, verify the JAR files:

```bash
ls -la launcher/identityhub/build/libs/
ls -la launcher/issuer-service/build/libs/
```

Both should contain the respective JAR files ready for deployment. 