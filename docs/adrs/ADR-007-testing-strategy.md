# ADR-007: Estrategia de Testing Multi-Layer

## Estado
Aceptado

## Contexto

ArtChain Auction Platform maneja transacciones financieras y certificación blockchain, donde bugs pueden resultar en:
- Pérdida financiera para usuarios
- Fraude en subastas
- Certificados de autenticidad incorrectos
- Violaciones de compliance

**Requisitos del PRD relacionados:**
- Disponibilidad: 99.9% uptime
- Performance: <200ms latency (p95)
- Seguridad: PCI-DSS, GDPR compliance
- Calidad: <1 bug crítico/mes en producción
- Métricas: Code coverage >80%

**Tipos de testing necesarios:**
1. Unit tests (lógica de negocio aislada)
2. Integration tests (servicios + DB)
3. E2E tests (flujos completos de usuario)
4. Contract tests (comunicación entre servicios)
5. Performance tests (carga, stress)
6. Security tests (penetration, vulnerabilidades)

**Restricciones:**
- Timeline de 4 meses para MVP
- Equipo de 2 QA engineers
- CI/CD debe ser rápido (<10 min)
- Stack multi-lenguaje (Node.js, Java, React)

## Decisión

Adoptaremos una **estrategia de testing multi-layer** basada en la pirámide de testing:

```
        ▲
       / \
      /E2E\     (10% - tests lentos, pocos)
     /─────\
    /Integ \   (30% - tests medios, algunos)
   /─────────\
  /   Unit    \ (60% - tests rápidos, muchos)
 /─────────────\
```

### 1. Unit Tests (60% de tests)

**Objetivo:** Testear lógica de negocio aislada, sin dependencias externas

#### Backend (Node.js) - Jest

**Configuración:**
```json
// jest.config.js
{
  "preset": "ts-jest",
  "testEnvironment": "node",
  "coverageDirectory": "coverage",
  "collectCoverageFrom": [
    "src/**/*.ts",
    "!src/**/*.spec.ts",
    "!src/main.ts"
  ],
  "coverageThreshold": {
    "global": {
      "branches": 80,
      "functions": 80,
      "lines": 80,
      "statements": 80
    }
  }
}
```

**Ejemplo test:**
```typescript
// auction.service.spec.ts
import { AuctionService } from './auction.service';
import { BidValidator } from './bid.validator';

describe('AuctionService', () => {
  let service: AuctionService;
  let mockRepository: jest.Mocked<AuctionRepository>;
  let mockBidValidator: jest.Mocked<BidValidator>;

  beforeEach(() => {
    mockRepository = {
      findById: jest.fn(),
      save: jest.fn(),
    } as any;

    mockBidValidator = {
      validate: jest.fn(),
    } as any;

    service = new AuctionService(mockRepository, mockBidValidator);
  });

  describe('placeBid', () => {
    it('should accept valid bid above current price', async () => {
      const auction = {
        id: 'auction-1',
        currentPrice: 1000,
        status: 'ACTIVE',
      };

      mockRepository.findById.mockResolvedValue(auction);
      mockBidValidator.validate.mockReturnValue({ valid: true });

      const result = await service.placeBid('auction-1', {
        userId: 'user-1',
        amount: 1100,
      });

      expect(result.status).toBe('CONFIRMED');
      expect(mockRepository.save).toHaveBeenCalled();
    });

    it('should reject bid below minimum increment', async () => {
      const auction = {
        id: 'auction-1',
        currentPrice: 1000,
        status: 'ACTIVE',
      };

      mockRepository.findById.mockResolvedValue(auction);
      mockBidValidator.validate.mockReturnValue({
        valid: false,
        error: 'BID_TOO_LOW',
      });

      await expect(
        service.placeBid('auction-1', { userId: 'user-1', amount: 1010 })
      ).rejects.toThrow('Bid must be at least 5% higher');
    });

    it('should reject bid on ended auction', async () => {
      const auction = {
        id: 'auction-1',
        currentPrice: 1000,
        status: 'ENDED',
      };

      mockRepository.findById.mockResolvedValue(auction);

      await expect(
        service.placeBid('auction-1', { userId: 'user-1', amount: 1100 })
      ).rejects.toThrow('Auction has ended');
    });
  });
});
```

#### Backend (Java) - JUnit 5 + Mockito

