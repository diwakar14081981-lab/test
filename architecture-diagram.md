# Dental Claims Processing System - Architecture Diagram

## High-Level Architecture Overview

```mermaid
graph TB
    subgraph "Client Layer"
        CLIENT[REST Client]
    end

    subgraph "Security & Filtering Layer"
        HEALTH[Health Check Filter]
        JWT[Stargate JWT Filter]
    end

    subgraph "Presentation Layer"
        CONTROLLER[ClaimSubmissionRestController<br/>POST /claim-submissions<br/>PUT /prices<br/>PUT /{claimId}]
    end

    subgraph "Service Layer"
        VALIDATOR[Request Validator]
        FACADE[DentalFacetsService<br/>Facade Pattern]
        BROKER[ClaimSubmissionBrokerEngine<br/>Integration Orchestrator]
    end

    subgraph "Data Transformation Layer"
        MAPPER1[ClaimSubmission<br/>MapperDao]
        MAPPER2[ClaimProcess<br/>MapperDao]
        MAPPER3[ClaimUpdate<br/>MapperDao]
    end

    subgraph "External Integration Layer"
        SOAP1[ClaimSubmission_v20<br/>SOAP Service]
        SOAP2[ClaimProcess_v20<br/>SOAP Service]
        SOAP3[ClaimUpdate_v15<br/>SOAP Service]
    end

    subgraph "Data Access Layer"
        REPO1[DisallowExplanation<br/>Repository]
        REPO2[ProcedureCode<br/>Repository]
    end

    subgraph "Data Layer"
        DB[(MS SQL Server<br/>CMC_EXCD_EXPL_CD<br/>CMC_DPDS_DESC)]
    end

    subgraph "Cross-Cutting Concerns"
        LOGGING[Logging Service<br/>+ Splunk Integration]
        EXCEPTION[Global Exception<br/>Handler]
    end

    subgraph "External Systems"
        TRIZETTO[Trizetto Facets FXI<br/>Legacy SOAP Services]
    end

    CLIENT -->|HTTPS/JSON| HEALTH
    HEALTH --> JWT
    JWT -->|Authenticated| CONTROLLER
    CONTROLLER --> VALIDATOR
    VALIDATOR -->|Valid Request| FACADE
    FACADE --> BROKER

    BROKER --> MAPPER1
    BROKER --> MAPPER2
    BROKER --> MAPPER3

    MAPPER1 -->|REST to SOAP| SOAP1
    MAPPER2 -->|REST to SOAP| SOAP2
    MAPPER3 -->|REST to SOAP| SOAP3

    SOAP1 -->|JAX-WS/SSL| TRIZETTO
    SOAP2 -->|JAX-WS/SSL| TRIZETTO
    SOAP3 -->|JAX-WS/SSL| TRIZETTO

    TRIZETTO -->|SOAP Response| SOAP1
    TRIZETTO -->|SOAP Response| SOAP2
    TRIZETTO -->|SOAP Response| SOAP3

    BROKER --> REPO1
    BROKER --> REPO2

    REPO1 -->|JPA/Hibernate| DB
    REPO2 -->|JPA/Hibernate| DB

    CONTROLLER -.->|Logs| LOGGING
    BROKER -.->|Logs| LOGGING
    CONTROLLER -.->|Errors| EXCEPTION
    BROKER -.->|Errors| EXCEPTION

    LOGGING -.->|Aggregates| SPLUNK[Splunk]

    style CLIENT fill:#e1f5ff
    style CONTROLLER fill:#bbdefb
    style FACADE fill:#90caf9
    style BROKER fill:#64b5f6
    style TRIZETTO fill:#fff9c4
    style DB fill:#c8e6c9
    style LOGGING fill:#ffccbc
    style EXCEPTION fill:#ffccbc
    style JWT fill:#f8bbd0
```

## Detailed Layered Architecture

