# Architecture Decision Records (ADRs)

## Descripción

Este directorio contiene los Architecture Decision Records (ADRs) para **ArtChain Auction Platform**. Los ADRs documentan las decisiones arquitectónicas significativas tomadas durante el diseño y desarrollo del sistema.

## ¿Qué es un ADR?

Un Architecture Decision Record (ADR) es un documento que captura una decisión arquitectónica importante junto con su contexto y consecuencias. Los ADRs ayudan a:

- Documentar el razonamiento detrás de decisiones técnicas
- Comunicar decisiones al equipo y futuros desarrolladores
- Evitar re-discutir decisiones ya tomadas
- Aprender de decisiones pasadas

## Formato

Cada ADR sigue una estructura consistente:

1. **Estado:** Propuesto | Aceptado | Rechazado | Deprecado
2. **Contexto:** El problema o necesidad que origina la decisión
3. **Decisión:** Qué se decidió y por qué
4. **Consecuencias:** Impactos positivos, negativos y neutrales
5. **Alternativas Consideradas:** Otras opciones evaluadas y razones para rechazarlas

## Índice de ADRs

### Arquitectura y Diseño del Sistema

- **[ADR-001: Arquitectura de Microservicios](./ADR-001-microservices-architecture.md)** ✅ Aceptado
  - Decisión de adoptar arquitectura de microservicios distribuidos con 7 servicios core
  - Event-driven architecture con CQRS
  - Comunicación REST (externa), gRPC (interna), SQS (asíncrona)
  - **Alternativas:** Monolito modular, serverless, híbrido

### Stack Tecnológico

- **[ADR-002: Stack Tecnológico Multi-Lenguaje](./ADR-002-tech-stack.md)** ✅ Aceptado
  - **Backend:** Node.js 20 LTS (TypeScript) + Java 21 (Spring Boot)
  - **Frontend:** React 18+ con TypeScript
  - Node.js para servicios I/O-intensive, Java para compute-intensive
  - **Alternativas:** Solo Node.js, solo Java, Go, Python

- **[ADR-008: Redux Toolkit para State Management](./ADR-008-redux-state-management.md)** ✅ Aceptado
  - Redux Toolkit para UI state y autenticación
  - React Query (TanStack Query) para server state
  - WebSocket integration con Redux middleware
  - **Alternativas:** Context API, Zustand, MobX, Recoil

### Datos y Persistencia

- **[ADR-003: DynamoDB como Base de Datos Principal](./ADR-003-dynamodb-database-strategy.md)** ✅ Aceptado
  - DynamoDB (on-demand) para User, Auction, Blockchain, Notification services
  - ElastiCache (Redis) para caching y sessions
  - OpenSearch para búsqueda full-text
  - Estrategia multi-database optimizada por caso de uso
  - **Alternativas:** PostgreSQL (RDS/Aurora), MongoDB Atlas, Cassandra, FaunaDB

### Infraestructura

- **[ADR-004: AWS como Cloud Provider](./ADR-004-aws-cloud-provider.md)** ✅ Aceptado
  - AWS como cloud provider principal
  - Amazon EKS (Kubernetes) para orquestación de contenedores
  - Multi-región: US-EAST-1 (primaria), EU-WEST-1, AP-SOUTHEAST-1
  - Servicios managed: DynamoDB, S3, SQS, CloudFront, Route 53
  - **Alternativas:** GCP, Azure, multi-cloud, serverless, bare metal

### Seguridad y Autenticación

- **[ADR-005: OAuth 2.0 + JWT para Autenticación](./ADR-005-oauth-jwt-authentication.md)** ✅ Aceptado
  - OAuth 2.0 con JWT (RS256) para autenticación
  - RBAC (Role-Based Access Control) para autorización
  - 4 roles: Buyer, Seller, Admin, Support
  - Access tokens (15 min) + Refresh tokens (7 días)
  - Social login (Google, Apple)
  - MFA para transacciones >$10,000
  - **Alternativas:** Session-based, API keys, Auth0/Okta, Paseto, magic links

### Comunicación y APIs

- **[ADR-006: Estrategia de API Híbrida](./ADR-006-api-strategy-rest-grpc.md)** ✅ Aceptado
  - **REST (JSON):** Comunicación externa (cliente ↔ backend)
  - **gRPC (Protocol Buffers):** Comunicación interna (servicio ↔ servicio)
  - **WebSockets (Socket.io):** Actualizaciones en tiempo real
  - **SQS:** Messaging asíncrono desacoplado
  - **Alternativas:** Solo REST, GraphQL, solo gRPC, SSE, Kafka

