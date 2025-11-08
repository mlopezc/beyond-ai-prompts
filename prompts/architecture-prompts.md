# Prompts para Claude Code - Análisis Expense Tracker

Este documento contiene una serie de prompts optimizados para usar con Claude Code y analizar el proyecto Expense Tracker ubicado en `/Users/mario/Documents/code-projects/Expense-Tracker`.

---

## 📋 Prompt 1: Análisis Inicial y Tech Stack Overview

```
Analiza en este proyecto y genera un Tech Stack Overview completo.

Por favor:
1. Explora todos los archivos de configuración (package.json, requirements.txt, pom.xml, etc.)
2. Identifica el stack tecnológico completo (frontend, backend, base de datos, infraestructura)
3. Documenta las versiones de las tecnologías utilizadas
4. Explica la justificación de cada tecnología elegida basándote en el código
5. Identifica patrones de diseño y arquitecturas utilizadas
6. Lista las dependencias principales y su propósito

Genera un documento markdown con:
- Resumen ejecutivo del stack
- Tabla de tecnologías por categoría (Frontend, Backend, Database, DevOps, Testing, etc.)
- Justificación técnica de cada elección
- Diagrama de dependencias principales
- Consideraciones de compatibilidad y versiones

Guarda el resultado en: /docs/tech-stack-overview.md
```

---

## 🏗️ Prompt 2: Architecture Decision Records (ADRs)

```
Analiza el código del proyecto Expense Tracker y genera Architecture Decision Records (ADRs) para las decisiones arquitectónicas clave.

Examina:
1. Estructura de carpetas y organización del código
2. Patrones de diseño implementados
3. Elección de frameworks y librerías principales
4. Estrategias de autenticación y autorización
5. Manejo de estado (si aplica)
6. Estrategias de persistencia de datos
7. APIs y protocolos de comunicación
8. Estrategias de testing
9. Configuración de deployment

Para cada decisión identificada, crea un ADR con el formato:
- **Título**: ADR-XXX: [Título descriptivo]
- **Estado**: Aceptado/Propuesto/Deprecado
- **Contexto**: ¿Qué problema estamos resolviendo?
- **Decisión**: ¿Qué decidimos hacer?
- **Consecuencias**: ¿Qué implicaciones tiene esta decisión?
- **Alternativas consideradas**: ¿Qué otras opciones evaluamos?

Crea ADRs para al menos 8-10 decisiones arquitectónicas importantes.

Guarda cada ADR en: /docs/adr/
Crea también un índice: /docs/adr/README.md
```

---

## 📐 Prompt 3: C4 Model - Context Diagram

```
Genera el diagrama de Contexto (C4 Level 1) para el sistema Expense Tracker.

Analiza el proyecto para identificar:
1. El sistema principal (Expense Tracker)
2. Todos los usuarios/actores externos (personas, roles)
3. Sistemas externos con los que interactúa (servicios de terceros, APIs, etc.)
4. Flujos de datos principales entre el sistema y entidades externas

Genera:
1. Un diagrama en formato plant uml del Context Diagram
2. Una descripción textual del sistema y sus límites
3. Tabla de actores externos con sus responsabilidades
4. Tabla de sistemas externos con propósito de integración

Formato de salida:
- Descripción en markdown
- Diagrama plant uml C4Context
- Explicación de cada relación

Guarda en: /docs/c4-model/01-context.md
```

---

## 📦 Prompt 4: C4 Model - Container Diagram

```
Genera el diagrama de Contenedores (C4 Level 2) para Expense Tracker.

Analiza la arquitectura del proyecto para identificar:
1. Aplicaciones (frontend, backend, móvil, etc.)
2. Bases de datos y sistemas de almacenamiento
3. Servicios y microservicios
4. Sistemas de mensajería o colas
5. APIs y sus tecnologías
6. Protocolos de comunicación entre contenedores

Para cada contenedor documenta:
- Nombre y propósito
- Tecnología utilizada
- Responsabilidades principales
- Interfaces expuestas

Genera:
1. Diagrama plant uml C4Container
2. Descripción detallada de cada contenedor
3. Tabla de tecnologías por contenedor
4. Protocolos de comunicación
5. Consideraciones de deployment

Guarda en: /docs/c4-model/02-container.md
```

---

## 🔧 Prompt 5: C4 Model - Component Diagram

```
Genera diagramas de Componentes (C4 Level 3) para los contenedores principales del sistema Expense Tracker.

Para cada contenedor principal (ej: Backend API, Frontend App):
1. Identifica los componentes principales (controladores, servicios, repositorios, etc.)
2. Mapea las responsabilidades de cada componente
3. Documenta las dependencias entre componentes
4. Identifica patrones arquitectónicos (MVC, Clean Architecture, etc.)

Genera para cada contenedor:
1. Diagrama plant uml C4Component
2. Descripción de cada componente
3. Tabla de responsabilidades
4. Flujos de datos internos
5. Patrones de diseño aplicados

Crea un archivo por cada contenedor principal en:
/docs/c4-model/03-components/
```

---

## 💻 Prompt 6: C4 Model - Code Diagram (Clases Principales)

```
Genera diagramas de Código (C4 Level 4) para los componentes más críticos del sistema.

Selecciona 3-5 componentes críticos y para cada uno:
1. Genera diagrama de clases UML
2. Documenta las principales clases y sus relaciones
3. Incluye métodos y propiedades relevantes
4. Muestra patrones de diseño implementados (Factory, Repository, Strategy, etc.)

Usa formato plant uml classDiagram para cada uno.

Aspectos a documentar:
- Herencia y composición
- Interfaces implementadas
- Dependencias principales
- Responsabilidad de cada clase (principio SRP)

Guarda en: /docs/c4-model/04-code/
```

