# ADR-003: DynamoDB como Base de Datos Principal con Estrategia Multi-DB

## Estado
Aceptado

## Contexto

La plataforma ArtChain necesita una solución de base de datos que soporte:
- 10,000 usuarios concurrentes
- Throughput de 100,000 requests/min
- Multi-región (US, EU, APAC) con latencia baja
- Disponibilidad 99.9%
- Escalabilidad automática sin downtime
- Diferentes patrones de acceso por servicio

**Requisitos del PRD relacionados:**
- Performance: Latencia API <200ms (p95)
- Escalabilidad: 10K usuarios concurrentes, autoscaling
- Multi-región: Global tables para US-EAST-1, EU-WEST-1, AP-SOUTHEAST-1
- Datos diversos: Usuarios, subastas, pujas, transacciones, notificaciones
- Búsqueda: Full-text search con filtros complejos

**Patrones de acceso identificados:**
1. **User Service:** CRUD usuarios, lookup por email/ID
2. **Auction Service:** Write-heavy (pujas), read-heavy (listados)
3. **Blockchain Service:** Write-once, read-many (certificados)
4. **Payment Service:** Transacciones financieras (ACID crítico)
5. **Search Service:** Full-text search, filtros complejos
6. **Notification Service:** Queue-like, TTL (mensajes expiran)

**Restricciones:**
- AWS como cloud provider (ver ADR-004)
- Budget de $25K/mes para infraestructura (producción)
- Necesidad de backups automáticos y point-in-time recovery
- Compliance GDPR (data isolation, deletion)

## Decisión

Adoptaremos una **estrategia multi-database** con DynamoDB como base de datos principal:

### Base de Datos Principal: DynamoDB

**Para:** User Service, Auction Service, Blockchain Service, Notification Service

**Configuración:**
- **Billing Mode:** On-demand (auto-scaling, predictable costs)
- **Global Tables:** Replicación automática a 3 regiones
- **Point-in-time Recovery:** Activado (PITR)
- **Backups:** Automáticos diarios, retención 35 días
- **Streams:** Activados para event sourcing
- **Encryption:** AWS KMS at rest, TLS in transit

**Tablas principales:**
```
Users (PK: userId, SK: timestamp)
  - GSI: email-index (para login)
  - GSI: role-index (para queries por rol)

Items (PK: itemId, SK: version)
  - GSI: seller-index (PK: sellerId, SK: createdAt)
  - GSI: category-index (PK: category, SK: price)
  - GSI: status-index (PK: status, SK: endDate)

Bids (PK: auctionId, SK: bidTimestamp)
  - Partition key = auctionId para queries eficientes
  - Sort key = timestamp para orden cronológico
  - GSI: bidder-index (PK: userId, SK: bidTimestamp)

Transactions (PK: transactionId, SK: timestamp)
  - GSI: user-index (para historial)
  - GSI: auction-index (para tracking)

Notifications (PK: userId, SK: timestamp)
  - TTL activado (mensajes expiran después de 90 días)
  - GSI: status-index (para pending notifications)
```

**Diseño de claves:**
- Partition Key (PK): ID principal para distribución uniforme
- Sort Key (SK): Timestamp o ID secundario para queries ordenadas
- GSI (Global Secondary Indexes): Para patrones de acceso alternativos
- Evitar hot partitions con composite keys donde necesario

### Base de Datos Complementaria: ElastiCache (Redis)

**Para:** Caching, sessions, rate limiting, real-time leaderboards

**Configuración:**
- **Cluster Mode:** Activado (sharding automático)
- **Replicas:** 3 nodos por shard (alta disponibilidad)
- **Node Type:** r6g.xlarge (memory optimized)
- **Persistence:** AOF (Append Only File) para durabilidad

**Casos de uso:**
```
Sessions: SET user:{userId}:session {sessionData} EX 3600
Rate Limiting: INCR rate:{userId}:{endpoint} + EXPIRE
Auction Cache: SET auction:{auctionId} {auctionData} EX 300
Leaderboards: ZADD leaderboard:{auctionId} {bidAmount} {userId}
```

### Base de Datos para Búsqueda: OpenSearch

**Para:** Search Service

**Configuración:**
- **Version:** OpenSearch 2.x
- **Cluster:** 3 master nodes, 6+ data nodes
- **Replication:** 2 replicas por shard
- **Index:** `items` con full-text search

