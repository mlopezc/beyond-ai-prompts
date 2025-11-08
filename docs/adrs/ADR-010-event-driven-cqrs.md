# ADR-010: Event-Driven Architecture con Patrón CQRS

## Estado
Aceptado

## Contexto

ArtChain Auction Platform tiene características que se benefician de arquitectura event-driven:
- **Pujas en tiempo real:** Múltiples usuarios viendo y pujando simultáneamente
- **Notificaciones:** Eventos de puja, ganadores, pagos disparan notificaciones
- **Blockchain integration:** Registro de eventos en blockchain
- **Analytics:** Tracking de eventos para business intelligence
- **Audit trail:** Historial completo de acciones para compliance

**Requisitos del PRD relacionados:**
- RF-004: Proceso de puja en tiempo real
- RF-008: Notificaciones multi-canal
- Latencia <200ms (p95)
- 10,000 usuarios concurrentes
- Disponibilidad 99.9%
- Audit trail para compliance

**Problemas a resolver:**
1. **Read/Write diferentes:** Lectura de subastas muy frecuente, escritura menos
2. **Eventual consistency:** Blockchain tarda en confirmar, pero UI debe actualizar
3. **Desacoplamiento:** Servicios deben comunicarse sin dependencias síncronas
4. **Escalabilidad:** Reads y writes escalan independientemente

**Restricciones:**
- Microservicios (ya decidido en ADR-001)
- DynamoDB como DB (ya decidido en ADR-003)
- AWS SQS para messaging

## Decisión

Adoptaremos **Event-Driven Architecture** con patrón **CQRS (Command Query Responsibility Segregation)** para servicios de escritura intensiva (especialmente Auction Service).

### Arquitectura General

```
┌─────────────────────────────────────────────────────────────┐
│                       Client (React)                         │
└────────────┬────────────────────────────┬────────────────────┘
             │ Commands                   │ Queries
             │ (write)                    │ (read)
             ▼                            ▼
┌─────────────────────┐         ┌──────────────────────┐
│   Command API       │         │    Query API         │
│   (Write Model)     │         │    (Read Model)      │
└──────────┬──────────┘         └──────────┬───────────┘
           │                               │
           │ Events                        │
           ▼                               │
┌──────────────────────┐                  │
│   Event Store        │                  │
│   (DynamoDB Stream)  │                  │
└──────────┬───────────┘                  │
           │ Projections                  │
           └──────────────────────────────▼
                          ┌──────────────────────┐
                          │  Read Models         │
                          │  (Denormalized)      │
                          │  - Auction List      │
                          │  - Auction Details   │
                          │  - User Bids         │
                          └──────────────────────┘
```

### 1. Commands (Write Operations)

**Command:** Intención de cambiar estado del sistema

```typescript
// commands/PlaceBidCommand.ts
export interface PlaceBidCommand {
  type: 'PlaceBid';
  payload: {
    auctionId: string;
    userId: string;
    amount: number;
    timestamp: number;
  };
  metadata: {
    commandId: string;
    userId: string;
    correlationId: string;
  };
}

// Command Handler
export class PlaceBidHandler {
  constructor(
    private auctionRepository: AuctionRepository,
    private bidValidator: BidValidator,
    private eventBus: EventBus
  ) {}

  async handle(command: PlaceBidCommand): Promise<void> {
    const { auctionId, userId, amount } = command.payload;

    // 1. Load aggregate (current state)
    const auction = await this.auctionRepository.findById(auctionId);

    if (!auction) {
      throw new AuctionNotFoundError(auctionId);
    }

    // 2. Validate business rules
    await this.bidValidator.validate(auction, { userId, amount });

    // 3. Execute business logic (produces event)
    const event = auction.placeBid({ userId, amount });

    // 4. Persist event
    await this.auctionRepository.save(auction);

    // 5. Publish event
    await this.eventBus.publish(event);
  }
}
```

### 2. Events (Things that Happened)

**Event:** Algo que ya ocurrió (inmutable)

```typescript
// events/BidPlacedEvent.ts
export interface BidPlacedEvent {
  type: 'BidPlaced';
  aggregateId: string; // auctionId
  aggregateType: 'Auction';
  eventId: string;
  timestamp: number;
  version: number;
  payload: {
    auctionId: string;
    bidId: string;
    userId: string;
    amount: number;
    previousPrice: number;
    newPrice: number;
  };
  metadata: {
    correlationId: string;
    causationId: string; // commandId que causó este evento
  };
}
```

