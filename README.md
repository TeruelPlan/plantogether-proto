# PlanTogether Protocol Buffers (gRPC)

> Définitions Protocol Buffer partagées pour la communication synchrone inter-services

## Rôle

Ce module Maven contient l'ensemble des fichiers `.proto` définissant les contrats gRPC entre les microservices
PlanTogether. Il génère automatiquement les stubs Java (client et serveur) via le plugin `protobuf-maven-plugin`.
Il doit être installé en premier avant tout autre microservice.

## Prérequis et installation

```bash
# À installer en premier, avant plantogether-common et tous les microservices
cd plantogether-proto
mvn clean install
```

## Services définis

### TripGrpcService (`trip_service.proto`)

Exposé par **trip-service** sur le port **9081**. Consommé par tous les autres microservices.

| RPC | Request | Response | Description |
|-----|---------|----------|-------------|
| `CheckMembership` | `MembershipRequest` | `MembershipResponse` | Vérifie si un user est membre d'un trip |
| `GetTripMembers` | `TripMembersRequest` | `TripMembersResponse` | Profils complets des membres |
| `GetUserProfiles` | `UserProfilesRequest` | `TripMembersResponse` | Résolution batch de profils par keycloak_ids |
| `GetTripCurrency` | `TripCurrencyRequest` | `TripCurrencyResponse` | Devise de référence du trip |

Messages clés :

```protobuf
message MembershipResponse {
  bool is_member = 1;
  string role = 2; // ORGANIZER | PARTICIPANT
}

message MemberProfile {
  string user_id = 1;
  string display_name = 2;
  string avatar_url = 3;
  string email = 4;
  string role = 5;
}
```

### FileGrpcService (`file_service.proto`)

Exposé par **file-service** sur le port **9088**. Consommé par les services ayant besoin de presigned URLs.

| RPC | Request | Response | Description |
|-----|---------|----------|-------------|
| `GetPresignedUrl` | `PresignedUrlRequest` | `PresignedUrlResponse` | URL de lecture MinIO signée |

## Structure du module

```
plantogether-proto/
├── src/
│   └── main/
│       └── proto/
│           ├── trip_service.proto      # Contrats TripGrpcService
│           └── file_service.proto      # Contrats FileGrpcService
└── pom.xml                             # Plugin protobuf-maven-plugin, génération stubs Java
```

## Conventions de versioning

Les fichiers `.proto` suivent les règles de compatibilité ascendante :

- **Interdit** : supprimer, renommer ou changer le type d'un champ existant
- **Autorisé** : ajouter de nouveaux champs optionnels avec de nouveaux numéros de champ
- Les numéros de champs supprimés doivent être réservés : `reserved 3;`

La CI vérifie ces règles via `buf lint` et `buf breaking` à chaque PR.

## Dépendances

Ce module est une dépendance de **tous les microservices** qui exposent ou consomment du gRPC :

| Service | Rôle |
|---------|------|
| plantogether-trip-service | Serveur gRPC (9081) |
| plantogether-file-service | Serveur gRPC (9088) |
| plantogether-poll-service | Client gRPC (trip) |
| plantogether-destination-service | Client gRPC (trip, file) |
| plantogether-expense-service | Client gRPC (trip, file) |
| plantogether-task-service | Client gRPC (trip) |
| plantogether-chat-service | Client gRPC (trip) |
| plantogether-notification-service | Client gRPC (trip) |