```mermaid
graph LR
    subgraph "Layer 1: Presentation"
        A[REST Controllers<br/>ClaimSubmissionRestController]
    end

    subgraph "Layer 2: Business Logic"
        B[Facade Service<br/>DentalFacetsServiceImpl]
        C[Broker Engine<br/>ClaimSubmissionBrokerEngineImpl]
    end

    subgraph "Layer 3: Integration"
        D[Mapper DAOs<br/>Submit/Process/Update]
        E[SOAP Clients<br/>JAX-WS]
    end

    subgraph "Layer 4: Data Access"
        F[JPA Repositories<br/>Spring Data]
    end

    subgraph "Layer 5: Domain"
        G[Entities & DTOs<br/>XSD Generated Objects]
    end

    A --> B
    B --> C
    C --> D
    D --> E
    C --> F
    F --> G
    E --> G

    style A fill:#e3f2fd
    style B fill:#bbdefb
    style C fill:#90caf9
    style D fill:#64b5f6
    style E fill:#42a5f5
    style F fill:#a5d6a7
    style G fill:#81c784
```

## Data Flow Diagram

```mermaid
sequenceDiagram
    participant Client
    participant Controller
    participant Validator
    participant Facade
    participant BrokerEngine
    participant Mapper
    participant SOAPClient
    participant Trizetto
    participant Repository
    participant Database

    Client->>Controller: POST /claim-submissions (JSON)
    Controller->>Validator: Validate Request
    Validator-->>Controller: Validation Result
    Controller->>Facade: Submit Claim
    Facade->>BrokerEngine: Process Submission
    BrokerEngine->>Mapper: Transform REST to SOAP
    Mapper-->>BrokerEngine: FXI Request Object
    BrokerEngine->>SOAPClient: Invoke SOAP Service
    SOAPClient->>Trizetto: Submit Claim (XML)
    Trizetto-->>SOAPClient: SOAP Response
    SOAPClient-->>BrokerEngine: FXI Response Object
    BrokerEngine->>Mapper: Transform SOAP to REST
    BrokerEngine->>Repository: Enrich with DB Data
    Repository->>Database: Fetch Procedure Codes
    Database-->>Repository: Descriptions
    Repository-->>BrokerEngine: Enriched Data
    BrokerEngine-->>Facade: Enhanced Response
    Facade-->>Controller: ClaimSubmissionResponse
    Controller-->>Client: HTTP 200 (JSON)
```

## Integration Architecture

```mermaid
graph TB
    subgraph "UHG Dental Claims API"
        API[REST API<br/>Spring Boot 3.4.5]
    end

    subgraph "Transformation Layer"
        TRANSFORM[JSON ↔ XML<br/>Mapper DAOs]
    end

    subgraph "Trizetto Facets FXI"
        FXI1[ClaimSubmission_v20<br/>FaWsvcInpClaimSubmission_v20.asmx]
        FXI2[ClaimProcess_v20<br/>FaWsvcInpClaimProcess_v20.asmx]
        FXI3[ClaimUpdate_v15<br/>FaWsvcInpClaimUpdate_v15.asmx]
    end

    subgraph "Enrichment Database"
        SQLDB[(MS SQL Server<br/>CMC Tables)]
    end

    subgraph "Observability"
        SPLUNK[Splunk<br/>Log Aggregation]
    end

    API <-->|REST/JSON| TRANSFORM
    TRANSFORM <-->|SOAP/XML<br/>SSL/TLS| FXI1
    TRANSFORM <-->|SOAP/XML<br/>SSL/TLS| FXI2
    TRANSFORM <-->|SOAP/XML<br/>SSL/TLS| FXI3
    API <-->|JPA/Hibernate<br/>HikariCP| SQLDB
    API -.->|Logs| SPLUNK

    style API fill:#4fc3f7
    style TRANSFORM fill:#fff59d
    style FXI1 fill:#ffb74d
    style FXI2 fill:#ffb74d
    style FXI3 fill:#ffb74d
    style SQLDB fill:#81c784
    style SPLUNK fill:#f06292
```

## Deployment Architecture