**Ejemplo test:**
```java
// PaymentServiceTest.java
@ExtendWith(MockitoExtension.class)
class PaymentServiceTest {

    @Mock
    private PaymentGateway paymentGateway;

    @Mock
    private PaymentRepository paymentRepository;

    @InjectMocks
    private PaymentService paymentService;

    @Test
    void shouldProcessPaymentSuccessfully() {
        // Given
        PaymentRequest request = PaymentRequest.builder()
            .auctionId("auction-1")
            .userId("user-1")
            .amount(1000)
            .method("CREDIT_CARD")
            .build();

        when(paymentGateway.charge(any())).thenReturn(
            ChargeResult.success("charge-123")
        );

        // When
        Payment result = paymentService.processPayment(request);

        // Then
        assertThat(result.getStatus()).isEqualTo(PaymentStatus.SUCCESS);
        assertThat(result.getChargeId()).isEqualTo("charge-123");

        verify(paymentRepository).save(any(Payment.class));
    }

    @Test
    void shouldHandlePaymentFailure() {
        // Given
        PaymentRequest request = PaymentRequest.builder()
            .auctionId("auction-1")
            .userId("user-1")
            .amount(1000)
            .method("CREDIT_CARD")
            .build();

        when(paymentGateway.charge(any())).thenReturn(
            ChargeResult.failure("INSUFFICIENT_FUNDS")
        );

        // When
        Payment result = paymentService.processPayment(request);

        // Then
        assertThat(result.getStatus()).isEqualTo(PaymentStatus.FAILED);
        assertThat(result.getErrorCode()).isEqualTo("INSUFFICIENT_FUNDS");
    }
}
```

#### Frontend (React) - Jest + React Testing Library

**Ejemplo test:**
```typescript
// BidForm.test.tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { BidForm } from './BidForm';

describe('BidForm', () => {
  it('should submit valid bid', async () => {
    const mockOnSubmit = jest.fn();

    render(
      <BidForm
        auctionId="auction-1"
        currentPrice={1000}
        onSubmit={mockOnSubmit}
      />
    );

    const input = screen.getByLabelText('Bid Amount');
    const button = screen.getByRole('button', { name: 'Place Bid' });

    fireEvent.change(input, { target: { value: '1100' } });
    fireEvent.click(button);

    await waitFor(() => {
      expect(mockOnSubmit).toHaveBeenCalledWith({
        amount: 1100,
      });
    });
  });

  it('should show error for bid below minimum', async () => {
    render(
      <BidForm
        auctionId="auction-1"
        currentPrice={1000}
        onSubmit={jest.fn()}
      />
    );

    const input = screen.getByLabelText('Bid Amount');
    const button = screen.getByRole('button', { name: 'Place Bid' });

    fireEvent.change(input, { target: { value: '1010' } }); // Too low (< 5%)
    fireEvent.click(button);

    await waitFor(() => {
      expect(screen.getByText(/minimum bid is/i)).toBeInTheDocument();
    });
  });
});
```

### 2. Integration Tests (30% de tests)

**Objetivo:** Testear integración con dependencias reales (DB, external services)

#### Backend Integration Tests - Testcontainers

**Node.js con DynamoDB Local:**
```typescript
// auction.integration.spec.ts
import { Test } from '@nestjs/testing';
import { DynamoDBDocumentClient } from '@aws-sdk/lib-dynamodb';
import { GenericContainer, StartedTestContainer } from 'testcontainers';

describe('Auction Service Integration', () => {
  let container: StartedTestContainer;
  let dynamoClient: DynamoDBDocumentClient;
  let auctionService: AuctionService;

  beforeAll(async () => {
    // Start DynamoDB Local container
    container = await new GenericContainer('amazon/dynamodb-local')
      .withExposedPorts(8000)
      .start();

    const port = container.getMappedPort(8000);

    // Initialize DynamoDB client
    dynamoClient = DynamoDBDocumentClient.from(
      new DynamoDBClient({
        endpoint: `http://localhost:${port}`,
        region: 'local',
      })
    );

    // Create test table
    await createTestTable(dynamoClient);

    // Initialize service with real DB
    const module = await Test.createTestingModule({
      providers: [
        AuctionService,
        { provide: 'DynamoDBClient', useValue: dynamoClient },
      ],
    }).compile();

    auctionService = module.get(AuctionService);
  });

  afterAll(async () => {
    await container.stop();
  });

  it('should persist auction to DynamoDB', async () => {
    const auction = await auctionService.create({
      title: 'Test Auction',
      startingPrice: 1000,
    });

    const retrieved = await auctionService.findById(auction.id);

    expect(retrieved).toMatchObject({
      title: 'Test Auction',
      startingPrice: 1000,
    });
  });
});
```

**Java con TestContainers:**
```java
@Testcontainers
@SpringBootTest
class PaymentServiceIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("test")
        .withUsername("test")
        .withPassword("test");

    @Autowired
    private PaymentService paymentService;

    @Test
    void shouldProcessPaymentEndToEnd() {
        // Real DB interaction
        PaymentRequest request = PaymentRequest.builder()
            .amount(1000)
            .build();

        Payment payment = paymentService.processPayment(request);

        assertThat(payment.getId()).isNotNull();

        // Verify persisted to DB
        Payment retrieved = paymentService.findById(payment.getId());
        assertThat(retrieved.getAmount()).isEqualTo(1000);
    }
}
```

### 3. Contract Tests (Communication entre servicios)

**Objetivo:** Verificar que contratos de API no se rompan entre servicios

#### Pact (Consumer-Driven Contract Testing)

**Consumer (API Gateway):**
```typescript
// auction-api.contract.spec.ts
import { pactWith } from 'jest-pact';

