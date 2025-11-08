# ADR-002: Stack Tecnológico Multi-Lenguaje (Node.js + Java + React)

## Estado
Aceptado

## Contexto

La plataforma ArtChain tiene diferentes tipos de servicios con requisitos distintos:
- **I/O-intensive:** API Gateway, notificaciones, búsqueda, blockchain integration
- **Compute-intensive:** Lógica de subastas, cálculos de pujas, procesamiento de pagos
- **Real-time:** Actualizaciones de pujas en tiempo real (WebSockets)
- **Frontend:** Aplicación web responsive, mobile-ready

**Requisitos del PRD relacionados:**
- Latencia API <200ms (p95), <500ms (p99)
- Confirmación de puja <2 segundos
- Notificaciones en tiempo real <1 segundo
- 10,000 usuarios concurrentes
- Frontend: Time to Interactive <5 segundos, Lighthouse score >90

**Factores técnicos:**
- Equipo tiene experiencia en Node.js, Java, y React
- Necesidad de performance para sistema de pujas
- Integración con blockchain (Web3.js disponible en Node.js)
- Ecosistema maduro y libraries disponibles

**Restricciones:**
- Timeline de 4 meses para MVP
- Equipo de 11 personas (2 FE, 3 BE, 1 Blockchain)
- Preferencia por tecnologías maduras y probadas en producción

## Decisión

Adoptaremos un **stack tecnológico multi-lenguaje** optimizado por tipo de servicio:

### Backend - Microservicios

#### Node.js 20 LTS con TypeScript
**Servicios:** API Gateway, User Service, Notification Service, Blockchain Service, Search Service

**Framework y Librerías:**
- **NestJS** como framework principal (estructurado, enterprise-ready)
- **Express.js** para servicios simples
- **TypeScript** obligatorio para type safety
- **Socket.io** para WebSockets (real-time updates)
- **Web3.js / Ethers.js** para integración blockchain
- **Bull** para job queues
- **Passport.js** para estrategias de autenticación
- **Winston** para logging estructurado
- **Jest** para testing

**Razones:**
- Event-driven, non-blocking I/O (perfecto para APIs, WebSockets, notificaciones)
- Ecosistema maduro de blockchain (Web3.js, Ethers.js)
- Excelente para I/O bound operations
- TypeScript mejora mantenibilidad y reduce bugs

#### Java 21 con Spring Boot 3.x
**Servicios:** Auction Service, Payment Service

**Framework y Librerías:**
- **Spring Boot 3.x** (framework enterprise-proven)
- **Spring Data** para acceso a datos
- **Spring Security** para autenticación/autorización
- **Resilience4j** para circuit breakers, retry
- **gRPC Spring Boot Starter** para comunicación interna
- **JUnit 5 + Mockito** para testing

**Razones:**
- Excelente para lógica de negocio compleja y transaccional
- Performance superior para cálculos intensivos
- Tipado fuerte y thread safety
- Ecosistema maduro para pagos (Stripe SDK, etc.)
- Spring Boot facilita configuración y deployment

### Frontend

#### React.js 18+ con TypeScript
**Framework y Librerías:**
- **React 18+** con Concurrent Features
- **TypeScript** para type safety
- **Redux Toolkit** para state management (ver ADR-008)
- **React Query (TanStack Query)** para data fetching y caching
- **Socket.io-client** para WebSockets
- **Tailwind CSS** para styling
- **React Router v6** para navegación
- **Formik + Yup** para forms y validación
- **Chart.js** para visualizaciones
- **Vite** como bundler (rápido, moderno)
- **Jest + React Testing Library** para testing

**Razones:**
- Ecosistema más grande y maduro
- Performance excelente con Virtual DOM y Concurrent Mode
- Comunidad activa y librerías de calidad
- Equipo tiene experiencia
- Excelente para aplicaciones complejas con mucho estado

### Build & DevOps

- **Docker** para containerización
- **Kubernetes (EKS)** para orquestación
- **GitHub Actions** para CI/CD
- **Terraform** para Infrastructure as Code
- **ESLint + Prettier** para linting
- **Husky** para pre-commit hooks

## Consecuencias

### Positivas

1. **Performance optimizado por caso de uso**
   - Node.js para I/O bound (API Gateway, WebSockets) → baja latencia
   - Java para CPU bound (Auction logic, Payments) → throughput alto

2. **Ecosistema maduro**
   - Todas las tecnologías tienen librerías de calidad
   - Fácil encontrar soluciones a problemas comunes
   - Documentación extensa

3. **Type Safety (TypeScript en Node + Java)**
   - Menos bugs en producción
   - Mejor IntelliSense y developer experience
   - Refactoring más seguro

4. **Blockchain Integration Natural**
   - Web3.js y Ethers.js son las libraries estándar
   - Mejor en Node.js que en otros lenguajes
   - Comunidad activa de developers blockchain en JS

5. **Hiring más fácil**
   - JavaScript/TypeScript y Java son los lenguajes más populares
   - React es el framework frontend más usado
   - Pool de talento grande

6. **Comunidad y soporte**
   - Stack Overflow, GitHub Issues, Discord communities
   - Actualizaciones regulares y seguridad
   - LTS versions disponibles

