## 🎯 Prompt 1: Análisis Inicial del PRD

```
Analiza el PRD ubicado en [ruta/al/PRD.md] y genera un análisis estructurado que incluya:

1. **Resumen Ejecutivo**: Objetivos principales del producto
2. **Requisitos Funcionales**: Lista priorizada de funcionalidades
3. **Requisitos No Funcionales**: Performance, seguridad, escalabilidad, etc.
4. **Stakeholders y Usuarios**: Perfiles y necesidades
5. **Restricciones y Dependencias**: Limitaciones técnicas y de negocio
6. **Alcance e Hitos**: Fases del proyecto

Guarda este análisis en `/docs/00-prd-analysis.md` para referencia en los siguientes pasos.
```

---

## 🏛️ Prompt 2: Architecture Decision Records (ADRs)

```
Basándote en el análisis del PRD en `/docs/00-prd-analysis.md`, genera Architecture Decision Records (ADRs) siguiendo el formato estándar:

Para cada decisión arquitectónica importante, crea un ADR en `/docs/adrs/` con la siguiente estructura:

# ADR-XXX: [Título de la Decisión]

## Estado
[Propuesto | Aceptado | Rechazado | Deprecado]

## Contexto
Describe el problema o necesidad que origina esta decisión, incluyendo:
- Requisitos del PRD relacionados
- Factores técnicos y de negocio
- Restricciones existentes

## Decisión
Explica claramente qué se decidió y por qué

## Consecuencias
### Positivas
- Beneficios de esta decisión

### Negativas
- Trade-offs y desventajas

### Neutrales
- Cambios necesarios o impactos generales

## Alternativas Consideradas
Lista otras opciones evaluadas y por qué fueron descartadas

---

Genera ADRs para al menos:
1. Arquitectura general del sistema (monolito vs microservicios vs híbrido)
2. Stack tecnológico principal (lenguajes, frameworks)
3. Estrategia de base de datos (SQL vs NoSQL, única vs múltiple)
4. Infraestructura y deployment (cloud provider, containerización)
5. Autenticación y autorización
6. Estrategia de API (REST, GraphQL, gRPC)
7. Estrategia de testing
8. Manejo de estado (si aplica para frontend)

Crea un índice en `/docs/adrs/README.md` listando todos los ADRs.
```

---

## 📐 Prompt 3: C4 Model - Diagrama de Contexto

```
Genera el diagrama de Contexto (nivel 1) del C4 Model basado en el PRD.

Crea el archivo `/docs/architecture/c4-01-context.md` con:

1. **Descripción del Sistema**: Propósito y alcance general
2. **Diagrama de Contexto** en formato Mermaid:
   - El sistema como caja central
   - Usuarios/actores externos
   - Sistemas externos con los que interactúa
   - Flujos de comunicación principales

3. **Descripción de Elementos**:
   - Para cada actor: rol y objetivos
   - Para cada sistema externo: propósito e integración

Ejemplo de formato:

\`\`\`mermaid
C4Context
    title Diagrama de Contexto del Sistema

    Person(user, "Usuario", "Descripción del usuario")
    Person(admin, "Administrador", "Descripción del admin")
    
    System(system, "Sistema Principal", "Descripción del sistema")
    
    System_Ext(external1, "Sistema Externo 1", "Descripción")
    System_Ext(external2, "Sistema Externo 2", "Descripción")
    
    Rel(user, system, "Usa", "HTTPS")
    Rel(system, external1, "Consulta datos", "API REST")
\`\`\`
```

---

## 📦 Prompt 4: C4 Model - Diagrama de Contenedores

```
Genera el diagrama de Contenedores (nivel 2) del C4 Model.

Crea el archivo `/docs/architecture/c4-02-container.md` con:

1. **Descripción de la Arquitectura de Contenedores**
2. **Diagrama de Contenedores** en formato Mermaid mostrando:
   - Aplicaciones web/móviles
   - APIs y servicios backend
   - Bases de datos
   - Sistemas de caché
   - Message queues (si aplica)
   - Servicios de almacenamiento

3. **Descripción Detallada** de cada contenedor:
   - Tecnología utilizada
   - Responsabilidades
   - Patrones de comunicación

\`\`\`mermaid
C4Container
    title Diagrama de Contenedores
    
    Person(user, "Usuario")
    
    Container_Boundary(system, "Sistema Principal") {
        Container(web, "Web Application", "React", "SPA que provee la interfaz")
        Container(api, "API Gateway", "Node.js/Express", "Punto de entrada de APIs")
        Container(service1, "Servicio de Negocio", "Python/FastAPI", "Lógica de negocio")
        ContainerDb(db, "Base de Datos", "PostgreSQL", "Almacena datos")
        ContainerDb(cache, "Cache", "Redis", "Cache de sesiones")
    }
    
    Rel(user, web, "Usa", "HTTPS")
    Rel(web, api, "Llama", "JSON/HTTPS")
    Rel(api, service1, "Enruta a", "JSON/HTTP")
\`\`\`
```

