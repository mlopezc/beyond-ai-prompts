# ADR-001: Arquitectura de Microservicios

## Estado
Aceptado

## Contexto

ArtChain Auction Platform necesita soportar 10,000 usuarios concurrentes con alta disponibilidad (99.9%) y escalar globalmente a múltiples regiones (América, EMEA, APAC). El sistema tiene varios dominios de negocio claramente diferenciados:

**Requisitos del PRD relacionados:**
- RF-001 a RF-002: Autenticación y gestión de usuarios
- RF-003 a RF-005: Sistema de subastas en tiempo real
- RF-006 a RF-007: Integración blockchain
- RF-008 a RF-009: Notificaciones multi-canal
- RF-010 a RF-011: Búsqueda y recomendaciones
- RF-012: Procesamiento de pagos

**Factores técnicos:**
- Necesidad de escalar componentes independientemente (ej. auction service durante picos)
- Diferentes requisitos de performance por servicio
- Tecnologías heterogéneas (Node.js para I/O intensivo, Java para lógica compleja)
- Equipos que pueden trabajar en paralelo

**Restricciones:**
- Timeline de 4 meses para MVP
- Equipo de 11 personas en Fase 1
- Budget de $810K para desarrollo
- Experiencia del equipo en arquitecturas distribuidas

## Decisión

Adoptaremos una **arquitectura de microservicios distribuidos** con los siguientes servicios principales:

### Servicios Core:
1. **API Gateway** (Node.js) - Punto de entrada único, routing, rate limiting
2. **User Service** (Node.js) - Autenticación, autorización, perfiles
3. **Auction Service** (Java Spring Boot) - Lógica de subastas, pujas, reglas de negocio
4. **Blockchain Service** (Node.js) - Integración con Polygon/Ethereum, smart contracts
5. **Payment Service** (Java Spring Boot) - Procesamiento de pagos, escrow
6. **Notification Service** (Node.js) - Multi-canal (email, SMS, push)
7. **Search Service** (Node.js) - Búsqueda, filtrado, indexación

### Comunicación entre servicios:
- **REST APIs** para comunicación externa (cliente → API Gateway)
- **gRPC** para comunicación interna de alta performance
- **SQS** para comunicación asíncrona (eventos, mensajes)
- **Event-driven architecture** con patrón CQRS para datos críticos

### Patrón de despliegue:
- Contenedores Docker en Kubernetes (EKS)
- Autoscaling independiente por servicio
- Circuit breakers y retry logic (Resilience4j)
- Service mesh (Istio) en fases posteriores

## Consecuencias

### Positivas

1. **Escalabilidad independiente**
   - Auction Service puede escalar a 10-50 pods durante alta demanda
   - Otros servicios mantienen réplicas mínimas, reduciendo costos

2. **Resiliencia mejorada**
   - Fallo de un servicio no derriba todo el sistema
   - Circuit breakers previenen cascadas de fallos
   - Degradación gradual en lugar de fallo total

3. **Velocidad de desarrollo**
   - Equipos pueden trabajar en paralelo en diferentes servicios
   - Deploy independiente (no necesita coordinar releases)
   - Menos conflictos de código

4. **Flexibilidad tecnológica**
   - Node.js para servicios I/O bound (API Gateway, Notifications, Search)
   - Java para lógica compleja y transaccional (Auction, Payment)
   - Mejor herramienta para cada trabajo

5. **Aislamiento de datos**
   - Cada servicio tiene su propio schema en DynamoDB
   - Reducción de acoplamiento
   - Más fácil cumplir con GDPR (data isolation)

6. **Facilita multi-tenancy y multi-región**
   - Servicios pueden desplegarse en diferentes regiones
   - Latencia reducida para usuarios globales

### Negativas

1. **Complejidad operacional**
   - Necesidad de monitoreo distribuido (X-Ray, CloudWatch)
   - Debugging más complejo (distributed tracing necesario)
   - Más puntos de fallo potenciales

2. **Latencia de red**
   - Llamadas inter-servicio agregan latencia (mitigado con gRPC)
   - Necesidad de caching agresivo (Redis)