### 3. Event Store (DynamoDB)

**Tabla de eventos:**
```
Table: Events
PK: aggregateId (auctionId)
SK: version#timestamp
Attributes:
  - eventId (UUID)
  - eventType (BidPlaced, AuctionCreated, etc.)
  - payload (JSON)
  - metadata
  - timestamp

GSI: EventTypeIndex
PK: eventType
SK: timestamp
```

**Ejemplo:**
```typescript
// Event persistence
export class DynamoDBEventStore {
  async append(event: DomainEvent): Promise<void> {
    const item = {
      PK: event.aggregateId,
      SK: `${event.version}#${event.timestamp}`,
      eventId: event.eventId,
      eventType: event.type,
      payload: event.payload,
      metadata: event.metadata,
      timestamp: event.timestamp,
      ttl: Math.floor(Date.now() / 1000) + 365 * 24 * 60 * 60, // 1 año
    };

    await this.dynamoClient.put({
      TableName: 'Events',
      Item: item,
      ConditionExpression: 'attribute_not_exists(PK)', // Optimistic concurrency
    });
  }

  async getEvents(aggregateId: string): Promise<DomainEvent[]> {
    const result = await this.dynamoClient.query({
      TableName: 'Events',
      KeyConditionExpression: 'PK = :aggregateId',
      ExpressionAttributeValues: {
        ':aggregateId': aggregateId,
      },
      ScanIndexForward: true, // Orden cronológico
    });

    return result.Items?.map(this.toDomainEvent) || [];
  }
}
```

### 4. Event Sourcing (Auction Aggregate)

**Aggregate:** Entidad que se reconstruye desde eventos

```typescript
// aggregates/Auction.ts
export class Auction {
  id: string;
  title: string;
  currentPrice: number;
  status: AuctionStatus;
  bids: Bid[] = [];
  version: number = 0;

  private uncommittedEvents: DomainEvent[] = [];

  // Event Sourcing: Reconstruir estado desde eventos
  static fromEvents(events: DomainEvent[]): Auction {
    const auction = new Auction();
    events.forEach((event) => auction.apply(event, false));
    return auction;
  }

  // Business logic: Place bid
  placeBid(params: { userId: string; amount: number }): BidPlacedEvent {
    // Validate
    if (this.status !== 'ACTIVE') {
      throw new AuctionNotActiveError();
    }

    if (params.amount <= this.currentPrice) {
      throw new BidTooLowError();
    }

    // Create event
    const event: BidPlacedEvent = {
      type: 'BidPlaced',
      aggregateId: this.id,
      aggregateType: 'Auction',
      eventId: uuid(),
      timestamp: Date.now(),
      version: this.version + 1,
      payload: {
        auctionId: this.id,
        bidId: uuid(),
        userId: params.userId,
        amount: params.amount,
        previousPrice: this.currentPrice,
        newPrice: params.amount,
      },
      metadata: {},
    };

    // Apply event (update state)
    this.apply(event, true);

    return event;
  }

  // Apply event to state
  private apply(event: DomainEvent, isNew: boolean): void {
    switch (event.type) {
      case 'AuctionCreated':
        this.id = event.payload.auctionId;
        this.title = event.payload.title;
        this.currentPrice = event.payload.startingPrice;
        this.status = 'DRAFT';
        break;

      case 'BidPlaced':
        this.currentPrice = event.payload.newPrice;
        this.bids.push({
          id: event.payload.bidId,
          userId: event.payload.userId,
          amount: event.payload.amount,
          timestamp: event.timestamp,
        });
        break;

      case 'AuctionEnded':
        this.status = 'ENDED';
        break;
    }

    this.version = event.version;

    if (isNew) {
      this.uncommittedEvents.push(event);
    }
  }

  getUncommittedEvents(): DomainEvent[] {
    return this.uncommittedEvents;
  }

  markEventsAsCommitted(): void {
    this.uncommittedEvents = [];
  }
}
```

### 5. Read Models (Projections)

**Read Model:** Vista desnormalizada optimizada para queries

```typescript
// projections/AuctionListProjection.ts
export class AuctionListProjection {
  constructor(private dynamoClient: DynamoDBDocumentClient) {}

