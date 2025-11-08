# ADR-006: Estrategia de API Híbrida (REST + gRPC)

## Estado
Aceptado

## Contexto

ArtChain Auction Platform tiene diferentes necesidades de comunicación:
- **Externa (Cliente ↔ Backend):** Web frontend, mobile app (Fase 3), partners API (Fase 3)
- **Interna (Servicio ↔ Servicio):** Microservicios que se comunican entre sí
- **Asíncrona (Event-driven):** Notificaciones, procesamiento de pujas, blockchain

**Requisitos del PRD relacionados:**
- Latencia API <200ms (p95), <500ms (p99)
- Throughput: 100,000 requests/min
- Real-time updates (WebSockets para pujas)
- API pública para integraciones (Fase 3)

**Características necesarias:**
- **Performance:** Baja latencia, alto throughput
- **Developer Experience:** Fácil de usar y documentar
- **Type Safety:** Contratos claros entre servicios
- **Streaming:** Real-time bidirectional communication
- **Compatibilidad:** Web browsers, mobile apps

**Restricciones:**
- Stack multi-lenguaje (Node.js, Java, React)
- Arquitectura de microservicios (comunicación inter-servicio frecuente)
- Timeline: 4 meses para MVP

## Decisión

Adoptaremos una **estrategia de API híbrida** usando diferentes protocolos según el caso de uso:

### 1. REST API (JSON) - Comunicación Externa

**Para:** Cliente (Web/Mobile) ↔ Backend

**Características:**
- HTTP/1.1 o HTTP/2
- JSON como formato de datos
- RESTful design principles
- OpenAPI 3.0 specification
- Versionado en URL (`/api/v1/...`)

**Endpoints principales:**
```
Auth:
  POST   /api/v1/auth/register
  POST   /api/v1/auth/login
  POST   /api/v1/auth/refresh
  POST   /api/v1/auth/logout

Users:
  GET    /api/v1/users/:id
  PATCH  /api/v1/users/:id
  DELETE /api/v1/users/:id

Auctions:
  GET    /api/v1/auctions
  POST   /api/v1/auctions
  GET    /api/v1/auctions/:id
  PATCH  /api/v1/auctions/:id
  DELETE /api/v1/auctions/:id

Bids:
  POST   /api/v1/auctions/:id/bids
  GET    /api/v1/auctions/:id/bids

Payments:
  POST   /api/v1/payments
  GET    /api/v1/payments/:id

Search:
  GET    /api/v1/search?q={query}&category={cat}&...
```

**API Gateway:**
- Kong o AWS API Gateway (evaluar costos)
- Rate limiting
- Request/Response transformation
- API key management (para API pública Fase 3)
- CORS handling
- SSL termination

**Response format estándar:**
```json
{
  "data": {
    "id": "auction-123",
    "title": "Vintage Painting",
    ...
  },
  "meta": {
    "timestamp": "2025-11-07T10:00:00Z",
    "requestId": "req-abc123"
  }
}

// Error response
{
  "error": {
    "code": "AUCTION_NOT_FOUND",
    "message": "Auction with ID auction-123 not found",
    "details": {}
  },
  "meta": {
    "timestamp": "2025-11-07T10:00:00Z",
    "requestId": "req-abc123"
  }
}
```

**HTTP Status Codes:**
```
200 OK - Success
201 Created - Resource created
204 No Content - Success (no response body)
400 Bad Request - Validation error
401 Unauthorized - Authentication required
403 Forbidden - Insufficient permissions
404 Not Found - Resource not found
409 Conflict - Resource conflict (ej. bid too low)
422 Unprocessable Entity - Business logic error
429 Too Many Requests - Rate limit exceeded
500 Internal Server Error - Server error
503 Service Unavailable - Service down
```

**Pagination:**
```
GET /api/v1/auctions?page=2&limit=20

Response:
{
  "data": [...],
  "pagination": {
    "page": 2,
    "limit": 20,
    "total": 150,
    "totalPages": 8,
    "hasNext": true,
    "hasPrev": true
  }
}
```

### 2. gRPC - Comunicación Interna (Servicio ↔ Servicio)

**Para:** Comunicación entre microservicios

**Características:**
- HTTP/2 (multiplexing, header compression)
- Protocol Buffers (binario, compacto, type-safe)
- Bidirectional streaming
- Code generation automático
- 4-5x más rápido que REST JSON

**Casos de uso:**
- User Service → Auction Service (verificar ownership)
- Auction Service → Blockchain Service (certificar artículo)
- Auction Service → Payment Service (procesar pago)
- API Gateway → Todos los servicios (proxy requests)

