# PlanTogether Protocol Buffers (gRPC)

> Définitions Protocol Buffer pour la communication inter-services gRPC

## Rôle

Le projet plantogether-proto contient les définitions Protocol Buffer (.proto) qui définissent les contrats de
communication gRPC entre microservices. Les buffers protocolaires permettent une sérialisation efficace et une
validation de type forte à la compilation.

### Fonctionnalités

- **Définitions de services** : Contrats gRPC pour communication inter-service
- **Messages structurés** : Sérialisation binaire efficace
- **Génération de code** : Compilation automatique en Java
- **Versioning** : Evolution des services sans breaking changes
- **Interopérabilité** : Support multi-langage (optionnel)

## Architecture

```
┌─────────────────────────────────────────────┐
│    plantogether-proto Maven Module          │
│                                             │
│  ┌─────────────────────────────────────┐   │
│  │  src/main/proto/                    │   │
│  │                                     │   │
│  │  ├── file/                          │   │
│  │  │   ├── file_service.proto         │   │
│  │  │   └── file_models.proto          │   │
│  │  │                                   │   │
│  │  └── common/                        │   │
│  │      └── common_models.proto        │   │
│  │                                     │   │
│  └─────────────────────────────────────┘   │
│                                             │
│  Build Output:                              │
│  target/generated-sources/protobuf/java/  │
│                                             │
│  ├── FileServiceGrpc.java                  │
│  ├── GeneratePresignedUrlRequest.java     │
│  ├── GeneratePresignedUrlResponse.java    │
│  └── ... (autres classes générées)        │
│                                             │
└─────────────────────────────────────────────┘
         ▲
         │ Dépendance Maven
         │
    ┌────┴────────────────────────────────┐
    │  Microservices gRPC Clients:         │
    │  - trip-service                     │
    │  - destination-service              │
    │  - chat-service                     │
    │  - file-service (gRPC Server)       │
    └─────────────────────────────────────┘
```

## Concepts clés

### Protocol Buffers

Format de sérialisation de données développé par Google. Plus compact et rapide que JSON ou XML.

### gRPC

Framework RPC basé sur HTTP/2 qui utilise Protocol Buffers. Permet la communication synchrone entre services.

### Compilation

Les fichiers .proto sont compilés en classes Java via le plugin protobuf-maven-plugin.

### Services définis

Actuellement, seul le file-service expose une API gRPC pour la génération d'URLs pré-signées MinIO.

## Lancer en local

### Prérequis

- Java 21+
- Maven 3.8.1+
- Protocol Buffer Compiler (protoc) 3.20+ (optionnel, inclus par Maven)

### Compilation

```bash
# Cloner le repository
git clone <repo-url>
cd plantogether-proto

# Compiler les .proto files en Java
mvn clean install

# Les fichiers générés apparaissent dans:
# target/generated-sources/protobuf/java/
```

### Vérification

```bash
# Vérifier que les classes sont générées
ls target/generated-sources/protobuf/java/com/plantogether/

# Doit afficher les fichiers .java générés
```

## Structure des fichiers Proto

### file_service.proto

Définit le service gRPC pour la gestion des fichiers (MinIO).

```protobuf
syntax = "proto3";

package com.plantogether.proto.file;

option java_multiple_files = true;
option java_package = "com.plantogether.proto.file";
option java_outer_classname = "FileServiceProto";

// Requête pour générer une URL pré-signée
message GeneratePresignedUrlRequest {
  string bucket = 1;
  string object_key = 2;
  int32 expiration_seconds = 3;  // Durée de validité de l'URL
}

// Réponse avec l'URL pré-signée
message GeneratePresignedUrlResponse {
  string presigned_url = 1;
  int64 expiration_time = 2;
  bool success = 3;
  string error_message = 4;
}

// Service gRPC pour les fichiers
service FileService {
  // Générer une URL pré-signée pour télécharger/téléverser
  rpc GeneratePresignedUrl(GeneratePresignedUrlRequest)
      returns (GeneratePresignedUrlResponse);
}
```

### file_models.proto

Définit les modèles de données additionnels pour les fichiers.

```protobuf
syntax = "proto3";

package com.plantogether.proto.file;

option java_multiple_files = true;
option java_package = "com.plantogether.proto.file";

// Métadonnées d'un fichier
message FileMetadata {
  string file_id = 1;
  string filename = 2;
  int64 size_bytes = 3;
  string mime_type = 4;
  int64 created_timestamp = 5;
  string uploaded_by = 6;
}

// Informations du bucket MinIO
message BucketInfo {
  string bucket_name = 1;
  int64 created_timestamp = 2;
  repeated FileMetadata files = 3;
}
```

### common_models.proto

Définit les modèles communs utilisés par plusieurs services.

