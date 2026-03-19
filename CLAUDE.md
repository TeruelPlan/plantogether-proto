# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

`plantogether-proto` is a shared Maven library that defines all gRPC service contracts (Protocol Buffer `.proto` files) for the PlanTogether microservices ecosystem. It compiles `.proto` files into Java gRPC stubs that other microservices consume as a Maven dependency.

**Stack:** Java 21, Maven 3.8.1+, gRPC 1.63.0, protobuf 3.25.3

## Commands

```bash
# Compile .proto files and install to local Maven repo
mvn clean install

# Just compile (generates Java stubs in target/generated-sources/protobuf/java/)
mvn compile

# Verify generated sources
ls target/generated-sources/protobuf/java/com/plantogether/
```

There are no tests in this project — it is purely a code generation library.

## Architecture

This module acts as the **single source of truth** for all inter-service gRPC contracts. Other services (trip-service, destination-service, file-service, chat-service, etc.) import this as a Maven dependency:

```xml
<dependency>
    <groupId>com.plantogether</groupId>
    <artifactId>plantogether-proto</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

### Proto file layout

All `.proto` files live under `src/main/proto/com/plantogether/`, organized by service domain:

```
src/main/proto/com/plantogether/
├── file/         → file_service.proto  (MinIO presigned URL generation)
├── trip/         → trip_service.proto  (trip retrieval, membership checks)
├── destination/  → (planned)
├── expense/      → (planned)
├── poll/         → (planned)
└── task/         → (planned)
```

### Defined services

| Service | Package | Methods |
|---------|---------|---------|
| `FileService` | `com.plantogether.file.grpc` | `GetPresignedUploadUrl`, `GetPresignedDownloadUrl`, `DeleteFile` |
| `TripService` | `com.plantogether.trip.grpc` | `GetTrip`, `IsMember` |

### Build pipeline

The `protobuf-maven-plugin` (v0.6.1) runs `protoc` automatically during `mvn compile`. It uses `os-maven-plugin` to detect the correct `protoc` binary for the current OS. No manual `protoc` installation is required.

Generated stubs appear in `target/generated-sources/protobuf/java/` and are included in the JAR via `mvn install`.

## Proto Versioning Rules

- **Never** remove or rename existing fields — this breaks consumers.
- **Never** change a field's type or number.
- Adding new optional fields (with new field numbers) is safe and backward-compatible.
- Use field numbers 1–15 for frequently used fields (single-byte encoding).
- Reserve removed field numbers with `reserved 16, 17;` to prevent accidental reuse.

## Adding a New Service

1. Create a new `.proto` file under `src/main/proto/com/plantogether/<domain>/`.
2. Set `syntax = "proto3";`, `option java_multiple_files = true;`, and `option java_package = "com.plantogether.<domain>.grpc";`.
3. Run `mvn clean install` to generate and publish the stubs.
4. Add the updated dependency to consuming microservices.