7. **Frontend Moderno**
   - React 18 Concurrent Features para mejor UX
   - Vite para build ultra-rápido (vs Webpack)
   - Tailwind CSS para desarrollo rápido

### Negativas

1. **Complejidad de multi-lenguaje**
   - Developers necesitan conocer al menos 2 lenguajes
   - Más configuraciones, linters, formatters
   - Contexto switching entre proyectos

2. **Más overhead de CI/CD**
   - Pipelines diferentes para Node.js vs Java
   - Docker images de diferentes tamaños
   - Testing frameworks diferentes

3. **Consistencia de código**
   - Necesidad de style guides por lenguaje
   - Code reviews requieren expertise específico
   - Patrones pueden divergir entre servicios

4. **Dependencies management**
   - npm/yarn para Node.js
   - Maven/Gradle para Java
   - Diferentes formas de manejar vulnerabilidades

5. **Learning curve**
   - Developers de Node necesitan aprender Java y viceversa
   - Onboarding de nuevos miembros más complejo

6. **Potencial duplicación**
   - Utilities pueden duplicarse entre lenguajes
   - Validations, error handling diferentes

### Neutrales

1. **Monorepo vs Polyrepo**
   - Consideraremos monorepo (Nx, Turborepo) para mejor DX
   - O polyrepo con templates bien definidos

2. **Shared code**
   - Protocol Buffers para gRPC compartido
   - OpenAPI specs compartidas

3. **Versioning**
   - Semantic versioning para todos los servicios
   - API versioning consistente

## Alternativas Consideradas

### Alternativa 1: Node.js para Todo
**Descripción:** Usar solo Node.js/TypeScript para todos los servicios backend

**Ventajas:**
- Stack único, más simple
- Un solo pipeline de CI/CD
- Developers full-stack (todos conocen todo)
- Código compartido más fácil

**Razones para rechazar:**
- Performance inferior para cálculos complejos (Auction logic)
- Node.js single-threaded no ideal para CPU-intensive tasks
- Menos robusto para transacciones complejas de pagos
- Tipado dinámico (incluso con TS) menos seguro que Java para lógica financiera

### Alternativa 2: Java/Kotlin para Todo
**Descripción:** Usar solo JVM (Java o Kotlin) para todos los servicios

**Ventajas:**
- Stack único
- Performance excelente
- Tipado fuerte en todo el backend
- Spring ecosystem muy completo

**Razones para rechazar:**
- Menos eficiente para I/O bound operations
- Blockchain integration más compleja (Web3j es menos maduro que Web3.js)
- WebSockets más verboso que en Node.js
- Cold start más lento
- Docker images más grandes

### Alternativa 3: Go para Todo
**Descripción:** Usar Go para todos los servicios backend

**Ventajas:**
- Performance excelente
- Concurrency nativa (goroutines)
- Binarios pequeños y rápidos
- Bajo consumo de memoria

**Razones para rechazar:**
- Equipo NO tiene experiencia en Go (learning curve de 4 meses)
- Ecosistema blockchain menos maduro
- Menos librerías third-party que Node/Java
- Verbosidad y falta de generics (hasta Go 1.18)
- Timeline agresivo (4 meses) no permite aprendizaje

### Alternativa 4: Python para Backend
**Descripción:** Usar Python (Django/FastAPI)

**Ventajas:**
- Excelente para data science y ML (recomendaciones Fase 3)
- Sintaxis limpia
- Buen soporte blockchain (Web3.py)

**Razones para rechazar:**
- Performance inferior a Node.js y Java
- GIL limita concurrency
- Tipado dinámico (incluso con type hints)
- Menos adecuado para sistemas de alta performance

### Alternativa 5: Vue.js o Svelte para Frontend
**Vue.js:**
- Más simple que React
- Excelente documentación
- **Rechazado:** Equipo tiene más experiencia en React, ecosistema menor

**Svelte:**
- Performance superior (compila a vanilla JS)
- Menos boilerplate
- **Rechazado:** Ecosistema inmaduro, menos librerías, riesgo para MVP

### Alternativa 6: Next.js (React framework)
**Descripción:** Usar Next.js en lugar de React puro

**Ventajas:**
- SSR out of the box
- Routing automático
- Image optimization
- API routes

**Consideración:** Evaluaremos Next.js en Fase 2 si SSR/SEO se vuelve importante. Para MVP, React SPA es suficiente.

## Referencias

- PRD Sección 5.2: Stack Tecnológico
- Node.js Best Practices: https://github.com/goldbergyoni/nodebestpractices
- Spring Boot Documentation: https://spring.io/projects/spring-boot
- React Documentation: https://react.dev

## Notas

- **Code generation:** Consideraremos OpenAPI Generator y gRPC code generation para reducir boilerplate
- **Shared libraries:** Crearemos shared types library para TypeScript (frontend + backend Node)
- **Linting:** ESLint para TS, Checkstyle para Java
- **Formatting:** Prettier para TS/React, Google Java Format para Java

## Historial

- 2025-11-07: Decisión inicial (Aceptado)