```protobuf
syntax = "proto3";

package com.plantogether.proto.common;

option java_multiple_files = true;
option java_package = "com.plantogether.proto.common";

// Timestamp standard
message Timestamp {
  int64 seconds = 1;
  int32 nanos = 2;
}

// Réponse d'erreur standard
message Error {
  string code = 1;
  string message = 2;
  string details = 3;
}

// Réponse paginée
message Page {
  int32 page_number = 1;
  int32 page_size = 2;
  int64 total_elements = 3;
  int32 total_pages = 4;
}
```

## Configuration

### pom.xml - Plugin Protobuf

```xml

<build>
    <plugins>
        <!-- Plugin Protobuf Maven -->
        <plugin>
            <groupId>org.xolstice.maven.plugins</groupId>
            <artifactId>protobuf-maven-plugin</artifactId>
            <version>0.6.1</version>
            <configuration>
                <!-- Version de protoc -->
                <protocArtifact>com.google.protobuf:protoc:3.24.0:exe:${os.detected.classifier}</protocArtifact>
                <!-- Répertoire source des .proto files -->
                <protoSourceRoot>${project.basedir}/src/main/proto</protoSourceRoot>
                <!-- Répertoire de sortie -->
                <outputDirectory>${project.build.directory}/generated-sources/protobuf/java</outputDirectory>
            </configuration>
            <executions>
                <execution>
                    <goals>
                        <goal>compile</goal>
                        <goal>compile-custom</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>

        <!-- Détection du système d'exploitation pour protoc -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-enforcer-plugin</artifactId>
            <version>3.0.0-M3</version>
            <executions>
                <execution>
                    <id>enforce-os-properties</id>
                    <goals>
                        <goal>enforce</goal>
                    </goals>
                    <configuration>
                        <rules>
                            <requireOS>
                                <family>unix</family>
                            </requireOS>
                        </rules>
                        <fail>false</fail>
                    </configuration>
                </execution>
            </executions>
        </plugin>

        <!-- OS Maven Plugin pour détection automatique -->
        <plugin>
            <groupId>kr.motd.maven</groupId>
            <artifactId>os-maven-plugin</artifactId>
            <version>1.7.1</version>
            <executions>
                <execution>
                    <phase>initialize</phase>
                    <goals>
                        <goal>detect</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

### Dépendances dans pom.xml

```xml

<dependencies>
    <!-- gRPC -->
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-netty-shaded</artifactId>
        <version>1.56.1</version>
    </dependency>

    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-protobuf</artifactId>
        <version>1.56.1</version>
    </dependency>

    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-stub</artifactId>
        <version>1.56.1</version>
    </dependency>

    <!-- Protocol Buffers -->
    <dependency>
        <groupId>com.google.protobuf</groupId>
        <artifactId>protobuf-java</artifactId>
        <version>3.24.0</version>
    </dependency>

    <!-- Pour les tests -->
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-testing</artifactId>
        <version>1.56.1</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

## Structure du projet

```
plantogether-proto/
├── pom.xml
├── src/
│   ├── main/
│   │   └── proto/
│   │       ├── file/
│   │       │   ├── file_service.proto
│   │       │   └── file_models.proto
│   │       └── common/
│   │           └── common_models.proto
│   └── test/
│       └── java/
│           └── com/plantogether/proto/
│               └── ProtoCompilationTest.java
└── target/
    └── generated-sources/
        └── protobuf/
            └── java/
                └── com/plantogether/proto/
```

## Utilisation dans les microservices

### Ajouter la dépendance

```xml

<dependency>
    <groupId>com.plantogether</groupId>
    <artifactId>plantogether-proto</artifactId>
    <version>1.0.0</version>
</dependency>

        <!-- Ajouter également les dépendances gRPC -->
<dependency>
<groupId>io.grpc</groupId>
<artifactId>grpc-netty-shaded</artifactId>
<version>1.56.1</version>
</dependency>

<dependency>
<groupId>io.grpc</groupId>
<artifactId>grpc-stub</artifactId>
<version>1.56.1</version>
</dependency>
```

### Implémenter un serveur gRPC (file-service)

```java
package com.plantogether.file.grpc;

import io.grpc.Server;
import io.grpc.ServerBuilder;
import com.plantogether.proto.file.*;

public class FileGrpcServer extends FileServiceGrpc.FileServiceImplBase {

    private final MinioClient minioClient;

    @Override
    public void generatePresignedUrl(
            GeneratePresignedUrlRequest request,
            io.grpc.stub.StreamObserver<GeneratePresignedUrlResponse> responseObserver
    ) {
        try {
            String presignedUrl = minioClient.getPresignedObjectUrl(
                    request.getBucket(),
                    request.getObjectKey(),
                    request.getExpirationSeconds()
            );

            GeneratePresignedUrlResponse response = GeneratePresignedUrlResponse
                    .newBuilder()
                    .setPresignedUrl(presignedUrl)
                    .setSuccess(true)
                    .setExpirationTime(
                            System.currentTimeMillis() +
                                    (request.getExpirationSeconds() * 1000L)
                    )
                    .build();

            responseObserver.onNext(response);
            responseObserver.onCompleted();

        } catch (Exception e) {
            GeneratePresignedUrlResponse errorResponse =
                    GeneratePresignedUrlResponse
                            .newBuilder()
                            .setSuccess(false)
                            .setErrorMessage(e.getMessage())
                            .build();

            responseObserver.onNext(errorResponse);
            responseObserver.onCompleted();
        }
    }

    public static void main(String[] args) throws Exception {
        Server server = ServerBuilder
                .forPort(9090)  // Port gRPC
                .addService(new FileGrpcServer())
                .build()
                .start();

        System.out.println("gRPC Server started on port 9090");
        server.awaitTermination();
    }
}
```