- **[ADR-010: Event-Driven Architecture con CQRS](./ADR-010-event-driven-cqrs.md)** ✅ Aceptado
  - Event-driven architecture para desacoplamiento
  - CQRS (Command Query Responsibility Segregation) para Auction Service
  - Event Sourcing para aggregates complejos
  - DynamoDB Streams + SQS para event bus
  - Projections (read models) desnormalizados
  - Saga pattern para transacciones distribuidas
  - **Alternativas:** CRUD simple, solo event-driven, event sourcing puro

### Blockchain

- **[ADR-009: Polygon como Blockchain Principal](./ADR-009-polygon-blockchain.md)** ✅ Aceptado
  - Polygon PoS (Layer 2) para transacciones principales
  - Costo: ~$0.01-0.05 por tx (vs $5-50+ Ethereum)
  - Velocidad: 2 segundos por bloque, <30s finality
  - Smart contracts: Solidity 0.8.x con OpenZeppelin
  - Ethereum Mainnet solo para artículos ultra alto valor (>$1M)
  - IPFS (Pinata/Infura) para metadata storage
  - **Alternativas:** Ethereum Mainnet, Solana, Avalanche, BSC, Flow, Hyperledger

### Calidad y Testing

- **[ADR-007: Estrategia de Testing Multi-Layer](./ADR-007-testing-strategy.md)** ✅ Aceptado
  - Pirámide de testing: 60% unit, 30% integration, 10% E2E
  - **Backend:** Jest (Node.js), JUnit 5 + Mockito (Java)
  - **Frontend:** Jest + React Testing Library
  - **Integration:** Testcontainers (DynamoDB Local, PostgreSQL)
  - **E2E:** Playwright
  - **Contract:** Pact (consumer-driven)
  - **Performance:** k6 (load testing)
  - **Security:** OWASP ZAP, Snyk
  - **Smart Contracts:** Hardhat + Chai
  - Code coverage target: >80%
  - **Alternativas:** Cypress, Mocha, otros frameworks

## Resumen de Decisiones Clave

### Stack Completo

```
┌─────────────────────────────────────────────────────────────┐
│                      FRONTEND                                │
│  React 18 + TypeScript + Redux Toolkit + React Query        │
│  Vite + Tailwind CSS + Socket.io-client                     │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼ REST API / WebSockets
┌─────────────────────────────────────────────────────────────┐
│                    API GATEWAY (Node.js)                     │
│  Rate limiting, Auth, CORS, Request routing                  │
└───┬─────────────────────────────────────────────────────┬───┘
    │                                                       │
    ▼ gRPC                                          gRPC   ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ User Service │  │Auction Service│  │ Blockchain  │  │ Notification │
│  (Node.js)   │  │   (Java)      │  │  Service    │  │   Service    │
│              │  │               │  │ (Node.js)   │  │  (Node.js)   │
└──────┬───────┘  └──────┬────────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                   │                 │
       │                 │ Events (SQS)      │                 │
       └─────────────────┴───────────────────┴─────────────────┘
                            │
                            ▼
         ┌──────────────────────────────────────────┐
         │         DATA LAYER                        │
         │  - DynamoDB (primary)                     │
         │  - ElastiCache Redis (cache)              │
         │  - OpenSearch (search)                    │
         │  - S3 (storage)                           │
         │  - Polygon Blockchain (certification)     │
         └──────────────────────────────────────────┘
                            │
                            ▼
         ┌──────────────────────────────────────────┐
         │     INFRASTRUCTURE (AWS)                  │
         │  - EKS (Kubernetes)                       │
         │  - CloudFront (CDN)                       │
         │  - Route 53 (DNS)                         │
         │  - ALB (Load Balancer)                    │
         │  - Multi-region (US, EU, APAC)            │
         └──────────────────────────────────────────┘
```

### Principios Arquitectónicos

1. **Microservices:** Servicios pequeños, independientes, con responsabilidad única
2. **Event-Driven:** Desacoplamiento vía eventos (SQS, DynamoDB Streams)
3. **CQRS:** Separación de lecturas y escrituras para escalabilidad
4. **API Gateway Pattern:** Punto de entrada único para clientes
5. **Database per Service:** Cada servicio tiene su propio schema
6. **Cloud-Native:** Aprovecha servicios managed de AWS
7. **Security in Depth:** OAuth 2.0, JWT, WAF, encryption at rest/transit
8. **Observability:** Logging, metrics, tracing (CloudWatch, X-Ray)

### Métricas de Éxito Técnico