pactWith({ consumer: 'APIGateway', provider: 'AuctionService' }, (provider) => {
  describe('Get Auction', () => {
    beforeEach(() => {
      return provider.addInteraction({
        state: 'auction exists',
        uponReceiving: 'a request for auction',
        withRequest: {
          method: 'GET',
          path: '/auctions/auction-1',
        },
        willRespondWith: {
          status: 200,
          headers: { 'Content-Type': 'application/json' },
          body: {
            id: 'auction-1',
            title: 'Vintage Painting',
            currentPrice: 1000,
          },
        },
      });
    });

    it('returns auction details', async () => {
      const auction = await auctionClient.getAuction('auction-1');
      expect(auction.id).toBe('auction-1');
    });
  });
});
```

**Provider (Auction Service) verification:**
```typescript
// auction-service.contract.spec.ts
import { Verifier } from '@pact-foundation/pact';

describe('Auction Service Pact Verification', () => {
  it('should validate provider', () => {
    return new Verifier({
      provider: 'AuctionService',
      providerBaseUrl: 'http://localhost:3000',
      pactUrls: ['./pacts/apigateway-auctionservice.json'],
      stateHandlers: {
        'auction exists': () => {
          // Setup: Create auction in DB
          return auctionRepository.create({ id: 'auction-1', ... });
        },
      },
    }).verifyProvider();
  });
});
```

### 4. E2E Tests (10% de tests)

**Objetivo:** Testear flujos completos de usuario

#### Playwright (para React frontend)

```typescript
// e2e/auction-bidding.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Auction Bidding Flow', () => {
  test('should allow user to place bid and win auction', async ({ page }) => {
    // 1. Login
    await page.goto('https://localhost:3000/login');
    await page.fill('[name=email]', 'buyer@test.com');
    await page.fill('[name=password]', 'password123');
    await page.click('button[type=submit]');

    await expect(page).toHaveURL('/dashboard');

    // 2. Navigate to auction
    await page.goto('/auctions/auction-1');

    // 3. Place bid
    await page.fill('[name=bidAmount]', '1100');
    await page.click('button:has-text("Place Bid")');

    // 4. Verify bid success
    await expect(page.locator('.bid-success')).toBeVisible();
    await expect(page.locator('.current-price')).toHaveText('$1,100');

    // 5. Verify real-time update (simulate another bid)
    // (requires test helper to trigger bid from backend)

    // 6. Wait for auction to end (or simulate)

    // 7. Verify won auction
    await expect(page.locator('.auction-winner')).toHaveText('You won!');
  });

  test('should reject bid below minimum', async ({ page }) => {
    await page.goto('/auctions/auction-1');

    await page.fill('[name=bidAmount]', '1010'); // Too low
    await page.click('button:has-text("Place Bid")');

    await expect(page.locator('.error-message')).toContainText(
      'Minimum bid is $1,050'
    );
  });
});
```

### 5. Performance Tests

**Objetivo:** Verificar que sistema soporta 10K usuarios concurrentes

#### k6 (Load Testing)

```javascript
// load-test/auction-load.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 1000 },   // Ramp up to 1K users
    { duration: '5m', target: 1000 },   // Stay at 1K
    { duration: '2m', target: 5000 },   // Ramp to 5K
    { duration: '5m', target: 5000 },   // Stay at 5K
    { duration: '2m', target: 10000 },  // Ramp to 10K
    { duration: '5m', target: 10000 },  // Stay at 10K
    { duration: '2m', target: 0 },      // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<200', 'p(99)<500'], // 95% < 200ms, 99% < 500ms
    http_req_failed: ['rate<0.01'],                // Error rate < 1%
  },
};

