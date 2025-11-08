# ADR-004: AWS como Cloud Provider con Kubernetes (EKS)

## Estado
Aceptado

## Contexto

ArtChain Auction Platform necesita infraestructura cloud que soporte:
- Multi-región global (US, EU, APAC)
- Alta disponibilidad (99.9%)
- Autoscaling automático
- Servicios managed (databases, caching, messaging)
- Containerización y orquestación
- CDN global
- Seguridad y compliance (GDPR, PCI-DSS)

**Requisitos del PRD relacionados:**
- 10,000 usuarios concurrentes
- Throughput de 100,000 requests/min
- Latencia <200ms (p95)
- Disponibilidad 99.9%
- Multi-región: US-EAST-1, EU-WEST-1, AP-SOUTHEAST-1
- RTO: 5 minutos, RPO: 1 minuto

**Servicios necesarios:**
- Compute (containers)
- Database (NoSQL, cache)
- Storage (objects, backups)
- Messaging (queues)
- CDN
- Monitoring & logging
- Security (WAF, DDoS protection)

**Restricciones:**
- Budget: $25K/mes infraestructura (producción)
- Equipo tiene experiencia en AWS
- Timeline: 4 meses para MVP
- Preferencia por managed services (menos ops overhead)

## Decisión

Adoptaremos **AWS como cloud provider principal** con la siguiente arquitectura:

### Infraestructura Core

#### Región Principal: US-EAST-1 (Virginia)
**Razón:** Región más grande, más servicios, mejor pricing, latencia baja para usuarios US

#### Regiones Secundarias:
- **EU-WEST-1 (Irlanda):** EMEA users
- **AP-SOUTHEAST-1 (Singapur):** APAC users

### Servicios AWS

#### 1. Compute: Amazon EKS (Elastic Kubernetes Service)

**Configuración:**
```yaml
Cluster:
  Version: 1.28+
  Multi-AZ: 3 availability zones
  Control Plane: AWS Managed (HA)

Node Groups:
  general-purpose:
    Instance Type: t3.xlarge
    Min: 2, Max: 10
    Auto Scaling: Enabled

  compute-optimized:
    Instance Type: c6i.2xlarge
    Min: 2, Max: 8
    Use: Auction Service (high CPU)

  memory-optimized:
    Instance Type: r6i.xlarge
    Min: 2, Max: 4
    Use: Search Service, Cache

Fargate Profiles:
  Use: Jobs esporádicos, batch processing
  Cost: Pay-per-use (no EC2 overhead)
```

**Características:**
- **Cluster Autoscaler:** Scale nodes automáticamente
- **Horizontal Pod Autoscaler (HPA):** Scale pods based on CPU/memory/custom metrics
- **Metrics Server:** Para HPA
- **Karpenter (Fase 2):** Provisioning más eficiente que Cluster Autoscaler

**Razones:**
- Kubernetes-native (portable, evita lock-in)
- AWS-managed control plane (99.95% SLA)
- Integración perfecta con AWS services
- Autoscaling robusto
- Equipo tiene experiencia en K8s

#### 2. Database: DynamoDB (ver ADR-003)
- On-demand billing
- Global Tables (multi-región)
- Point-in-time recovery

#### 3. Caching: ElastiCache (Redis)
- Cluster mode enabled
- Multi-AZ (3 replicas)
- r6g.xlarge nodes

#### 4. Search: OpenSearch
- 3 master nodes, 6 data nodes
- Multi-AZ deployment
- Automated snapshots

#### 5. Storage: Amazon S3

**Buckets:**
```
artchain-assets-prod:
  Purpose: Imágenes y videos de artículos
  Storage Class: Standard → IA (90d) → Glacier (1y)
  Versioning: Enabled
  Replication: Cross-region a EU-WEST-1 y AP-SOUTHEAST-1

artchain-documents-prod:
  Purpose: Certificados y documentos legales
  Encryption: SSE-KMS
  MFA Delete: Enabled

artchain-backups-prod:
  Purpose: Backups de base de datos
  Storage Class: Glacier Instant Retrieval

artchain-logs-prod:
  Purpose: Application logs
  Lifecycle: Expiración 90 días
```

