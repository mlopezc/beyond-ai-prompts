# ADR-005: OAuth 2.0 + JWT para Autenticación y RBAC para Autorización

## Estado
Aceptado

## Contexto

La plataforma ArtChain necesita un sistema de autenticación y autorización robusto que soporte:
- Múltiples métodos de autenticación (email/password, social login)
- 4 roles de usuario: Comprador, Vendedor, Admin, Soporte
- Permisos granulares por acción
- Sesiones seguras
- MFA para transacciones de alto valor (>$10,000)
- Compliance con GDPR y regulaciones de privacidad

**Requisitos del PRD relacionados:**
- RF-001: Registro de usuarios con verificación de email
- RF-002: Sistema de roles y permisos (RBAC)
- Seguridad: Password hashing bcrypt, sesiones seguras, MFA
- Performance: Latencia <200ms (autenticación no debe degradar)

**Matriz de Permisos (del PRD):**
| Acción | Comprador | Vendedor | Admin | Soporte |
|--------|-----------|----------|-------|---------|
| Crear listado | ❌ | ✅ | ✅ | ❌ |
| Realizar puja | ✅ | ❌ | ✅ | ❌ |
| Aprobar listado | ❌ | ❌ | ✅ | ❌ |
| Ver datos sensibles | Propios | Propios | Todos | Limitado |
| Gestionar usuarios | ❌ | ❌ | ✅ | ❌ |

**Restricciones:**
- Stateless architecture (microservicios)
- Multi-región (sesiones deben funcionar cross-region)
- Performance (no queremos DB hit en cada request)

## Decisión

Implementaremos **OAuth 2.0 con JWT (JSON Web Tokens)** para autenticación y **RBAC (Role-Based Access Control)** para autorización.

### Arquitectura de Autenticación

#### 1. Flujo de Registro y Login

**Registro:**
```
1. Usuario POST /api/auth/register
   {
     email, password, firstName, lastName
   }

2. User Service:
   - Valida email único
   - Hash password con bcrypt (cost factor 12)
   - Crea usuario en DynamoDB
   - Genera token de verificación (JWT corto, 24h)
   - Envía email de verificación

3. Usuario click en link de verificación
   GET /api/auth/verify-email?token={verificationToken}

4. User Service:
   - Valida token
   - Marca email como verificado
   - Usuario puede hacer login
```

**Login:**
```
1. Usuario POST /api/auth/login
   { email, password }

2. User Service:
   - Busca usuario por email (GSI en DynamoDB)
   - Verifica password con bcrypt.compare()
   - Genera tokens:
     * Access Token (JWT, 15 min expiry)
     * Refresh Token (JWT, 7 días, almacenado en Redis)

3. Response:
   {
     accessToken: "eyJhbGc...",
     refreshToken: "eyJhbGc...",
     user: { id, email, role, ... }
   }

4. Cliente almacena:
   - accessToken en memory (NO localStorage por seguridad)
   - refreshToken en httpOnly cookie (más seguro)
```

#### 2. JWT Structure

**Access Token (15 min TTL):**
```json
{
  "header": {
    "alg": "RS256",
    "typ": "JWT"
  },
  "payload": {
    "sub": "user123",
    "email": "user@example.com",
    "role": "BUYER",
    "permissions": ["bid:create", "auction:read", "profile:update"],
    "iat": 1699372800,
    "exp": 1699373700,
    "iss": "artchain-auth-service",
    "aud": "artchain-api"
  },
  "signature": "..."
}
```

**Refresh Token (7 días TTL):**
```json
{
  "payload": {
    "sub": "user123",
    "tokenId": "unique-token-id",
    "type": "refresh",
    "iat": 1699372800,
    "exp": 1699977600
  }
}
```

**Por qué RS256 (no HS256):**
- Asymmetric keys (public/private)
- Servicios pueden verificar token sin secret compartido
- Más seguro en arquitectura distribuida

#### 3. OAuth 2.0 Social Login

**Proveedores:** Google, Apple (Fase 1)