```mermaid
graph TB
    subgraph "Kubernetes Cluster"
        subgraph "Pod"
            CONTAINER[Docker Container<br/>OpenJDK 17<br/>Spring Boot App]
            VOLUME[PersistentVolume<br/>/ebiz mount]
        end

        SERVICE[Kubernetes Service<br/>ClusterIP]
        PROBES[Health Probes<br/>Startup/Readiness/Liveness]
    end

    subgraph "Build Pipeline"
        JENKINS[Jenkins CI/CD]
        MAVEN[Maven Build<br/>JaCoCo Coverage]
        ARTIFACTORY[Artifactory<br/>Artifact Repository]
    end

    subgraph "External Dependencies"
        SQLSERVER[(MS SQL Server)]
        FACETS[Trizetto Facets<br/>SOAP Services]
        SPLUNK_EXT[Splunk]
    end

    JENKINS --> MAVEN
    MAVEN --> ARTIFACTORY
    ARTIFACTORY --> CONTAINER

    SERVICE --> CONTAINER
    CONTAINER --> VOLUME
    PROBES -.-> CONTAINER

    CONTAINER <-->|JDBC| SQLSERVER
    CONTAINER <-->|HTTPS| FACETS
    CONTAINER -.->|Logs| SPLUNK_EXT

    style CONTAINER fill:#4fc3f7
    style SERVICE fill:#81c784
    style JENKINS fill:#ce93d8
    style SQLSERVER fill:#a5d6a7
    style FACETS fill:#ffb74d
```

## Technology Stack

```mermaid
graph LR
    subgraph "Application Layer"
        SPRING[Spring Boot 3.4.5<br/>Java 17]
        MVC[Spring MVC<br/>REST]
    end

    subgraph "Persistence Layer"
        JPA[Spring Data JPA<br/>Hibernate]
        HIKARI[HikariCP<br/>Connection Pool]
    end

    subgraph "Integration Layer"
        JAXWS[JAX-WS<br/>SOAP Client]
        SSL[SSL/TLS<br/>Custom Truststore]
    end

    subgraph "Security Layer"
        JWT_SEC[Stargate JWT<br/>Authentication]
        SEC_CTX[Security Context<br/>K8s Hardening]
    end

    subgraph "Observability Layer"
        LOGBACK[Logback/SLF4J<br/>Logging]
        SPLUNK_TECH[Splunk<br/>Log Aggregation]
    end

    subgraph "Testing Layer"
        JUNIT[JUnit 4<br/>Mockito]
        JACOCO[JaCoCo<br/>Code Coverage]
    end

    SPRING --> MVC
    SPRING --> JPA
    MVC --> JAXWS
    JPA --> HIKARI
    JAXWS --> SSL
    MVC --> JWT_SEC
    JWT_SEC --> SEC_CTX
    SPRING --> LOGBACK
    LOGBACK --> SPLUNK_TECH
    SPRING --> JUNIT
    JUNIT --> JACOCO

    style SPRING fill:#6db33f
    style JPA fill:#59666c
    style JAXWS fill:#ffd966
    style JWT_SEC fill:#f48fb1
    style LOGBACK fill:#ff7043
    style JUNIT fill:#25a162
```

## Design Patterns Used

```mermaid
mindmap
  root((Design Patterns))
    Structural
      Layered Architecture
      Facade Pattern
      Adapter Pattern
    Behavioral
      Strategy Pattern
      Chain of Responsibility
    Creational
      Dependency Injection
      Builder Pattern
    Integration
      Repository Pattern
      DTO Pattern
      Mapper Pattern
```

## Key Architectural Characteristics

| Characteristic | Implementation |
|---|---|
| **Scalability** | Stateless design, Kubernetes deployment, connection pooling |
| **Security** | JWT authentication, SSL/TLS, security context hardening |
| **Reliability** | Comprehensive error handling, health checks, validation |
| **Observability** | Multi-level logging, Splunk integration, request/response tracing |
| **Maintainability** | Clear layer separation, dependency injection, interface-based design |
| **Performance** | HikariCP connection pooling, efficient data enrichment |
| **Integration** | Adapter pattern for SOAP integration, flexible transformation layer |

---

**Architecture Style:** Layered/N-Tier with Service-Oriented Architecture (SOA)
**Framework:** Spring Boot 3.4.5 (Java 17)
**Deployment:** Kubernetes + Docker
**Primary Use Case:** Dental Claims Processing & Integration with Legacy SOAP Systems