**Schema:**
```json
{
  "mappings": {
    "properties": {
      "title": { "type": "text", "analyzer": "standard" },
      "description": { "type": "text" },
      "category": { "type": "keyword" },
      "price": { "type": "integer" },
      "artist": { "type": "text" },
      "status": { "type": "keyword" },
      "createdAt": { "type": "date" },
      "tags": { "type": "keyword" }
    }
  }
}
```

**Sincronización:**
- DynamoDB Streams → Lambda → OpenSearch (indexación automática)
- Eventual consistency aceptable para búsqueda

### Base de Datos para Analytics: DynamoDB + S3

**Para:** Logs, analytics históricos

- **CloudWatch Logs** → S3 (archival)
- **DynamoDB exports** → S3 → Athena (queries SQL)
- **Retención:** 90 días en OpenSearch, 1 año en S3

## Consecuencias

### Positivas

1. **Escalabilidad automática**
   - DynamoDB on-demand escala sin intervención
   - No hay límites prácticos de throughput
   - Auto-partitioning cuando crece el dataset

2. **Multi-región nativa**
   - Global Tables replican automáticamente a 3 regiones
   - Latencia <100ms para lecturas locales
   - Active-Active para lecturas, Active-Passive para escrituras

3. **Alta disponibilidad**
   - SLA de 99.99% de AWS (mejor que nuestro target de 99.9%)
   - Multi-AZ automático
   - No downtime para scaling

4. **Managed Service**
   - No administración de servidores
   - Backups automáticos
   - Patching automático
   - Monitoring integrado (CloudWatch)

5. **Performance predecible**
   - Latencia single-digit milliseconds
   - Throughput ilimitado (on-demand mode)
   - Índices automáticos (GSI)

6. **Costo eficiente**
   - On-demand: paga solo por uso
   - No sobreaprovisionamiento
   - Free tier generoso (25 GB storage, 25 WCU/RCU)

7. **Seguridad y compliance**
   - Encryption at rest (KMS)
   - VPC Endpoints para acceso privado
   - IAM para control de acceso granular
   - PITR para recuperación

8. **Cache layer (Redis)**
   - Latencia sub-millisecond para datos calientes
   - Reduce load en DynamoDB (menor costo)
   - Casos de uso especializados (rate limiting, sessions)

9. **Búsqueda potente (OpenSearch)**
   - Full-text search con relevancia
   - Aggregations para analytics
   - Filtros complejos y faceted search

### Negativas

1. **Modelo de datos NoSQL**
   - Requiere diseñar schema basado en patrones de acceso
   - No hay joins (necesidad de denormalización)
   - Menos flexible que SQL para queries ad-hoc

2. **Consistencia eventual**
   - Global Tables son eventually consistent
   - Lecturas pueden no reflejar escrituras recientes
   - Necesidad de strongly consistent reads donde crítico

3. **Complejidad de modelado**
   - Single-table design es complejo de diseñar
   - Necesidad de GSI bien pensados
   - Difícil cambiar schema después

4. **Costos variables**
   - On-demand puede ser costoso con spikes inesperados
   - GSI cuentan en costos de storage y throughput
   - Necesidad de monitoring de costos

5. **Query limitations**
   - Solo queries por PK o GSI
   - Scan operations costosas (evitar en producción)
   - No hay LIKE, no hay full-text search nativo

6. **Item size limit**
   - 400 KB por item (documentos grandes deben ir a S3)
   - Descripciones largas de artículos pueden ser problema

7. **Vendor lock-in**
   - DynamoDB es AWS-specific
   - Migración a otro proveedor compleja
   - APIs propietarias

8. **Multi-DB complexity**
   - Sincronización entre DynamoDB y OpenSearch
   - Eventual consistency entre bases
   - Más puntos de fallo

### Neutrales

1. **DynamoDB Streams**
   - Útil para event sourcing y replicación
   - Necesidad de Lambda functions para procesamiento

2. **Backup strategy**
   - PITR para recovery granular
   - Snapshots diarios para compliance
   - S3 exports para analytics

3. **Migration strategy**
   - Plan de migración si necesitamos cambiar DB
   - Abstracción en código (repository pattern)

## Alternativas Consideradas

### Alternativa 1: PostgreSQL (RDS o Aurora)
**Descripción:** Base de datos SQL relacional managed

**Ventajas:**
- Modelo relacional familiar
- Joins y queries complejas
- ACID transactions
- Más flexible para queries ad-hoc
- Amplio ecosistema y herramientas