export default function () {
  // Get auctions list
  let res = http.get('https://api.artchain.com/api/v1/auctions');
  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 200ms': (r) => r.timings.duration < 200,
  });

  sleep(1);

  // Get auction details
  res = http.get('https://api.artchain.com/api/v1/auctions/auction-1');
  check(res, {
    'status is 200': (r) => r.status === 200,
  });

  sleep(2);

  // Place bid
  const payload = JSON.stringify({
    amount: 1100,
  });

  res = http.post(
    'https://api.artchain.com/api/v1/auctions/auction-1/bids',
    payload,
    {
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${__ENV.AUTH_TOKEN}`,
      },
    }
  );

  check(res, {
    'bid accepted': (r) => r.status === 201,
    'bid latency < 2s': (r) => r.timings.duration < 2000,
  });

  sleep(5);
}
```

### 6. Security Tests

**Objetivo:** Identificar vulnerabilidades

#### OWASP ZAP (Automated Security Testing)

```yaml
# zap-scan.yaml
zap-scan:
  target: https://staging.artchain.com
  rules:
    - sql-injection
    - xss
    - csrf
    - insecure-auth
  alerts:
    fail-on: high
```

#### Snyk (Dependency Scanning)

```bash
# CI/CD pipeline
snyk test --severity-threshold=high
snyk monitor
```

### 7. Smart Contract Tests (Hardhat)

**Objetivo:** Testear smart contracts blockchain

```typescript
// test/ItemAuthenticity.test.ts
import { expect } from 'chai';
import { ethers } from 'hardhat';

describe('ItemAuthenticity', () => {
  let contract: ItemAuthenticity;
  let owner: SignerWithAddress;
  let user: SignerWithAddress;

  beforeEach(async () => {
    [owner, user] = await ethers.getSigners();

    const Factory = await ethers.getContractFactory('ItemAuthenticity');
    contract = await Factory.deploy();
  });

  it('should certify item', async () => {
    const tx = await contract.certifyItem(
      'item-1',
      'ipfs://Qm...',
      'Artist Name'
    );

    await tx.wait();

    const cert = await contract.getCertificate('item-1');
    expect(cert.itemId).to.equal('item-1');
    expect(cert.metadataHash).to.equal('ipfs://Qm...');
  });

  it('should prevent double certification', async () => {
    await contract.certifyItem('item-1', 'ipfs://Qm...', 'Artist');

    await expect(
      contract.certifyItem('item-1', 'ipfs://Qm2...', 'Artist')
    ).to.be.revertedWith('Item already certified');
  });

  it('should emit event on certification', async () => {
    await expect(contract.certifyItem('item-1', 'ipfs://Qm...', 'Artist'))
      .to.emit(contract, 'ItemCertified')
      .withArgs('item-1', owner.address);
  });
});
```

## CI/CD Pipeline

```yaml
# .github/workflows/ci.yml
name: CI/CD

on: [push, pull_request]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm ci
      - run: npm run test:unit
      - run: npm run test:coverage
      - uses: codecov/codecov-action@v3

  integration-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm ci
      - run: npm run test:integration

  e2e-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm ci
      - run: npx playwright install
      - run: npm run test:e2e

  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

  deploy:
    needs: [unit-tests, integration-tests, e2e-tests, security-scan]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploy to production"
```

## Consecuencias

### Positivas

1. **Confidence in releases**
   - 80%+ code coverage
   - Bug detection temprana
   - Menos hotfixes en producción

2. **Fast feedback**
   - Unit tests run en <2 min
   - Integration tests en <5 min
   - E2E tests en <10 min (parallelized)

3. **Regression prevention**
   - Tests automáticos previenen breaking changes
   - Contract tests protegen APIs

4. **Documentation**
   - Tests sirven como documentación viva
   - Ejemplos de uso de APIs

5. **Refactoring safety**
   - Podemos refactorizar con confianza
   - Tests catch regressions

### Negativas

1. **Time investment**
   - Escribir tests toma tiempo (30-50% extra)
   - Mantenimiento de tests

2. **Flaky tests**
   - E2E tests pueden ser flaky
   - Requiere retry logic y debugging

3. **Infrastructure**
   - Testcontainers, Playwright requieren recursos
   - CI/CD runners más costosos

4. **Learning curve**
   - Equipo debe aprender mejores prácticas
   - Mocking, fixtures, factories

## Alternativas Consideradas

- **Cypress vs Playwright:** Playwright elegido por mejor performance y multi-browser
- **Jest vs Mocha:** Jest elegido por zero-config y built-in coverage
- **JUnit 4 vs JUnit 5:** JUnit 5 elegido por features modernas

## Referencias

- Testing Pyramid: Martin Fowler
- React Testing Library: https://testing-library.com/
- Playwright: https://playwright.dev/
- k6: https://k6.io/

## Historial

- 2025-11-07: Decisión inicial (Aceptado)