#### 6. Messaging: Amazon SQS

**Colas:**
```
artchain-bids-queue.fifo:
  Type: FIFO (orden garantizado)
  Use: Procesamiento de pujas
  Visibility Timeout: 60s

artchain-notifications-queue:
  Type: Standard (high throughput)
  Use: Email, SMS, Push

artchain-blockchain-queue:
  Type: Standard
  Use: Transacciones blockchain

artchain-payments-queue.fifo:
  Type: FIFO
  Use: Procesamiento de pagos

artchain-dlq:
  Type: Standard
  Use: Dead letter queue
  Max Receive Count: 3
```

#### 7. CDN: CloudFront
- 200+ edge locations globally
- Custom SSL certificate (ACM)
- Geo-restriction (si necesario)
- Cache policies:
  - Static assets (images, CSS, JS): 24h
  - API responses: 5 min (cacheable endpoints)

#### 8. DNS: Route 53
- Hosted zone para artchain.com
- Geoproximity routing (latencia-based)
- Health checks activos
- Failover automático

#### 9. Load Balancing: Application Load Balancer (ALB)
- Multi-AZ
- Target groups para EKS pods
- SSL termination
- Path-based routing

#### 10. Monitoring & Logging

**CloudWatch:**
- Custom dashboards
- Alarms (PagerDuty integration)
- Logs centralizados
- Metrics (business + technical)

**X-Ray:**
- Distributed tracing
- Service map
- Performance insights

**OpenSearch (logs):**
- Centralized logging
- 90 días retention (hot)
- 1 año retention (warm)

#### 11. Security

**AWS WAF (Web Application Firewall):**
- SQL injection protection
- XSS protection
- Rate-based rules
- Geographic restrictions (opcional)

**AWS Shield Standard:**
- DDoS protection (incluido)
- Protección layer 3/4

**Secrets Manager:**
- API keys, DB credentials
- Rotación automática (90 días)

**IAM:**
- Least privilege principle
- Roles para servicios
- MFA para acceso humano

**VPC:**
- Public subnets (ALB, NAT Gateway)
- Private subnets (EKS nodes, DBs)
- Security Groups (firewall stateful)
- NACLs (firewall stateless)

#### 12. Otros Servicios

**SES (Simple Email Service):**
- Emails transaccionales
- High deliverability

**SNS (Simple Notification Service):**
- Push notifications
- SMS (via Twilio integration)

**ACM (Certificate Manager):**
- SSL/TLS certificates
- Auto-renewal

**KMS (Key Management Service):**
- Encryption keys
- Envelope encryption

### Arquitectura de Red

```
┌────────────────── CloudFront (Global CDN) ──────────────────┐
│                   Edge Locations (200+)                       │
└───────────────────────┬───────────────────────────────────────┘
                        │
┌───────────────────────▼───────────────────────────────────────┐
│                  Route 53 (DNS)                               │
│            Geoproximity Routing + Health Checks                │
└───┬──────────────────┬────────────────────┬───────────────────┘
    │                  │                    │
┌───▼─────────┐  ┌─────▼────────┐  ┌───────▼────────┐
│  US-EAST-1  │  │  EU-WEST-1   │  │ AP-SOUTHEAST-1 │
│  (Primary)  │  │ (Secondary)  │  │  (Secondary)   │
└─────────────┘  └──────────────┘  └────────────────┘

Cada región contiene:
┌─────────────────────────────────────────┐
│             VPC (Multi-AZ)              │
│  ┌───────────────────────────────────┐  │
│  │  Public Subnets (3 AZs)           │  │
│  │  - ALB                            │  │
│  │  - NAT Gateway                    │  │
│  └───────────────────────────────────┘  │
│  ┌───────────────────────────────────┐  │
│  │  Private Subnets (3 AZs)          │  │
│  │  - EKS Worker Nodes               │  │
│  │  - ElastiCache (Redis)            │  │
│  │  - OpenSearch                     │  │
│  └───────────────────────────────────┘  │
│                                         │
│  DynamoDB (Global Tables)               │
│  S3 (Cross-region replication)          │
└─────────────────────────────────────────┘
```