### Utiliser un client gRPC (trip-service ou autre)

```java
package com.plantogether.trip.client;

import io.grpc.ManagedChannel;
import io.grpc.ManagedChannelBuilder;
import com.plantogether.proto.file.*;

@Component
public class FileServiceGrpcClient {

    private final FileServiceGrpc.FileServiceBlockingStub fileStub;

    public FileServiceGrpcClient() {
        ManagedChannel channel = ManagedChannelBuilder
                .forAddress("localhost", 9090)
                .usePlaintext()
                .build();

        this.fileStub = FileServiceGrpc.newBlockingStub(channel);
    }

    public String generatePresignedUrl(String bucket, String objectKey) {
        GeneratePresignedUrlRequest request =
                GeneratePresignedUrlRequest
                        .newBuilder()
                        .setBucket(bucket)
                        .setObjectKey(objectKey)
                        .setExpirationSeconds(3600)  // 1 heure
                        .build();

        GeneratePresignedUrlResponse response = fileStub.generatePresignedUrl(request);

        if (response.getSuccess()) {
            return response.getPresignedUrl();
        } else {
            throw new RuntimeException(
                    "Failed to generate presigned URL: " + response.getErrorMessage()
            );
        }
    }
}
```

## Bonnes pratiques

### Versioning des messages

```protobuf
// V1 (original)
message FileRequest {
  string filename = 1;
}

// V2 (ajout optionnel - backward compatible)
message FileRequest {
  string filename = 1;
  optional string content_type = 2;  // Nouveau champ
}

// V3 (avec valeur par défaut)
message FileRequest {
  string filename = 1;
  string content_type = 2 [default = "application/octet-stream"];
}
```

### Éviter les changements incompatibles

```protobuf
// BON - Ajouter un nouveau champ optionnel
message FileRequest {
  string filename = 1;
  optional string description = 2;
}

// MAUVAIS - Supprimer un champ
// message FileRequest {
//     string filename = 1;
//     // REMOVED: optional string description = 2;
// }

// MAUVAIS - Changer le type d'un champ
// message FileRequest {
//     int32 filename = 1;  // CHANGED from string
// }
```

### Numérotation des champs

- Utiliser 1-15 pour les champs les plus fréquemment utilisés
- Réserver les numéros pour les champs potentiels : `reserved 16, 17, 18;`

## Dépendances / Prérequis

### Outils externes

- **Protocol Buffer Compiler (protoc)** 3.20+ (téléchargé par Maven)
- **Java 21+**
- **Maven 3.8.1+**

### Dépendances Maven

```xml
<!-- Protocol Buffers -->
<dependency>
    <groupId>com.google.protobuf</groupId>
    <artifactId>protobuf-java</artifactId>
    <version>3.24.0</version>
</dependency>

        <!-- gRPC -->
<dependency>
<groupId>io.grpc</groupId>
<artifactId>grpc-netty-shaded</artifactId>
<version>1.56.1</version>
</dependency>

<dependency>
<groupId>io.grpc</groupId>
<artifactId>grpc-protobuf</artifactId>
<version>1.56.1</version>
</dependency>

<dependency>
<groupId>io.grpc</groupId>
<artifactId>grpc-stub</artifactId>
<version>1.56.1</version>
</dependency>
```

## Troubleshooting

### La compilation protobuf échoue

- Vérifier que protoc est disponible : `protoc --version`
- Vérifier la syntaxe des fichiers .proto
- Consulter les logs Maven

### Les classes générées ne sont pas trouvées

- Vérifier que `mvn clean install` a été exécuté
- Vérifier le classpath du projet
- Rafraîchir le projet dans l'IDE

### Problèmes de version

- Assurez-vous que la version de protobuf-java correspond à la version de protoc
- Version supportée : 3.20+

## Documentation supplémentaire

- [Protocol Buffers Documentation](https://developers.google.com/protocol-buffers)
- [gRPC Java Tutorial](https://grpc.io/docs/languages/java/quickstart/)
- [protobuf-maven-plugin](https://xolstice.github.io/protobuf-maven-plugin/)
- [gRPC Best Practices](https://grpc.io/docs/guides/performance-best-practices/)
