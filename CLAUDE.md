# CLAUDE.md

This file provides guidance to Claude when working with code in this repository.

## Overview

`plantogether-proto` is a shared Maven library that defines all gRPC service contracts (Protocol Buffer `.proto` files)
for the PlanTogether microservices ecosystem. It compiles `.proto` files into Java gRPC stubs consumed by other
microservices.

**Stack:** Java 21, Maven 3.8.1+, gRPC 1.62+, protobuf 3.x

## Commands

```bash
# Compile .proto files and install stubs to local Maven repo
mvn clean install

# Just compile (generates Java stubs in target/generated-sources/protobuf/java/)
mvn compile

# Verify generated sources
ls target/generated-sources/protobuf/java/com/plantogether/
```

No tests — this is a pure code generation library.

## Architecture

This module is the **single source of truth** for all inter-service gRPC contracts. Consuming services declare:

```xml
<dependency>
    <groupId>com.plantogether</groupId>
    <artifactId>plantogether-proto</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

### Proto file layout

```
src/main/proto/com/plantogether/
├── trip/         → trip_service.proto   (membership checks, member profiles, currency)
├── file/         → file_service.proto   (MinIO presigned URL generation)
├── destination/  → (planned)
├── expense/      → (planned)
├── poll/         → (planned)
└── task/         → (planned)
```

### Defined gRPC services

#### `TripGrpcService` — `com.plantogether.trip.grpc` (server: trip-service port 9081)

| Method | Request | Response | Purpose |
|---|---|---|---|
| `CheckMembership` | `MembershipRequest{trip_id, user_id}` | `MembershipResponse{is_member, role}` | Verify user belongs to a trip (called by all services before any operation) |
| `GetTripMembers` | `TripMembersRequest{trip_id}` | `TripMembersResponse{members[]}` | Retrieve all trip members with profiles |
| `GetUserProfiles` | `UserProfilesRequest{keycloak_ids[]}` | `TripMembersResponse{members[]}` | Bulk profile lookup by Keycloak ID |
| `GetTripCurrency` | `TripCurrencyRequest{trip_id}` | `TripCurrencyResponse{currency}` | Get trip's reference currency (called by expense-service) |

`MemberProfile` message: `user_id`, `display_name`, `avatar_url`, `email`, `role`.
`MembershipResponse.role`: `ORGANIZER` | `PARTICIPANT`.

#### `FileGrpcService` — `com.plantogether.file.grpc` (server: file-service port 9088)

| Method | Purpose |
|---|---|
| `GetPresignedUrl` | Generate a short-lived presigned MinIO URL for upload or download |

### gRPC client usage (per service)

| Service | gRPC client(s) used |
|---|---|
| poll-service | TripGrpcService.CheckMembership |
| destination-service | TripGrpcService.CheckMembership |
| expense-service | TripGrpcService.CheckMembership, TripGrpcService.GetTripCurrency |
| task-service | TripGrpcService.CheckMembership |
| chat-service | TripGrpcService.CheckMembership |
| notification-service | TripGrpcService.GetTripMembers |
| All services | FileGrpcService.GetPresignedUrl |

## Proto Versioning Rules

- **Never** remove, rename, or retype existing fields — this breaks consumers.
- **Never** change a field's type or number.
- Adding new optional fields (with new field numbers) is safe and backward-compatible.
- Use field numbers 1–15 for frequently used fields (single-byte encoding).
- Reserve removed field numbers with `reserved 16;` to prevent accidental reuse.
- Use `buf lint` and `buf breaking` in CI to enforce these rules automatically.

## Adding a New gRPC Service

1. Create a new `.proto` file under `src/main/proto/com/plantogether/<domain>/`.
2. Set `syntax = "proto3";`, `option java_multiple_files = true;`, `option java_package = "com.plantogether.<domain>.grpc";`.
3. Run `mvn clean install`.
4. Add the updated dependency to consuming microservices.