**Flujo (OAuth 2.0 Authorization Code Flow):**
```
1. Usuario click "Login with Google"

2. Frontend redirect a Google OAuth:
   https://accounts.google.com/o/oauth2/v2/auth?
     client_id={CLIENT_ID}&
     redirect_uri={CALLBACK_URL}&
     response_type=code&
     scope=openid email profile

3. Usuario autentica en Google

4. Google redirect a callback:
   {CALLBACK_URL}?code={AUTHORIZATION_CODE}

5. Frontend envía code a backend:
   POST /api/auth/social/google
   { code }

6. User Service:
   - Exchange code por tokens con Google
   - Obtiene perfil de usuario (email, name)
   - Busca usuario por email
   - Si existe: login
   - Si no existe: crea usuario
   - Genera Access + Refresh tokens propios

7. Response: { accessToken, refreshToken, user }
```

#### 4. Token Refresh Flow

```
1. Access Token expira (15 min)

2. Frontend detecta 401 Unauthorized

3. POST /api/auth/refresh
   Cookie: refreshToken={REFRESH_TOKEN}

4. User Service:
   - Verifica Refresh Token signature
   - Verifica no está en blacklist (Redis)
   - Verifica no expiró
   - Genera nuevo Access Token
   - (Opcional) Rota Refresh Token

5. Response: { accessToken }

6. Frontend reintenta request original con nuevo token
```

#### 5. Session Management (Redis)

**Redis Structure:**
```
Key: session:{userId}:{tokenId}
Value: {
  userId,
  role,
  deviceInfo,
  ipAddress,
  createdAt,
  lastActivity
}
TTL: 7 días (sincronizado con Refresh Token)

Key: blacklist:token:{tokenId}
Value: true
TTL: tiempo restante hasta expiración del token
```

**Casos de uso:**
- **Revocación de tokens:** Agregar a blacklist
- **Logout:** Eliminar sesión de Redis
- **Logout all devices:** Eliminar todas las sesiones del usuario
- **Rate limiting:** Contar requests por usuario

### RBAC (Role-Based Access Control)

#### Roles y Permisos

**Roles:**
```typescript
enum Role {
  BUYER = 'BUYER',
  SELLER = 'SELLER',
  ADMIN = 'ADMIN',
  SUPPORT = 'SUPPORT'
}

const ROLE_PERMISSIONS: Record<Role, string[]> = {
  BUYER: [
    'auction:read',
    'auction:search',
    'bid:create',
    'bid:read:own',
    'profile:read:own',
    'profile:update:own',
    'payment:create:own',
  ],

  SELLER: [
    'auction:read',
    'auction:create',
    'auction:update:own',
    'auction:delete:own',
    'bid:read:own-auctions',
    'profile:read:own',
    'profile:update:own',
    'analytics:read:own',
  ],

  ADMIN: [
    '*:*', // All permissions
  ],

  SUPPORT: [
    'auction:read',
    'user:read',
    'bid:read',
    'transaction:read',
    'ticket:create',
    'ticket:update',
    // NO: user:update, transaction:update, bid:create
  ],
};
```

#### Middleware de Autorización

**Express/NestJS Middleware:**
```typescript
// Guard para verificar autenticación
@Injectable()
export class JwtAuthGuard implements CanActivate {
  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const token = this.extractToken(request);

    if (!token) {
      throw new UnauthorizedException('No token provided');
    }

    try {
      // Verify JWT signature
      const payload = await this.jwtService.verify(token);

      // Check blacklist (Redis)
      const isBlacklisted = await this.redis.get(`blacklist:token:${payload.jti}`);
      if (isBlacklisted) {
        throw new UnauthorizedException('Token revoked');
      }

      // Attach user to request
      request.user = payload;
      return true;

    } catch (error) {
      throw new UnauthorizedException('Invalid token');
    }
  }
}

// Guard para verificar permisos
@Injectable()
export class PermissionsGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredPermissions = this.reflector.get<string[]>(
      'permissions',
      context.getHandler()
    );

    if (!requiredPermissions) {
      return true; // No permissions required
    }

    const request = context.switchToHttp().getRequest();
    const user = request.user;

    return this.hasPermissions(user.permissions, requiredPermissions);
  }

  private hasPermissions(userPerms: string[], required: string[]): boolean {
    if (userPerms.includes('*:*')) return true; // Admin

    return required.every(perm => {
      return userPerms.some(userPerm => this.matchPermission(userPerm, perm));
    });
  }

  private matchPermission(userPerm: string, required: string): boolean {
    const [userResource, userAction] = userPerm.split(':');
    const [reqResource, reqAction] = required.split(':');

    if (userResource === '*' || userResource === reqResource) {
      if (userAction === '*' || userAction === reqAction) {
        return true;
      }
    }
    return false;
  }
}

// Uso en controllers
@Controller('auctions')
export class AuctionsController {
  @Post()
  @UseGuards(JwtAuthGuard, PermissionsGuard)
  @RequirePermissions('auction:create')
  async createAuction(@Body() dto: CreateAuctionDto, @User() user) {
    // Solo usuarios con permiso "auction:create" pueden llegar aquí
    // (SELLER, ADMIN)
  }

  @Get(':id')
  @UseGuards(JwtAuthGuard)
  async getAuction(@Param('id') id: string) {
    // Cualquier usuario autenticado puede leer
  }
}
```