**Ejemplo Protocol Buffer:**
```protobuf
syntax = "proto3";

package artchain.auction.v1;

service AuctionService {
  rpc CreateAuction(CreateAuctionRequest) returns (Auction);
  rpc GetAuction(GetAuctionRequest) returns (Auction);
  rpc PlaceBid(PlaceBidRequest) returns (Bid);
  rpc GetAuctionBids(GetAuctionBidsRequest) returns (stream Bid);
}

message Auction {
  string id = 1;
  string title = 2;
  string description = 3;
  string seller_id = 4;
  int64 starting_price = 5;
  int64 current_price = 6;
  google.protobuf.Timestamp start_date = 7;
  google.protobuf.Timestamp end_date = 8;
  AuctionStatus status = 9;
}

enum AuctionStatus {
  AUCTION_STATUS_UNSPECIFIED = 0;
  AUCTION_STATUS_DRAFT = 1;
  AUCTION_STATUS_PENDING_APPROVAL = 2;
  AUCTION_STATUS_ACTIVE = 3;
  AUCTION_STATUS_ENDED = 4;
  AUCTION_STATUS_CANCELLED = 5;
}

message CreateAuctionRequest {
  string title = 1;
  string description = 2;
  int64 starting_price = 3;
  google.protobuf.Timestamp start_date = 4;
  google.protobuf.Timestamp end_date = 5;
}

message PlaceBidRequest {
  string auction_id = 1;
  string user_id = 2;
  int64 amount = 3;
}

message Bid {
  string id = 1;
  string auction_id = 2;
  string user_id = 3;
  int64 amount = 4;
  google.protobuf.Timestamp created_at = 5;
  BidStatus status = 6;
}
```

**Implementación Node.js:**
```typescript
// Server (Auction Service)
import * as grpc from '@grpc/grpc-js';
import * as protoLoader from '@grpc/proto-loader';

const packageDef = protoLoader.loadSync('auction.proto');
const grpcObject = grpc.loadPackageDefinition(packageDef);

const server = new grpc.Server();

server.addService(grpcObject.artchain.auction.v1.AuctionService.service, {
  createAuction: async (call, callback) => {
    const { title, description, starting_price } = call.request;
    // Business logic
    const auction = await auctionService.create({ title, description, starting_price });
    callback(null, auction);
  },

  getAuctionBids: (call) => {
    const { auction_id } = call.request;
    // Stream bids (real-time updates)
    const stream = bidService.watchBids(auction_id);
    stream.on('data', (bid) => call.write(bid));
    stream.on('end', () => call.end());
  },
});

server.bindAsync('0.0.0.0:50051', grpc.ServerCredentials.createInsecure(), () => {
  server.start();
});

// Client (API Gateway)
import * as grpc from '@grpc/grpc-js';
import * as protoLoader from '@grpc/proto-loader';

const packageDef = protoLoader.loadSync('auction.proto');
const grpcObject = grpc.loadPackageDefinition(packageDef);

const client = new grpcObject.artchain.auction.v1.AuctionService(
  'auction-service:50051',
  grpc.credentials.createInsecure()
);

// Call gRPC service
client.createAuction({ title, description, starting_price }, (err, response) => {
  if (err) {
    console.error(err);
    return;
  }
  console.log('Auction created:', response);
});
```

**Service Mesh (Fase 2):**
- Istio para mTLS entre servicios
- Load balancing automático
- Circuit breaking
- Observability (tracing, metrics)

### 3. WebSockets - Real-time Updates

**Para:** Actualizaciones en tiempo real de pujas

**Características:**
- Bidirectional communication
- Low latency (<100ms)
- Persistent connection
- Socket.io (library que abstrae WebSockets + fallbacks)

**Flujo:**
```
1. Cliente conecta al WebSocket server:
   socket.connect('wss://api.artchain.com')

2. Cliente se subscribe a auction room:
   socket.emit('subscribe:auction', { auctionId: 'auction-123' })

3. Cuando alguien hace bid:
   - Auction Service procesa bid
   - Publica evento a Redis Pub/Sub
   - WebSocket server escucha Redis
   - WebSocket server emite a todos los clientes en room:
     socket.to('auction-123').emit('bid:new', { bid })

4. Cliente recibe update en tiempo real:
   socket.on('bid:new', (bid) => {
     updateUI(bid);
   })
```