### Estrategia de Deployment

**Fase 1 (MVP):**
- Deploy solo en US-EAST-1
- Capacidad para 1,000 concurrentes

**Fase 2 (Escalamiento):**
- Deploy en EU-WEST-1 y AP-SOUTHEAST-1
- Capacidad para 10,000 concurrentes
- Failover automático

**CI/CD:**
- GitHub Actions para CI
- ArgoCD (Fase 2) para GitOps
- Blue-Green deployments
- Canary releases para servicios críticos

## Consecuencias

### Positivas

1. **Servicios Managed**
   - Menos overhead operacional
   - Patching automático
   - Backups automáticos
   - Alta disponibilidad out-of-the-box

2. **Escalabilidad Global**
   - Multi-región nativa
   - CDN con 200+ edge locations
   - DynamoDB Global Tables
   - Route 53 latency-based routing

3. **Performance**
   - Infraestructura global de AWS (baja latencia)
   - EKS autoscaling rápido
   - CloudFront caching agresivo

4. **Seguridad**
   - Compliance: PCI-DSS, GDPR, SOC 2
   - WAF, Shield, Secrets Manager incluidos
   - VPC isolation
   - IAM granular

5. **Integración**
   - Todos los servicios se integran nativamente
   - CloudWatch monitoring unificado
   - X-Ray tracing cross-service
   - IAM roles para servicios

6. **Ecosystem**
   - Documentación extensa
   - Comunidad grande
   - Soporte enterprise (si necesario)
   - Marketplace de soluciones

7. **Expertise del equipo**
   - Equipo DevOps tiene experiencia en AWS
   - Menos learning curve
   - Faster time to market

8. **Costo predecible**
   - Calculadoras de costo
   - Free tier generoso
   - Reserved Instances para savings
   - Spot Instances para batch jobs

9. **Kubernetes-native (EKS)**
   - Portable (reduce lock-in)
   - Ecosistema Kubernetes (Helm, Operators)
   - Multi-cloud future si necesario

### Negativas

1. **Vendor Lock-in**
   - Servicios propietarios (DynamoDB, SQS, etc.)
   - Migración a otro cloud compleja y costosa
   - APIs específicas de AWS

2. **Complejidad**
   - Muchos servicios que configurar
   - IAM policies complejas
   - Networking (VPC, subnets, routing) tiene curva de aprendizaje

3. **Costos variables**
   - Difícil predecir exactamente (on-demand scaling)
   - Necesidad de monitoring constante
   - Riesgo de bill shock si mal configurado

4. **Soporte limitado (sin plan Enterprise)**
   - Support básico solo incluye billing
   - Technical support requiere pago adicional ($100-$15K/mes)

5. **Outages**
   - AWS no es inmune a outages (ej. us-east-1 diciembre 2021)
   - Dependencia de disponibilidad de AWS

6. **EKS Costs**
   - $0.10/hora por cluster ($73/mes por cluster)
   - 3 regiones = $219/mes solo por control plane
   - EC2 nodes adicionales

7. **Learning curve para nuevos**
   - Onboarding de nuevos ingenieros requiere tiempo
   - Certificaciones AWS útiles pero costosas

### Neutrales

1. **Multi-cloud future**
   - Kubernetes permite portabilidad parcial
   - Servicios managed (DynamoDB) no son portables
   - Podríamos usar abstracciones (ej. Pulumi)

2. **Infrastructure as Code**
   - Terraform recomendado (multi-cloud)
   - CloudFormation alternativa (AWS-only)

3. **Monitoring**
   - CloudWatch suficiente para MVP
   - Datadog/NewRelic para observabilidad avanzada (Fase 2)

## Alternativas Consideradas

### Alternativa 1: Google Cloud Platform (GCP)
**Ventajas:**
- GKE (Kubernetes) mejor que EKS (más features)
- BigQuery excelente para analytics
- Networking más simple
- Precios más bajos en compute