  // Event handler: Update read model when event occurs
  async onBidPlaced(event: BidPlacedEvent): Promise<void> {
    await this.dynamoClient.update({
      TableName: 'AuctionListView',
      Key: { auctionId: event.payload.auctionId },
      UpdateExpression: `
        SET currentPrice = :newPrice,
            bidCount = bidCount + :one,
            lastBidAt = :timestamp,
            lastBidder = :userId
      `,
      ExpressionAttributeValues: {
        ':newPrice': event.payload.newPrice,
        ':one': 1,
        ':timestamp': event.timestamp,
        ':userId': event.payload.userId,
      },
    });
  }

  async onAuctionCreated(event: AuctionCreatedEvent): Promise<void> {
    await this.dynamoClient.put({
      TableName: 'AuctionListView',
      Item: {
        auctionId: event.payload.auctionId,
        title: event.payload.title,
        currentPrice: event.payload.startingPrice,
        bidCount: 0,
        status: 'DRAFT',
        createdAt: event.timestamp,
      },
    });
  }

  async onAuctionEnded(event: AuctionEndedEvent): Promise<void> {
    await this.dynamoClient.update({
      TableName: 'AuctionListView',
      Key: { auctionId: event.payload.auctionId },
      UpdateExpression: 'SET #status = :ended, endedAt = :timestamp',
      ExpressionAttributeNames: {
        '#status': 'status',
      },
      ExpressionAttributeValues: {
        ':ended': 'ENDED',
        ':timestamp': event.timestamp,
      },
    });
  }
}

// Query Service (Read API)
export class AuctionQueryService {
  constructor(private dynamoClient: DynamoDBDocumentClient) {}

  async getAuctions(filters: AuctionFilters): Promise<Auction[]> {
    // Query read model (optimized for reads)
    const result = await this.dynamoClient.scan({
      TableName: 'AuctionListView',
      FilterExpression: filters.status ? '#status = :status' : undefined,
      ExpressionAttributeNames: filters.status ? { '#status': 'status' } : undefined,
      ExpressionAttributeValues: filters.status
        ? { ':status': filters.status }
        : undefined,
    });

    return result.Items as Auction[];
  }

  async getAuction(id: string): Promise<Auction | null> {
    const result = await this.dynamoClient.get({
      TableName: 'AuctionListView',
      Key: { auctionId: id },
    });

    return result.Item as Auction | null;
  }
}
```

### 6. Event Bus (SQS + DynamoDB Streams)

**Opción 1: DynamoDB Streams (Intra-service)**
```typescript
// Lambda triggered by DynamoDB Stream
export const handler = async (event: DynamoDBStreamEvent) => {
  for (const record of event.Records) {
    if (record.eventName === 'INSERT') {
      const domainEvent = parseDomainEvent(record.dynamodb.NewImage);

      // Update projections
      await auctionListProjection.handle(domainEvent);
      await userBidsProjection.handle(domainEvent);

      // Publish to other services (SQS)
      await eventPublisher.publish(domainEvent);
    }
  }
};
```

**Opción 2: SQS (Inter-service)**
```typescript
// Publish event to SQS
export class SQSEventPublisher {
  async publish(event: DomainEvent): Promise<void> {
    await this.sqs.send(
      new SendMessageCommand({
        QueueUrl: 'https://sqs.../artchain-events-queue',
        MessageBody: JSON.stringify(event),
        MessageAttributes: {
          eventType: {
            DataType: 'String',
            StringValue: event.type,
          },
        },
      })
    );
  }
}

// Consumer (Notification Service)
export class EventConsumer {
  async consume(): Promise<void> {
    while (true) {
      const { Messages } = await this.sqs.send(
        new ReceiveMessageCommand({
          QueueUrl: '...',
          MaxNumberOfMessages: 10,
          WaitTimeSeconds: 20,
        })
      );

      if (!Messages) continue;

      for (const message of Messages) {
        const event = JSON.parse(message.Body!);

        await this.handleEvent(event);

        await this.sqs.send(
          new DeleteMessageCommand({
            QueueUrl: '...',
            ReceiptHandle: message.ReceiptHandle,
          })
        );
      }
    }
  }