**Razones para rechazar:**
- **Escalabilidad vertical limitada:** Necesita sharding manual a escala
- **Multi-región compleja:** Aurora Global Database más costoso y complejo
- **Performance:** Read replicas tienen lag, no tan rápido como DynamoDB
- **Operacional:** Necesita más tuning (vacuuming, indexing)
- **Costo:** Aurora más costoso que DynamoDB on-demand a nuestra escala
- **Availability:** 99.95% (Aurora) vs 99.99% (DynamoDB)

### Alternativa 2: MongoDB Atlas
**Descripción:** Base de datos NoSQL document-oriented managed

**Ventajas:**
- Flexible schema
- Rich query language
- Aggregation framework
- Buena para datos semi-estructurados
- Multi-cloud (no vendor lock-in)

**Razones para rechazar:**
- **Multi-región:** Menos maduro que DynamoDB Global Tables
- **Costo:** Atlas más costoso que DynamoDB
- **Performance:** Latencia mayor que DynamoDB
- **AWS Integration:** Peor integración con servicios AWS
- **Expertise:** Equipo tiene más experiencia con DynamoDB

### Alternativa 3: Cassandra (Keyspaces)
**Descripción:** Wide-column NoSQL (AWS Keyspaces es Cassandra managed)

**Ventajas:**
- Excelente para write-heavy workloads
- Multi-región nativo
- Escalabilidad lineal
- Open source (menos lock-in)

**Razones para rechazar:**
- **Complejidad:** Más complejo de modelar que DynamoDB
- **Consistency:** Tunable consistency pero más complejo
- **Ecosystem:** Menos integración AWS que DynamoDB
- **Performance:** Comparable pero no superior
- **Expertise:** Equipo NO tiene experiencia con Cassandra

### Alternativa 4: Single Table Design en DynamoDB
**Descripción:** Usar una sola tabla DynamoDB para todos los servicios

**Ventajas:**
- Menos tablas que administrar
- Queries más eficientes (menos round trips)
- Mejor para relaciones many-to-many

**Razones para rechazar:**
- **Complejidad extrema:** Muy difícil de diseñar correctamente
- **Acoplamiento:** Servicios comparten schema
- **Riesgo:** Cambios en un servicio afectan otros
- **Learning curve:** Equipo necesita expertise avanzado
- **Decisión:** Preferimos **multiple tables** (una por servicio) para aislamiento

### Alternativa 5: FaunaDB
**Descripción:** Base de datos serverless multi-modelo

**Ventajas:**
- Transactions ACID distribuidas
- Multi-región nativo
- GraphQL nativo
- Serverless (auto-scaling)

**Razones para rechazar:**
- **Inmaduro:** Ecosistema menos establecido
- **Vendor lock-in:** Startup dependency risk
- **Costo:** Modelo de pricing menos predecible
- **AWS Integration:** No nativo a AWS

## Referencias

- PRD Sección 5.2: Stack Tecnológico - Bases de Datos
- AWS DynamoDB Best Practices: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/best-practices.html
- DynamoDB Book (Alex DeBrie)
- OpenSearch Documentation: https://opensearch.org/docs/

## Notas

### Patrón de Acceso a Datos

**Repository Pattern:**
```typescript
interface UserRepository {
  findById(userId: string): Promise<User>;
  findByEmail(email: string): Promise<User>;
  save(user: User): Promise<void>;
  delete(userId: string): Promise<void>;
}

class DynamoDBUserRepository implements UserRepository {
  // Implementación específica de DynamoDB
  // Permite cambiar DB sin afectar lógica de negocio
}
```

### Caching Strategy

```
┌──────────────┐
│  Application │
└──────┬───────┘
       │ Query
       ▼
┌──────────────┐
│ Redis Cache  │ ← Cache Hit (latency <1ms)
└──────┬───────┘
       │ Cache Miss
       ▼
┌──────────────┐
│  DynamoDB    │ ← Fetch from DB (latency ~10ms)
└──────────────┘
```

**TTL Strategy:**
- User profiles: 5 min
- Auction data (active): 30 sec
- Auction data (ended): 5 min
- Search results: 1 min

### Backup & Recovery

- **PITR (Point-in-Time Recovery):** Últimos 35 días
- **Snapshots diarios:** Retención 30 días
- **S3 exports:** Mensual para compliance
- **RTO:** 5 minutos (failover a otra región)
- **RPO:** 1 minuto (PITR)

## Historial

- 2025-11-07: Decisión inicial (Aceptado)