3. **Consistencia eventual**
   - Transacciones distribuidas complejas
   - Necesidad de sagas o compensating transactions
   - Sincronización de datos más compleja

4. **Mayor overhead de desarrollo inicial**
   - Setup de infraestructura más complejo
   - Necesidad de API contracts (OpenAPI, Protocol Buffers)
   - Más configuración y boilerplate

5. **Costos de infraestructura**
   - Más pods/contenedores corriendo
   - Load balancers adicionales
   - Networking costs entre servicios

6. **Curva de aprendizaje**
   - Equipo necesita entender sistemas distribuidos
   - Patrones como circuit breakers, sagas, event sourcing

### Neutrales

1. **Necesidad de service mesh (Fase 2+)**
   - Istio para observabilidad, security, traffic management
   - Agrega complejidad pero beneficios a escala

2. **CI/CD más sofisticado**
   - Pipelines por servicio
   - Deployment orchestration
   - Feature flags para releases controlados

3. **Contratos de API estrictos**
   - OpenAPI para REST
   - Protocol Buffers para gRPC
   - Versionado de APIs

## Alternativas Consideradas

### Alternativa 1: Monolito Modular
**Descripción:** Aplicación única con módulos bien definidos internamente

**Ventajas:**
- Más simple de desarrollar inicialmente
- Deployment más sencillo (un solo artefacto)
- Transacciones ACID más fáciles
- Debugging más simple

**Razones para rechazar:**
- No escala independientemente (todo o nada)
- Deployment de riesgo (un bug afecta todo)
- Difícil trabajar en paralelo (más conflictos de código)
- No permite tecnologías heterogéneas
- Migración a microservicios después es costosa

### Alternativa 2: Serverless (AWS Lambda)
**Descripción:** Functions as a Service, sin servidores

**Ventajas:**
- Escala automáticamente
- Paga solo por uso
- No administración de servidores
- Deploy muy rápido

**Razones para rechazar:**
- Cold starts afectan latencia (<2s para pujas es crítico)
- Límites de timeout (15 min Lambda) problemáticos para procesos largos
- Difícil para conexiones persistentes (WebSockets para real-time)
- Vendor lock-in fuerte con AWS
- Debugging y local development más complejo
- Costos impredecibles a gran escala

### Alternativa 3: Microservicios en EC2 (sin Kubernetes)
**Descripción:** Microservicios desplegados directamente en EC2 con Auto Scaling Groups

**Ventajas:**
- Menos complejidad que Kubernetes
- Control total sobre VMs
- Costos potencialmente menores

**Razones para rechazar:**
- Menos portable (vendor lock-in)
- Orquestación manual más compleja
- No se beneficia del ecosistema Kubernetes (Helm, Operators, etc.)
- Scaling menos eficiente que K8s
- Equipo tiene experiencia en Kubernetes

### Alternativa 4: Arquitectura Híbrida (Monolito + Microservicios)
**Descripción:** Core como monolito, servicios especializados separados (ej. solo Blockchain Service separado)

**Ventajas:**
- Balance entre simplicidad y escalabilidad
- Migración gradual posible
- Menos overhead inicial

**Razones para rechazar:**
- Peor de ambos mundos a mediano plazo
- Límita escalabilidad del core
- Decisión de qué separar es subjetiva
- Migración futura inevitable, mejor hacerlo desde inicio
- No aprovecha experiencia del equipo en microservicios

## Referencias

- PRD Sección 5.1: Arquitectura del Sistema
- PRD Sección 5.2: Stack Tecnológico
- Microservices Patterns (Chris Richardson)
- Building Microservices (Sam Newman)
- AWS Well-Architected Framework - Microservices Lens

## Notas

- En Fase 1 (MVP), empezaremos con los 7 servicios core
- Fase 2 puede agregar servicios adicionales (Analytics Service, Recommendation Service)
- Service mesh (Istio) se evaluará en Fase 2 si la complejidad lo justifica
- Consideraremos BFF (Backend for Frontend) pattern si mobile app requiere APIs diferentes

## Historial

- 2025-11-07: Decisión inicial (Aceptado)