**Implementation (Socket.io Server):**
```typescript
import { Server } from 'socket.io';
import { createAdapter } from '@socket.io/redis-adapter';
import { createClient } from 'redis';

const io = new Server(server, {
  cors: { origin: 'https://artchain.com' }
});

// Redis adapter (para multi-instancia)
const pubClient = createClient({ url: 'redis://localhost:6379' });
const subClient = pubClient.duplicate();
io.adapter(createAdapter(pubClient, subClient));

// Authentication middleware
io.use(async (socket, next) => {
  const token = socket.handshake.auth.token;
  try {
    const user = await verifyJWT(token);
    socket.data.user = user;
    next();
  } catch (err) {
    next(new Error('Authentication failed'));
  }
});

io.on('connection', (socket) => {
  console.log(`User ${socket.data.user.id} connected`);

  // Subscribe to auction
  socket.on('subscribe:auction', ({ auctionId }) => {
    socket.join(`auction:${auctionId}`);
    console.log(`User subscribed to auction ${auctionId}`);
  });

  // Unsubscribe
  socket.on('unsubscribe:auction', ({ auctionId }) => {
    socket.leave(`auction:${auctionId}`);
  });

  socket.on('disconnect', () => {
    console.log('User disconnected');
  });
});

// Redis Pub/Sub listener
const subscriber = createClient();
subscriber.subscribe('auction:bid:new', (message) => {
  const { auctionId, bid } = JSON.parse(message);
  io.to(`auction:${auctionId}`).emit('bid:new', bid);
});
```

### 4. Async Messaging (SQS) - Event-Driven

**Para:** Comunicación asíncrona desacoplada

**Casos de uso:**
- Procesamiento de pujas (FIFO queue)
- Envío de notificaciones
- Transacciones blockchain
- Procesamiento de pagos

**Ventajas:**
- Desacoplamiento
- Resiliencia (retry automático)
- Buffering (absorbe spikes de tráfico)
- Order guarantee (FIFO queues)

**Ejemplo:**
```typescript
// Producer (Auction Service)
import { SQSClient, SendMessageCommand } from '@aws-sdk/client-sqs';

const sqs = new SQSClient({ region: 'us-east-1' });

async function publishBidEvent(bid: Bid) {
  await sqs.send(new SendMessageCommand({
    QueueUrl: 'https://sqs.us-east-1.amazonaws.com/.../artchain-bids-queue.fifo',
    MessageBody: JSON.stringify(bid),
    MessageGroupId: bid.auctionId, // FIFO grouping
    MessageDeduplicationId: bid.id, // Deduplication
  }));
}

// Consumer (Blockchain Service)
import { SQSClient, ReceiveMessageCommand, DeleteMessageCommand } from '@aws-sdk/client-sqs';

async function processBidQueue() {
  while (true) {
    const { Messages } = await sqs.send(new ReceiveMessageCommand({
      QueueUrl: '...',
      MaxNumberOfMessages: 10,
      WaitTimeSeconds: 20, // Long polling
    }));

    if (!Messages) continue;

    for (const message of Messages) {
      const bid = JSON.parse(message.Body);

      try {
        await registerBidOnBlockchain(bid);

        // Delete message (success)
        await sqs.send(new DeleteMessageCommand({
          QueueUrl: '...',
          ReceiptHandle: message.ReceiptHandle,
        }));
      } catch (err) {
        // Message will be retried (visibility timeout)
        console.error('Failed to process bid:', err);
      }
    }
  }
}
```

## Consecuencias

### Positivas

1. **Performance optimizado**
   - gRPC 4-5x más rápido que REST para comunicación interna
   - WebSockets para latencia mínima (<100ms) en real-time
   - HTTP/2 multiplexing reduce overhead

2. **Type Safety**
   - Protocol Buffers generan código con tipos
   - Menos errores en runtime
   - IntelliSense en desarrollo

3. **Developer Experience**
   - REST familiar para frontend developers
   - OpenAPI auto-genera documentación
   - Code generation de protobuf reduce boilerplate

4. **Escalabilidad**
   - gRPC load balancing eficiente
   - WebSockets con Redis adapter escala horizontalmente
   - SQS absorbe spikes de tráfico

5. **Resiliencia**
   - Async messaging con retry automático
   - Dead letter queues para mensajes fallidos
   - Circuit breakers en gRPC (con Resilience4j)

6. **Flexibilidad**
   - REST para API pública (universal compatibility)
   - gRPC para performance interna
   - WebSockets para real-time
   - SQS para desacoplamiento

7. **Streaming**
   - gRPC bidirectional streaming para casos avanzados
   - WebSockets para real-time bidirectional

### Negativas

1. **Complejidad**
   - 4 protocolos diferentes (REST, gRPC, WebSockets, SQS)
   - Más para aprender y mantener
   - Debugging más complejo

2. **Tooling**
   - gRPC requiere protoc compiler
   - Protocol Buffers menos human-readable que JSON
   - Debugging de binario más difícil

3. **Browser support**
   - gRPC no funciona directamente en browsers (necesita grpc-web + proxy)
   - Por eso usamos REST para frontend