### MFA (Multi-Factor Authentication)

**Para transacciones >$10,000:**
```typescript
@Post('payment/process')
@UseGuards(JwtAuthGuard, MfaGuard)
async processPayment(@Body() dto: PaymentDto, @User() user) {
  // MfaGuard verifica que usuario haya pasado MFA recientemente
  // Si no, devuelve 403 con instrucción de completar MFA
}

// Flujo MFA
1. Usuario intenta transacción >$10K
2. Backend detecta y requiere MFA
3. Envía código 6 dígitos por SMS/Email
4. Usuario ingresa código
5. POST /api/auth/mfa/verify { code }
6. Backend verifica código
7. Marca sesión como "MFA verified" por 10 minutos (Redis)
8. Usuario puede proceder con transacción
```

### KYC para Vendedores

**Flujo de verificación:**
```
1. Usuario se registra como SELLER

2. Role inicialmente = SELLER_UNVERIFIED
   Permissions = [ 'auction:read' ] // Muy limitado

3. Usuario completa KYC:
   - Upload de ID
   - Prueba de dirección
   - (Integración con servicio KYC: Jumio, Onfido)

4. Admin o sistema automatizado aprueba KYC

5. Role cambia a SELLER
   Permissions = full SELLER permissions

6. Usuario puede listar artículos
```

## Consecuencias

### Positivas

1. **Stateless Architecture**
   - JWT permite verificación sin DB hit
   - Escalabilidad horizontal fácil
   - Multi-región sin sincronización compleja

2. **Security**
   - Tokens firmados (no pueden ser alterados)
   - Short-lived access tokens (15 min) reduce ventana de ataque
   - Refresh tokens en httpOnly cookies (no accesibles por JS → XSS protection)
   - Password hashing con bcrypt (cost 12)

3. **Performance**
   - Verificación de JWT es rápida (no DB)
   - Redis para sesiones (sub-millisecond latency)
   - Permisos incluidos en token (no query en cada request)

4. **User Experience**
   - Social login (menos fricción)
   - Refresh automático (usuario no ve expiración)
   - MFA solo cuando necesario

5. **Granularidad de permisos**
   - RBAC con permisos específicos
   - Fácil agregar nuevos permisos
   - Ownership checks (`:own` suffix)

6. **Compliance**
   - GDPR: Revocación de tokens, logout
   - Audit trail: Sesiones en Redis con metadata

7. **Revocación**
   - Logout funciona (blacklist en Redis)
   - Logout all devices funciona
   - Admin puede revocar tokens de usuario

### Negativas

1. **Token size**
   - JWT puede ser grande (200-500 bytes)
   - Enviado en cada request (overhead de bandwidth)
   - Mitigado con compression y permisos limitados

2. **No revocación instantánea**
   - Access token válido hasta expiración (15 min)
   - Blacklist mitiga pero no previene 100%
   - Trade-off: TTL corto vs latencia de verificación

3. **Key management**
   - Private key debe estar segura (AWS Secrets Manager)
   - Key rotation necesaria (complejidad)
   - Old public keys deben mantenerse para verificar tokens viejos

4. **Complejidad de refresh**
   - Frontend debe manejar refresh automático
   - Race conditions posibles (múltiples requests simultáneos)

5. **Redis dependency**
   - Sesiones dependen de Redis
   - Si Redis cae, logout no funciona (pero auth sí con JWT puro)
   - Mitigado con Redis cluster (HA)

6. **MFA friction**
   - UX potencialmente molesto para usuarios
   - Balance entre seguridad y experiencia

### Neutrales

1. **Token rotation**
   - Refresh token rotation aumenta seguridad
   - Pero agrega complejidad (old tokens deben invalidarse)

2. **Session storage**
   - Redis vs DynamoDB
   - Redis preferido por performance
   - DynamoDB como backup si Redis cae

## Alternativas Consideradas