---

## 🔧 Prompt 5: C4 Model - Diagramas de Componentes

```
Genera diagramas de Componentes (nivel 3) del C4 Model para los contenedores más importantes.

Para cada contenedor principal (típicamente 2-3), crea un archivo `/docs/architecture/c4-03-component-[nombre].md` con:

1. **Descripción del Contenedor**
2. **Diagrama de Componentes** en Mermaid mostrando:
   - Controladores/Handlers
   - Servicios de negocio
   - Repositorios/DAOs
   - Componentes de seguridad
   - Utilidades y helpers

3. **Responsabilidades** de cada componente
4. **Patrones implementados** (Repository, Service Layer, etc.)

Ejemplo:

\`\`\`mermaid
C4Component
    title Componentes - API Backend
    
    Container_Boundary(api, "API Backend") {
        Component(authController, "Auth Controller", "Express Router", "Maneja autenticación")
        Component(userController, "User Controller", "Express Router", "Maneja usuarios")
        
        Component(authService, "Auth Service", "Service", "Lógica de autenticación")
        Component(userService, "User Service", "Service", "Lógica de usuarios")
        
        Component(userRepo, "User Repository", "Repository", "Acceso a datos de usuarios")
        
        ComponentDb(db, "Database", "PostgreSQL", "Almacenamiento")
    }
    
    Rel(authController, authService, "Usa")
    Rel(authService, userRepo, "Lee/Escribe usuarios")
\`\`\`
```

---

## 💻 Prompt 6: C4 Model - Diagramas de Código (Opcional)

```
Para los componentes más críticos o complejos, genera diagramas de Código (nivel 4) del C4 Model.

Crea archivos `/docs/architecture/c4-04-code-[componente].md` con:

1. Diagramas de clases UML en Mermaid
2. Interfaces y contratos principales
3. Flujo de datos dentro del componente

\`\`\`mermaid
classDiagram
    class UserController {
        -userService: UserService
        +getUser(id: string)
        +createUser(data: UserDto)
        +updateUser(id: string, data: UserDto)
    }
    
    class UserService {
        -userRepository: UserRepository
        -validator: Validator
        +findById(id: string): User
        +create(data: UserDto): User
        +update(id: string, data: UserDto): User
    }
    
    class UserRepository {
        -db: Database
        +findById(id: string): User
        +save(user: User): User
    }
    
    UserController --> UserService
    UserService --> UserRepository
\`\`\`
```

---

## 📘 Prompt 7: System Design Document

```
Genera un System Design Document completo en `/docs/system-design.md` que incluya:

## 1. Visión General
- Propósito del sistema
- Objetivos de alto nivel
- Stakeholders

## 2. Arquitectura del Sistema
- Resumen de la arquitectura elegida
- Referencias a los ADRs relevantes
- Diagrama de arquitectura de alto nivel

## 3. Componentes Principales
Para cada componente:
- Descripción y responsabilidades
- Tecnologías utilizadas
- Interfaces y contratos
- Dependencias

## 4. Flujos de Datos
- Diagramas de flujo para procesos principales
- Formato de datos en cada etapa
- Transformaciones aplicadas

## 5. Modelo de Datos
- Esquema de base de datos
- Entidades principales y relaciones
- Estrategias de indexación

## 6. Seguridad
- Autenticación y autorización
- Encriptación (en tránsito y reposo)
- Manejo de secretos
- Cumplimiento normativo

## 7. Escalabilidad y Performance
- Estrategias de escalado (horizontal/vertical)
- Caching
- Load balancing
- Optimizaciones previstas

## 8. Resiliencia y Alta Disponibilidad
- Manejo de fallos
- Estrategias de backup
- Disaster recovery
- Monitoreo y alertas

## 9. Despliegue e Infraestructura
- Arquitectura de infraestructura
- CI/CD pipeline
- Ambientes (dev, staging, prod)
- Estrategia de releases

## 10. Observabilidad
- Logging
- Métricas
- Tracing distribuido
- Dashboards

Incluye diagramas en Mermaid donde sea relevante.
```

---

## 📡 Prompt 8: Documentación de API