4. **Overhead de desarrollo**
   - Definir .proto files
   - Mantener OpenAPI specs
   - Sincronizar contratos entre equipos

5. **Versionado**
   - Protocol Buffers requiere backward compatibility cuidadosa
   - Breaking changes complejos

6. **WebSocket infrastructure**
   - Más complejo que HTTP stateless
   - Sticky sessions si no usa Redis adapter
   - Más difícil de escalar horizontalmente (aunque Socket.io + Redis lo resuelve)

### Neutrales

1. **API Gateway**
   - Necesita soportar REST → gRPC translation
   - Kong, Envoy, o custom gateway

2. **Monitoring**
   - Métricas diferentes por protocolo
   - Tracing distribuido necesario (X-Ray)

## Alternativas Consideradas

### Alternativa 1: Solo REST para Todo
**Ventajas:**
- Más simple (un solo protocolo)
- Universal compatibility
- Tooling maduro

**Razones para rechazar:**
- **Performance:** REST JSON menos eficiente para comunicación interna
- **Real-time:** HTTP polling ineficiente vs WebSockets
- **Type safety:** JSON no tiene tipos nativos

### Alternativa 2: GraphQL
**Ventajas:**
- Cliente pide exactamente lo que necesita
- Un solo endpoint
- Schema with types
- Subscriptions para real-time

**Razones para rechazar:**
- **Complejidad:** N+1 query problem, caching complejo
- **Performance:** Overhead de resolvers
- **Backend complexity:** Más complejo implementar que REST
- **Over-fetching aún posible:** En queries complejas
- **Equipo no familiar:** Learning curve
- **Decisión:** REST suficiente para MVP, GraphQL en Fase 3 si necesario

### Alternativa 3: Solo gRPC para Todo (incluyendo frontend)
**Ventajas:**
- Performance uniforme
- Type safety en toda la stack

**Razones para rechazar:**
- **Browser support:** Necesita grpc-web + proxy (complejidad)
- **Tooling:** Less mature para browsers
- **Developer experience:** REST más familiar para frontend devs
- **Public API:** Partners prefieren REST

### Alternativa 4: Server-Sent Events (SSE) para Real-time
**Ventajas:**
- Más simple que WebSockets
- HTTP-based (fácil de proxy)
- Auto-reconnect

**Razones para rechazar:**
- **Unidirectional:** Solo server → client (no bidirectional)
- **Menos features:** No rooms, no broadcast
- **Performance:** Inferior a WebSockets
- **Use case:** Quizás para notificaciones simples, pero no para bidding real-time

### Alternativa 5: Apache Kafka en lugar de SQS
**Ventajas:**
- Más features (stream processing, replay)
- Mejor para event sourcing
- Open source (menos lock-in)

**Razones para rechazar:**
- **Complejidad operacional:** Necesita cluster management (Zookeeper)
- **Overhead:** Overkill para MVP
- **Costo:** EC2 instances vs SQS serverless
- **Expertise:** Equipo no tiene experiencia
- **Decisión:** SQS suficiente para Fase 1, Kafka en Fase 3 si necesario

## Referencias

- PRD Sección 5.2: Stack Tecnológico
- gRPC Documentation: https://grpc.io/docs/
- Protocol Buffers: https://protobuf.dev/
- Socket.io Documentation: https://socket.io/docs/
- REST API Design Best Practices

## Notas

### API Versioning Strategy

**URL Versioning (REST):**
```
/api/v1/auctions
/api/v2/auctions (cuando hay breaking change)
```

**Protocol Buffers Versioning:**
```protobuf
syntax = "proto3";
package artchain.auction.v1; // v1, v2, etc.
```

**Deprecation Policy:**
- Old version supported por 6 meses después de new version
- Warnings enviados en headers:
  ```
  Deprecation: true
  Sunset: Sat, 31 Dec 2025 23:59:59 GMT
  Link: <https://api.artchain.com/v2/auctions>; rel="successor-version"
  ```

### OpenAPI Specification

**Location:** `/docs/api/openapi.yaml`

**Auto-generation:**
- NestJS: @nestjs/swagger
- Spring Boot: Springdoc OpenAPI

**Documentation UI:**
- Swagger UI: https://api.artchain.com/docs
- ReDoc: https://api.artchain.com/redoc

### gRPC Service Discovery

**Fase 1:**
- Hardcoded service addresses (environment variables)
- Kubernetes DNS (`auction-service.default.svc.cluster.local:50051`)

**Fase 2:**
- Service mesh (Istio) con auto-discovery
- Client-side load balancing

## Historial

- 2025-11-07: Decisión inicial (Aceptado)