**Razones para rechazar:**
- Equipo NO tiene experiencia en GCP
- Menos servicios managed que AWS
- Ecosistema menor
- DynamoDB no tiene equivalente directo (Firestore diferente)
- Learning curve de 4 meses no permite

### Alternativa 2: Microsoft Azure
**Ventajas:**
- AKS (Kubernetes) gratuito (no cobra por control plane)
- Integración con Microsoft ecosystem
- Cosmos DB (multi-modelo)

**Razones para rechazar:**
- Equipo NO tiene experiencia en Azure
- Menos servicios que AWS
- UI/UX más compleja
- Pricing más difícil de entender

### Alternativa 3: Multi-cloud (AWS + GCP)
**Ventajas:**
- No vendor lock-in
- Mejor pricing (puede elegir)
- Redundancia cross-cloud

**Razones para rechazar:**
- **Complejidad extrema:** 2x configuración, monitoring, expertise
- **Costos:** Networking cross-cloud muy costoso
- **Timeline:** 4 meses no permite
- **Equipo:** No tenemos expertise en ambos
- **Decisión:** No justificado para startup temprana

### Alternativa 4: Serverless (AWS Lambda + API Gateway)
**Ventajas:**
- Auto-scaling infinito
- Paga solo por uso
- No administración de servidores

**Razones para rechazar:**
- Cold starts afectan latencia (<2s para pujas crítico)
- Límites de timeout (15 min)
- Difícil para WebSockets (real-time updates)
- Vendor lock-in extremo
- Costos impredecibles a escala

### Alternativa 5: Bare Metal / Self-hosted
**Ventajas:**
- Control total
- Potencialmente más barato a gran escala
- No vendor lock-in

**Razones para rechazar:**
- **Overhead operacional:** Necesitaríamos equipo SRE grande
- **Availability:** Difícil lograr 99.9% sin expertise
- **Multi-región:** Muy complejo y costoso
- **Timeline:** Imposible en 4 meses
- **Riesgo:** Hardware failures, networking issues

### Alternativa 6: Hybrid (AWS + On-premise)
**Razones para rechazar:**
- Complejidad innecesaria
- No hay necesidad de on-premise
- Peor de ambos mundos

## Referencias

- PRD Sección 5.3: Infraestructura AWS
- AWS Well-Architected Framework: https://aws.amazon.com/architecture/well-architected/
- EKS Best Practices: https://aws.github.io/aws-eks-best-practices/

## Notas

### Estimación de Costos (Producción - 10K concurrentes)

```
EKS Control Plane: $219/mes (3 regiones)
EC2 Nodes (mixed instances): $8,000/mes
DynamoDB (on-demand): $3,000/mes
ElastiCache (Redis): $2,500/mes
OpenSearch: $3,500/mes
S3: $1,000/mes
CloudFront: $2,000/mes
ALB: $500/mes
Data Transfer: $2,500/mes
Other (SQS, SES, etc.): $1,000/mes

TOTAL: ~$24,219/mes ✅ (dentro de budget de $25K)
```

**Optimizaciones:**
- Reserved Instances (1 año): -30% en EC2
- Spot Instances (batch jobs): -70% en EC2
- S3 Intelligent Tiering: savings automáticos
- CloudFront Reserved Capacity: -20% en CDN

### Failover Strategy

**RTO (Recovery Time Objective): 5 minutos**
1. Route 53 health check detecta fallo (30 seg)
2. Route 53 redirige a región secundaria (30 seg)
3. Warm standby en regiones secundarias (ya running)
4. DynamoDB Global Tables ya sincronizadas

**RPO (Recovery Point Objective): 1 minuto**
- DynamoDB replication lag: <1 segundo típicamente
- PITR permite recovery a cualquier punto en últimos 35 días

### Infrastructure as Code

```hcl
# Terraform structure
terraform/
  ├── modules/
  │   ├── eks/
  │   ├── dynamodb/
  │   ├── s3/
  │   └── vpc/
  ├── environments/
  │   ├── dev/
  │   ├── staging/
  │   └── prod/
  └── main.tf
```

## Historial

- 2025-11-07: Decisión inicial (Aceptado)