  private async handleEvent(event: DomainEvent): Promise<void> {
    switch (event.type) {
      case 'BidPlaced':
        await this.notificationService.sendBidNotification(event);
        break;
      case 'AuctionEnded':
        await this.notificationService.sendWinnerNotification(event);
        break;
    }
  }
}
```

### 7. Saga Pattern (Distributed Transactions)

**Ejemplo: Payment Saga**
```typescript
// Saga: Coordinar proceso de pago (múltiples servicios)
export class PaymentSaga {
  async execute(auctionId: string): Promise<void> {
    const auction = await this.getAuction(auctionId);
    const winner = auction.winnerId;

    try {
      // Step 1: Process payment
      const payment = await this.paymentService.charge({
        userId: winner,
        amount: auction.finalPrice,
      });

      // Step 2: Transfer funds to seller
      await this.paymentService.transfer({
        from: 'escrow',
        to: auction.sellerId,
        amount: auction.finalPrice * 0.92, // 8% commission
      });

      // Step 3: Update auction status
      await this.auctionService.markAsPaid(auctionId);

      // Step 4: Issue NFT
      await this.blockchainService.transferOwnership({
        itemId: auction.itemId,
        from: auction.sellerId,
        to: winner,
      });

      // Step 5: Send notifications
      await this.notificationService.sendPaymentConfirmation(winner);

    } catch (error) {
      // Compensating transactions (rollback)
      await this.rollback(auctionId, error);
    }
  }

  private async rollback(auctionId: string, error: Error): Promise<void> {
    // Refund payment
    // Revert auction status
    // Notify user of failure
  }
}
```

## Consecuencias

### Positivas

1. **Scalability**
   - Read y Write escalan independientemente
   - Read models desnormalizados (queries rápidas)
   - Writes no bloquean reads

2. **Performance**
   - Queries optimizadas (no joins, denormalizado)
   - Eventual consistency aceptable para UI
   - Caching agresivo de read models

3. **Audit Trail**
   - Todos los eventos almacenados (compliance)
   - Replay de eventos para debugging
   - Time-travel: Reconstruir estado en cualquier momento

4. **Flexibility**
   - Múltiples read models para diferentes vistas
   - Agregar nuevo read model sin cambiar writes
   - Business logic en un solo lugar (aggregate)

5. **Resilience**
   - Desacoplamiento vía eventos
   - Fallo de un consumer no afecta otros
   - Retry automático (SQS)

6. **Real-time**
   - Eventos disparan actualizaciones inmediatas
   - WebSocket updates desde eventos

### Negativas

1. **Complejidad**
   - Curva de aprendizaje (CQRS, Event Sourcing)
   - Más código que CRUD simple
   - Debugging más difícil (distributed)

2. **Eventual Consistency**
   - Read models pueden estar desactualizados
   - UI debe manejar (optimistic updates)

3. **Event Schema Evolution**
   - Breaking changes en eventos difíciles
   - Necesidad de versionado

4. **Storage**
   - Event store crece infinitamente
   - Snapshots necesarios para performance

5. **Operational Overhead**
   - Monitoring de projections
   - Asegurar consumers procesando eventos

### Neutrales

1. **Event Replay**
   - Útil para recovery
   - Pero puede ser lento para aggregates grandes

2. **Snapshots**
   - Necesarios para performance
   - Pero agregan complejidad

## Alternativas Consideradas

### Alternativa 1: CRUD Simple (No CQRS)
**Razones para rechazar:**
- Read/write patterns muy diferentes (auction list vs place bid)
- Difícil escalar reads y writes independientemente

### Alternativa 2: Solo Event-Driven (Sin CQRS)
**Razones para rechazar:**
- Queries complejas requieren denormalización
- CQRS optimiza reads

### Alternativa 3: Event Sourcing puro (Todo event-sourced)
**Razones para rechazar:**
- Overkill para entidades simples (Users, Categories)
- Solo aplicar a aggregates complejos (Auctions)

## Referencias

- CQRS Pattern: Martin Fowler
- Event Sourcing: Greg Young
- Domain-Driven Design: Eric Evans

## Historial

- 2025-11-07: Decisión inicial (Aceptado)
