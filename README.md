# PlanTogether Protocol Buffers (gRPC)

> Shared Protocol Buffer definitions for synchronous inter-service communication

## Purpose

This Maven module contains all `.proto` files defining gRPC contracts between PlanTogether microservices. It automatically generates Java stubs (client and server) via the `protobuf-maven-plugin`. It must be installed first before any other microservice.

## Prerequisites and installation

```bash
# Install first, before plantogether-common and all microservices
cd plantogether-proto
mvn clean install
```

## Defined Services

### TripService (`trip_service.proto`)

Exposed by **trip-service** on port **9081**. Consumed by all other microservices.

| RPC | Request | Response | Description |
|-----|---------|----------|-------------|
| `IsMember` | `IsMemberRequest{trip_id, device_id}` | `IsMemberResponse{is_member, role}` | Check if a device is a member of a trip |
| `GetTripMembers` | `GetTripMembersRequest{trip_id}` | `GetTripMembersResponse{members[]}` | Get all trip members with display names |
| `GetTripCurrency` | `GetTripCurrencyRequest{trip_id}` | `GetTripCurrencyResponse{currency_code}` | Get the trip's reference currency |
| `GetTrip` | `GetTripRequest{trip_id}` | `TripResponse{trip_id, name, ...}` | Get trip details |

Key messages:

```protobuf
message IsMemberResponse {
  bool is_member = 1;
  string role = 2; // ORGANIZER | PARTICIPANT
}

message TripMemberProto {
  string device_id = 1;
  string role = 2;
  string display_name = 3;
}
```

### FileService (`file_service.proto`)

Exposed by **file-service** on port **9088**. Consumed by services needing presigned URLs.

| RPC | Request | Response | Description |
|-----|---------|----------|-------------|
| `GetPresignedUrl` | `PresignedUrlRequest` | `PresignedUrlResponse` | MinIO signed read URL |

## Module Structure

```
plantogether-proto/
├── src/
│   └── main/
│       └── proto/
│           └── com/plantogether/
│               ├── trip/trip_service.proto    # TripService contracts
│               └── file/file_service.proto    # FileService contracts
└── pom.xml                                    # protobuf-maven-plugin, Java stub generation
```

## Versioning Rules

Proto files follow backward compatibility rules:

- **Forbidden**: removing, renaming, or changing the type of an existing field
- **Allowed**: adding new optional fields with new field numbers
- Removed field numbers must be reserved: `reserved 3;`

CI enforces these rules via `buf lint` and `buf breaking` on every PR.

## Dependencies

This module is a dependency of **all microservices** that expose or consume gRPC:

| Service | Role |
|---------|------|
| plantogether-trip-service | gRPC server (9081) |
| plantogether-file-service | gRPC server (9088) |
| plantogether-poll-service | gRPC client (trip) |
| plantogether-destination-service | gRPC client (trip, file) |
| plantogether-expense-service | gRPC client (trip, file) |
| plantogether-task-service | gRPC client (trip) |
| plantogether-chat-service | gRPC client (trip) |
| plantogether-notification-service | gRPC client (trip) |