| Métrica | Target | ADR Relacionado |
|---------|--------|-----------------|
| Uptime | 99.9% | ADR-004 (AWS) |
| Latency (p95) | <200ms | ADR-001 (Microservices), ADR-003 (DynamoDB) |
| Latency (p99) | <500ms | ADR-001, ADR-003 |
| Confirmación de puja | <2s | ADR-009 (Polygon), ADR-010 (Event-driven) |
| Usuarios concurrentes | 10,000+ | ADR-004 (EKS), ADR-003 (DynamoDB) |
| Code coverage | >80% | ADR-007 (Testing) |
| Deploy frequency | >1/día | ADR-004 (CI/CD) |
| MTTR | <30 min | ADR-001 (Resilience) |

## Proceso para Nuevos ADRs

### Cuándo Crear un ADR

Crea un ADR cuando:
- Tomas una decisión arquitectónica significativa
- La decisión tiene impacto a largo plazo
- La decisión afecta a múltiples equipos/servicios
- La decisión tiene trade-offs importantes
- La decisión es difícil de revertir

### Cómo Crear un ADR

1. **Copia el template:** Usa un ADR existente como plantilla
2. **Numbering:** Usa el siguiente número disponible (ADR-011, ADR-012, etc.)
3. **Filename:** `ADR-XXX-short-descriptive-title.md`
4. **Estado inicial:** Comienza con "Propuesto"
5. **Discusión:** Comparte con el equipo para feedback
6. **Decisión:** Cambia estado a "Aceptado" cuando se apruebe
7. **Actualiza este README:** Agrega el nuevo ADR al índice

### Template

```markdown
# ADR-XXX: [Título de la Decisión]

## Estado
[Propuesto | Aceptado | Rechazado | Deprecado]

## Contexto
[Descripción del problema, requisitos del PRD, factores técnicos, restricciones]

## Decisión
[Qué se decidió y por qué]

## Consecuencias
### Positivas
- [Beneficios]

### Negativas
- [Trade-offs y desventajas]

### Neutrales
- [Cambios necesarios o impactos generales]

## Alternativas Consideradas
[Otras opciones evaluadas y por qué fueron descartadas]

## Referencias
[Enlaces a documentación, PRD, etc.]

## Historial
- YYYY-MM-DD: Decisión inicial (Estado)
```

## Recursos

### Documentación Relacionada

- **[PRD Analysis](../00-prd-analysis.md):** Análisis completo del Product Requirements Document
- **[PRD Original](../../prd-example/bid-auctions-prd.md):** Product Requirements Document de ArtChain
- **API Documentation:** `/docs/api/` (a crear)
- **Architecture Diagrams:** `/docs/architecture/` (a crear)

### Lecturas Recomendadas

- [Architecture Decision Records (ADR) by Michael Nygard](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions)
- [Microservices Patterns by Chris Richardson](https://microservices.io/patterns/index.html)
- [Building Microservices by Sam Newman](https://www.oreilly.com/library/view/building-microservices-2nd/9781492034018/)
- [Domain-Driven Design by Eric Evans](https://www.domainlanguage.com/ddd/)
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)

### Tools y Frameworks

#### Backend
- [NestJS](https://nestjs.com/) - Node.js framework
- [Spring Boot](https://spring.io/projects/spring-boot) - Java framework
- [gRPC](https://grpc.io/) - RPC framework
- [Socket.io](https://socket.io/) - WebSocket library

#### Frontend
- [React](https://react.dev/) - UI library
- [Redux Toolkit](https://redux-toolkit.js.org/) - State management
- [React Query](https://tanstack.com/query/latest) - Server state
- [Tailwind CSS](https://tailwindcss.com/) - Styling

#### Blockchain
- [Ethers.js](https://docs.ethers.org/) - Ethereum library
- [Hardhat](https://hardhat.org/) - Smart contract development
- [OpenZeppelin](https://www.openzeppelin.com/contracts) - Smart contract libraries
- [Polygon](https://polygon.technology/) - Layer 2 blockchain

#### Testing
- [Jest](https://jestjs.io/) - JavaScript testing
- [JUnit 5](https://junit.org/junit5/) - Java testing
- [Playwright](https://playwright.dev/) - E2E testing
- [k6](https://k6.io/) - Load testing

#### Infrastructure
- [Terraform](https://www.terraform.io/) - Infrastructure as Code
- [Kubernetes](https://kubernetes.io/) - Container orchestration
- [Docker](https://www.docker.com/) - Containerization

## Historial de Cambios

| Fecha | Versión | Cambios |
|-------|---------|---------|
| 2025-11-07 | 1.0 | Creación inicial con 10 ADRs |

## Contacto

Para preguntas o sugerencias sobre las decisiones arquitectónicas, contacta al equipo de arquitectura.

---

**Última actualización:** 7 de Noviembre, 2025
**Versión:** 1.0
**Estado del Proyecto:** Fase 1 - MVP