```
Genera documentación completa de API en formato OpenAPI 3.0 en `/docs/api-specification.yaml` y su versión legible en `/docs/api-documentation.md`.

La documentación debe incluir:

## 1. Información General
- Título y versión de la API
- Descripción
- Servidor base URL
- Esquemas de autenticación

## 2. Para cada Endpoint:
- Ruta y método HTTP
- Descripción y propósito
- Parámetros (path, query, header)
- Request body (con ejemplos)
- Respuestas posibles (con códigos HTTP y ejemplos)
- Códigos de error comunes

## 3. Modelos de Datos
- Esquemas de todos los DTOs
- Tipos de datos y validaciones
- Campos requeridos vs opcionales

## 4. Autenticación
- Flujos de autenticación
- Formato de tokens
- Renovación de tokens

## 5. Rate Limiting
- Límites por endpoint
- Headers de rate limit

## 6. Ejemplos de Uso
- Curl examples
- SDK examples (si aplica)

Genera también un archivo Postman/Insomnia collection en `/docs/api-collection.json` para testing.
```

---

## 📊 Prompt 9: Diagramas de Secuencia

```
Genera diagramas de secuencia en Mermaid para los flujos principales del sistema.

Crea archivos individuales en `/docs/diagrams/sequences/` para:

1. **Autenticación y Autorización**
2. **Flujo principal de usuario** (happy path)
3. **Procesos de negocio críticos** (mínimo 3-5 según el PRD)
4. **Manejo de errores** en escenarios importantes
5. **Integraciones con sistemas externos**

Formato para cada diagrama:

\`\`\`mermaid
sequenceDiagram
    actor Usuario
    participant WebApp
    participant API
    participant AuthService
    participant Database
    
    Usuario->>WebApp: Ingresa credenciales
    WebApp->>API: POST /auth/login
    API->>AuthService: Valida credenciales
    AuthService->>Database: Busca usuario
    Database-->>AuthService: Retorna datos usuario
    AuthService->>AuthService: Verifica password
    AuthService-->>API: Token JWT
    API-->>WebApp: {token, user}
    WebApp->>WebApp: Guarda token
    WebApp-->>Usuario: Redirige a dashboard
\`\`\`

Incluye notas explicativas para pasos complejos.
```

---

## 🔄 Prompt 10: Diagramas de Flujo de Datos

```
Genera diagramas de flujo de datos (DFD) para el sistema en `/docs/diagrams/data-flows/`.

Crea diagramas para:

1. **DFD Nivel 0**: Visión general del sistema
2. **DFD Nivel 1**: Procesos principales descompuestos
3. **DFD Nivel 2**: Detalles de procesos críticos

Usa Mermaid flowchart:

\`\`\`mermaid
flowchart TD
    A[Usuario ingresa datos] --> B{Validación}
    B -->|Válido| C[Procesamiento]
    B -->|Inválido| D[Mensaje de error]
    C --> E[Guardar en BD]
    E --> F[Generar respuesta]
    F --> G[Enviar notificación]
    F --> H[Retornar al usuario]
    
    style B fill:#f9f,stroke:#333,stroke-width:4px
    style E fill:#bbf,stroke:#333,stroke-width:2px
\`\`\`

Para cada diagrama incluye:
- Descripción del flujo
- Entradas y salidas
- Transformaciones aplicadas
- Puntos de decisión
- Manejo de errores
```

---

## 🏗️ Prompt 11: Diagramas de Infraestructura

```
Genera diagramas de infraestructura en `/docs/diagrams/infrastructure/` para:

1. **Arquitectura de Producción**
2. **Arquitectura de Desarrollo/Staging**
3. **Red y Seguridad**
4. **CI/CD Pipeline**

Usa Mermaid graph o flowchart:

\`\`\`mermaid
graph TB
    subgraph "AWS Cloud"
        subgraph "VPC"
            subgraph "Public Subnet"
                ALB[Application Load Balancer]
                NAT[NAT Gateway]
            end
            
            subgraph "Private Subnet"
                API1[API Server 1]
                API2[API Server 2]
                API3[API Server 3]
            end
            
            subgraph "Data Subnet"
                RDS[(RDS PostgreSQL<br/>Primary)]
                RDS_R[(RDS Replica)]
                Redis[(Redis Cluster)]
            end
        end
        
        S3[S3 Bucket]
        CF[CloudFront CDN]
    end
    
    Internet[Internet] --> CF
    CF --> ALB
    ALB --> API1
    ALB --> API2
    ALB --> API3
    
    API1 --> RDS
    API2 --> RDS
    API3 --> RDS
    
    RDS --> RDS_R
    
    API1 --> Redis
    API2 --> Redis
    API3 --> Redis
\`\`\`

Incluye:
- Componentes de infraestructura
- Zonas de disponibilidad
- Grupos de seguridad y firewalls
- Estrategias de backup
- Monitoreo y logging
```

---

## 🛠️ Prompt 12: Tech Stack Overview