### Alternativa 1: Session-based Authentication (cookies)
**Descripción:** Sesiones tradicionales con cookies y session store

**Ventajas:**
- Más simple de implementar
- Revocación instantánea
- Tokens más pequeños (solo session ID)

**Razones para rechazar:**
- **Stateful:** Requiere DB/Redis hit en cada request (peor performance)
- **Multi-región:** Sesiones deben replicarse (complejo)
- **Microservicios:** Cada servicio necesita acceso a session store
- **CORS:** Más complejo con cookies cross-origin

### Alternativa 2: API Keys
**Descripción:** API keys estáticas para autenticación

**Ventajas:**
- Muy simple
- No expiración

**Razones para rechazar:**
- **No para usuarios humanos:** Solo para machine-to-machine
- **No revocación fácil:** Requiere cambiar key en todos los clientes
- **No permisos granulares:** Una key = todos los permisos
- **Uso:** Solo para API pública (Fase 3)

### Alternativa 3: OAuth 2.0 con Authorization Server externo (Auth0, Okta)
**Ventajas:**
- Managed service (menos ops overhead)
- MFA incluido
- Social login incluido
- Compliance incluido

**Razones para rechazar:**
- **Costo:** $100-$500/mes (más costoso que self-hosted)
- **Vendor lock-in:** Difícil migrar después
- **Customización limitada:** No podemos modificar flows
- **Latency:** Request adicional a servicio externo
- **Decisión:** Preferimos control y flexibilidad para MVP

### Alternativa 4: Paseto (Platform-Agnostic Security Tokens)
**Descripción:** Alternativa a JWT, más seguro por diseño

**Ventajas:**
- Más seguro (menos opciones de algoritmo = menos errores)
- No necesita especificar algoritmo en token

**Razones para rechazar:**
- **Menos maduro:** Ecosistema menor que JWT
- **Menos libraries:** Especialmente para Java
- **Equipo no familiar:** Learning curve
- **JWT suficiente:** Con buenas prácticas (RS256, no HS256)

### Alternativa 5: Magic Links (passwordless)
**Descripción:** Login sin password, solo email con link

**Ventajas:**
- UX simple
- No passwords que robar
- No necesita almacenar passwords

**Razones para rechazar:**
- **Email dependency:** Si email cae, no puedes login
- **UX:** Usuarios deben check email cada vez
- **No offline:** Necesita connectivity
- **Adopción:** Usuarios no familiarizados
- **Decisión:** Quizás agregar en Fase 3 como opción

## Referencias

- PRD RF-001, RF-002
- OAuth 2.0 RFC 6749: https://tools.ietf.org/html/rfc6749
- JWT Best Practices: https://tools.ietf.org/html/rfc8725
- OWASP Authentication Cheat Sheet

## Notas

### Key Rotation Strategy

```
1. Generar nuevo key pair (RSA 2048 bits)
2. Añadir public key a JWKS endpoint con kid (key ID) único
3. User Service empieza a firmar con new private key
4. Old public keys permanecen en JWKS (para verificar tokens existentes)
5. Después de 7 días (max refresh token TTL), remover old public key

Frecuencia: Cada 90 días
```

### JWKS Endpoint

```
GET /api/auth/.well-known/jwks.json

Response:
{
  "keys": [
    {
      "kty": "RSA",
      "kid": "key-2025-11",
      "use": "sig",
      "alg": "RS256",
      "n": "...", // Public key modulus
      "e": "AQAB" // Public key exponent
    },
    {
      "kty": "RSA",
      "kid": "key-2025-08", // Old key (still valid)
      "use": "sig",
      "alg": "RS256",
      "n": "...",
      "e": "AQAB"
    }
  ]
}
```

### Frontend Token Management

```typescript
// Token interceptor (Axios)
axios.interceptors.response.use(
  response => response,
  async error => {
    const originalRequest = error.config;

    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;

      try {
        // Refresh token (cookie enviado automáticamente)
        const { data } = await axios.post('/api/auth/refresh');

        // Actualizar access token en memory
        setAccessToken(data.accessToken);

        // Reintentar request original
        originalRequest.headers.Authorization = `Bearer ${data.accessToken}`;
        return axios(originalRequest);

      } catch (refreshError) {
        // Refresh failed, redirect to login
        redirectToLogin();
      }
    }

    return Promise.reject(error);
  }
);
```

## Historial

- 2025-11-07: Decisión inicial (Aceptado)