---

## 📘 Prompt 7: System Design Document

```
Genera un System Design Document completo para Expense Tracker.

El documento debe incluir:

1. **Visión General del Sistema**
   - Propósito y objetivos
   - Usuarios objetivo
   - Casos de uso principales

2. **Arquitectura del Sistema**
   - Estilo arquitectónico (monolítico, microservicios, etc.)
   - Componentes principales
   - Diagrama de alto nivel

3. **Diseño de Datos**
   - Modelo de datos (ERD)
   - Esquema de base de datos
   - Estrategias de almacenamiento

4. **Diseño de APIs**
   - Endpoints principales
   - Autenticación y autorización
   - Rate limiting y seguridad

5. **Diseño de Frontend**
   - Arquitectura de componentes
   - Gestión de estado
   - Routing y navegación

6. **Consideraciones No Funcionales**
   - Performance y escalabilidad
   - Seguridad
   - Monitoreo y logging
   - Backup y recuperación

7. **Deployment y DevOps**
   - Estrategia de deployment
   - CI/CD pipeline
   - Ambientes (dev, staging, prod)

Guarda en: /docs/system-design-document.md
```

---

## 🔌 Prompt 8: API Documentation

```
Genera documentación completa de la API del sistema Expense Tracker.

Analiza el código backend para:
1. Identificar todos los endpoints de la API
2. Métodos HTTP utilizados
3. Parámetros de entrada (path, query, body)
4. Responses y códigos de estado
5. Schemas de datos (request/response)
6. Autenticación requerida
7. Ejemplos de uso

Para cada endpoint documenta:
- Ruta completa
- Método HTTP
- Descripción y propósito
- Parámetros (con tipos y validaciones)
- Request body (schema JSON)
- Response body (schema JSON)
- Códigos de estado HTTP posibles
- Ejemplo de llamada (curl/fetch)
- Consideraciones de seguridad

Genera la documentación en formato OpenAPI 3.0 (Swagger) Y también en Markdown.

Guarda en:
- OpenAPI: /docs/api-spec.yaml
- Markdown: /docs/api-documentation.md
```

---

## 📊 Prompt 9: Diagramas de Secuencia

```
Genera diagramas de secuencia para los flujos principales del sistema Expense Tracker.

Identifica y documenta los siguientes flujos (o los más relevantes que encuentres):
1. Registro de usuario
2. Login/Autenticación
3. Creación de un gasto
4. Consulta de gastos con filtros
5. Actualización de un gasto
6. Generación de reportes
7. [Otros flujos críticos identificados]

Para cada flujo crea:
1. Diagrama de secuencia en plant uml
2. Descripción paso a paso
3. Actores involucrados
4. Validaciones y manejo de errores
5. Llamadas a APIs o servicios externos

Guarda en: /docs/diagrams/sequence/
```

---

## 🔄 Prompt 10: Diagramas de Flujo de Datos

```
Genera diagramas de flujo de datos (DFD) para el sistema Expense Tracker.

Crea diagramas para:
1. **DFD Nivel 0 (Contexto)**: Sistema completo como caja negra
2. **DFD Nivel 1**: Procesos principales del sistema
3. **DFD Nivel 2**: Descomposición de procesos complejos

Identifica y documenta:
- Procesos (transformaciones de datos)
- Almacenes de datos (databases, caches, etc.)
- Flujos de datos entre procesos
- Entidades externas
- Validaciones y transformaciones

Usa plant uml flowchart para representar los DFDs.

Incluye para cada nivel:
- Diagrama visual
- Descripción de cada proceso
- Tabla de flujos de datos
- Transformaciones aplicadas

Guarda en: /docs/diagrams/data-flow/
```

---

## 🏗️ Prompt 11: Diagrama de Infraestructura

```
Genera un diagrama de infraestructura y deployment para Expense Tracker.

Analiza la configuración del proyecto (Docker, Kubernetes, cloud configs) para identificar:
1. Componentes de infraestructura (servidores, contenedores, bases de datos)
2. Networking (load balancers, DNS, CDN)
3. Servicios de cloud utilizados
4. Estrategia de deployment
5. Ambientes (desarrollo, staging, producción)
6. Servicios de monitoreo y logging

Genera:
1. Diagrama de arquitectura de infraestructura (plant uml)
2. Diagrama de red y comunicaciones
3. Descripción de cada componente
4. Tabla de servicios y sus configuraciones
5. Estrategia de alta disponibilidad y escalamiento
6. Consideraciones de seguridad (firewalls, VPC, etc.)

Guarda en: /docs/diagrams/infrastructure.md
```

---

## 📋 Prompt 12: Consolidación y Índice General

```
Crea un documento índice maestro que consolide toda la documentación generada.

Genera un README.md principal en /docs/ que incluya:

1. **Introducción al Proyecto**
   - Descripción breve
   - Propósito y objetivos
   - Audiencia de la documentación

2. **Índice de Documentación**
   - Tech Stack Overview (con link)
   - Architecture Decision Records (con links)
   - C4 Model (con links a todos los niveles)
   - System Design Document (con link)
   - API Documentation (con link)
   - Diagramas (con links a todos)

3. **Guías de Navegación**
   - Para desarrolladores nuevos: ¿qué leer primero?
   - Para arquitectos: documentos de arquitectura
   - Para API consumers: documentación de API
   - Para DevOps: infraestructura y deployment

4. **Convenciones**
   - Formato de documentación
   - Cómo actualizar los docs
   - Proceso de revisión

5. **Resumen Visual**
   - Diagrama de contexto principal
   - Stack diagram

Asegúrate de que todos los links funcionen correctamente.

Guarda en: /docs/README.md
```