```
Genera un documento completo del Tech Stack en `/docs/tech-stack.md` que incluya:

## 1. Stack General
Tabla resumen:

| Categoría | Tecnología | Versión | Justificación |
|-----------|-----------|---------|---------------|
| Frontend Framework | React | 18.x | [Razón de la elección] |
| Backend Framework | Node.js + Express | 20.x LTS | [Razón] |
| Base de Datos | PostgreSQL | 15.x | [Razón] |
| Cache | Redis | 7.x | [Razón] |
| ... | ... | ... | ... |

## 2. Frontend
- Framework principal y librerías
- State management
- Styling (CSS framework, CSS-in-JS)
- Build tools y bundlers
- Testing frameworks
- Justificación para cada elección

## 3. Backend
- Lenguaje y runtime
- Framework web
- ORM/Query builder
- Librerías principales
- Testing frameworks
- Justificación para cada elección

## 4. Base de Datos y Almacenamiento
- Base de datos principal
- Bases de datos secundarias (si aplica)
- Estrategia de cache
- Object storage
- Justificación y trade-offs

## 5. Infraestructura y DevOps
- Cloud provider
- Containerización (Docker, K8s)
- CI/CD tools
- Infrastructure as Code (Terraform, etc.)
- Monitoreo y observabilidad
- Justificación de la arquitectura elegida

## 6. Seguridad
- Autenticación (JWT, OAuth, etc.)
- Encriptación
- Secrets management
- Herramientas de seguridad

## 7. Dependencias Críticas
Lista de librerías y servicios externos críticos con:
- Propósito
- Alternativas evaluadas
- Riesgos y mitigación

## 8. Análisis Comparativo
Para cada decisión importante, incluye tabla comparativa:

| Criterio | Opción A | Opción B | Opción C | Ganador |
|----------|----------|----------|----------|---------|
| Performance | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | B |
| Developer Experience | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | A |
| Comunidad | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | A |
| Costo | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | B |

## 9. Roadmap Tecnológico
- Tecnologías a adoptar en el futuro
- Deprecaciones planeadas
- Actualizaciones mayores previstas

## 10. Referencias
- Links a documentación oficial
- ADRs relacionados
- Recursos de aprendizaje
```

---

## 🔍 Prompt 13: Generación del Índice Master

```
Genera un índice master en `/docs/README.md` que organice toda la documentación generada:

# Documentación Técnica del Sistema

## 📋 Documentos Fundamentales
- [Análisis del PRD](./00-prd-analysis.md)
- [System Design Document](./system-design.md)
- [Tech Stack Overview](./tech-stack.md)

## 🏛️ Architecture Decision Records (ADRs)
- [Índice de ADRs](./adrs/README.md)
- Lista de ADRs individuales con links

## 📐 Arquitectura (C4 Model)
- [C4-01: Diagrama de Contexto](./architecture/c4-01-context.md)
- [C4-02: Diagrama de Contenedores](./architecture/c4-02-container.md)
- [C4-03: Diagramas de Componentes](./architecture/)
- [C4-04: Diagramas de Código](./architecture/)

## 📡 API
- [Especificación OpenAPI](./api-specification.yaml)
- [Documentación de API](./api-documentation.md)
- [Colección Postman](./api-collection.json)

## 📊 Diagramas
### Secuencia
- Lista de diagramas de secuencia

### Flujo de Datos
- Lista de DFDs

### Infraestructura
- Lista de diagramas de infraestructura

## 🔗 Referencias Cruzadas
- Mapa de cómo los documentos se relacionan entre sí
- Guía de navegación por caso de uso

## 📝 Convenciones y Estándares
- Guías de estilo de código
- Convenciones de nomenclatura
- Estándares de documentación

Incluye un índice visual/gráfico en Mermaid mostrando la estructura de la documentación.
```

---

## 🚀 Prompt Bonus: Validación y Revisión

```
Realiza una revisión completa de toda la documentación generada:

1. **Consistencia**: Verifica que todos los documentos estén alineados entre sí
2. **Completitud**: Confirma que todos los requisitos del PRD están cubiertos
3. **Calidad**: Revisa diagramas, formatos y claridad
4. **Enlaces**: Verifica que todos los links internos funcionen
5. **Actualización**: Genera un documento de gaps y mejoras pendientes

Crea un reporte en `/docs/review-report.md` con:
- ✅ Elementos completos y correctos
- ⚠️ Elementos que necesitan revisión
- ❌ Elementos faltantes o incorrectos
- 💡 Sugerencias de mejora

También genera un checklist de calidad en `/docs/quality-checklist.md`.
```

---

## 📖 Guía de Uso

### Orden Recomendado de Ejecución

1. **Prompt 1**: Análisis del PRD (base para todo)
2. **Prompt 2**: ADRs (decisiones arquitectónicas)
3. **Prompts 3-6**: C4 Model (arquitectura visual)
4. **Prompt 7**: System Design Document (consolidación)
5. **Prompt 8**: API Documentation
6. **Prompts 9-11**: Diagramas específicos
7. **Prompt 12**: Tech Stack Overview
8. **Prompt 13**: Índice Master
9. **Prompt Bonus**: Validación

