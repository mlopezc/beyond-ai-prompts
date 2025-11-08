# ADR-008: Redux Toolkit para State Management en Frontend

## Estado
Aceptado

## Contexto

La aplicación frontend de ArtChain necesita manejar estado complejo:
- **Autenticación:** Usuario actual, tokens, permisos
- **Subastas:** Lista de subastas, detalles, filtros, paginación
- **Pujas en tiempo real:** Actualizaciones WebSocket, optimistic updates
- **UI State:** Modals, notificaciones, loading states
- **Cache:** Datos fetched de APIs

**Requisitos del PRD relacionados:**
- Performance: Time to Interactive <5 segundos, Lighthouse >90
- Real-time: Actualizaciones de pujas <1 segundo
- UX: Optimistic updates, no flickering
- Múltiples páginas con estado compartido

**Características necesarias:**
- **Global state:** Accesible desde cualquier componente
- **Predictability:** Estado predecible y debuggable
- **Performance:** No re-renders innecesarios
- **DevTools:** Debugging con time-travel
- **TypeScript:** Type safety
- **Server state:** Caching, invalidation, refetching

**Restricciones:**
- React 18+
- TypeScript
- Equipo de 2 frontend developers (familiaridad con Redux)

## Decisión

Adoptaremos **Redux Toolkit + React Query (TanStack Query)** como solución híbrida:
- **Redux Toolkit:** Global UI state y autenticación
- **React Query:** Server state (API data fetching y caching)

### Arquitectura de State Management

```
┌─────────────────────────────────────────────────┐
│              React Components                    │
└───┬───────────────────────────────┬──────────────┘
    │                               │
    │ UI State                      │ Server State
    │ (auth, modals, etc)           │ (auctions, bids)
    ▼                               ▼
┌──────────────┐              ┌──────────────┐
│ Redux Toolkit│              │ React Query  │
│   Store      │              │   Cache      │
└──────────────┘              └──────────────┘
```

### 1. Redux Toolkit Setup

**Store configuration:**
```typescript
// src/store/index.ts
import { configureStore } from '@reduxjs/toolkit';
import authReducer from './slices/authSlice';
import uiReducer from './slices/uiSlice';

export const store = configureStore({
  reducer: {
    auth: authReducer,
    ui: uiReducer,
  },
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware({
      serializableCheck: false, // For WebSocket objects
    }),
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

**Auth Slice (ejemplo):**
```typescript
// src/store/slices/authSlice.ts
import { createSlice, PayloadAction } from '@reduxjs/toolkit';

interface User {
  id: string;
  email: string;
  role: 'BUYER' | 'SELLER' | 'ADMIN' | 'SUPPORT';
  firstName: string;
  lastName: string;
}

interface AuthState {
  user: User | null;
  accessToken: string | null;
  isAuthenticated: boolean;
  isLoading: boolean;
}

const initialState: AuthState = {
  user: null,
  accessToken: null,
  isAuthenticated: false,
  isLoading: true,
};

const authSlice = createSlice({
  name: 'auth',
  initialState,
  reducers: {
    setUser: (state, action: PayloadAction<User>) => {
      state.user = action.payload;
      state.isAuthenticated = true;
      state.isLoading = false;
    },
    setAccessToken: (state, action: PayloadAction<string>) => {
      state.accessToken = action.payload;
    },
    logout: (state) => {
      state.user = null;
      state.accessToken = null;
      state.isAuthenticated = false;
    },
    setLoading: (state, action: PayloadAction<boolean>) => {
      state.isLoading = action.payload;
    },
  },
});

export const { setUser, setAccessToken, logout, setLoading } = authSlice.actions;
export default authSlice.reducer;
```

**UI Slice (modals, notifications):**
```typescript
// src/store/slices/uiSlice.ts
import { createSlice, PayloadAction } from '@reduxjs/toolkit';

interface Notification {
  id: string;
  type: 'success' | 'error' | 'info' | 'warning';
  message: string;
}

interface UIState {
  modals: {
    bidModal: { isOpen: boolean; auctionId: string | null };
    loginModal: { isOpen: boolean };
  };
  notifications: Notification[];
}

const initialState: UIState = {
  modals: {
    bidModal: { isOpen: false, auctionId: null },
    loginModal: { isOpen: false },
  },
  notifications: [],
};

const uiSlice = createSlice({
  name: 'ui',
  initialState,
  reducers: {
    openBidModal: (state, action: PayloadAction<string>) => {
      state.modals.bidModal = { isOpen: true, auctionId: action.payload };
    },
    closeBidModal: (state) => {
      state.modals.bidModal = { isOpen: false, auctionId: null };
    },
    addNotification: (state, action: PayloadAction<Omit<Notification, 'id'>>) => {
      state.notifications.push({
        id: Date.now().toString(),
        ...action.payload,
      });
    },
    removeNotification: (state, action: PayloadAction<string>) => {
      state.notifications = state.notifications.filter(
        (n) => n.id !== action.payload
      );
    },
  },
});

export const {
  openBidModal,
  closeBidModal,
  addNotification,
  removeNotification,
} = uiSlice.actions;

export default uiSlice.reducer;
```

**Typed hooks:**
```typescript
// src/store/hooks.ts
import { TypedUseSelectorHook, useDispatch, useSelector } from 'react-redux';
import type { RootState, AppDispatch } from './index';

export const useAppDispatch = () => useDispatch<AppDispatch>();
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector;
```

### 2. React Query Setup

**Query client configuration:**
```typescript
// src/lib/queryClient.ts
import { QueryClient } from '@tanstack/react-query';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5, // 5 minutes
      cacheTime: 1000 * 60 * 30, // 30 minutes
      retry: 3,
      refetchOnWindowFocus: false,
    },
  },
});
```

**API hooks:**
```typescript
// src/hooks/api/useAuctions.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { auctionsApi } from '@/api/auctions';

export function useAuctions(filters?: AuctionFilters) {
  return useQuery({
    queryKey: ['auctions', filters],
    queryFn: () => auctionsApi.getAll(filters),
    staleTime: 1000 * 60, // 1 minute (subastas cambian frecuentemente)
  });
}

export function useAuction(id: string) {
  return useQuery({
    queryKey: ['auctions', id],
    queryFn: () => auctionsApi.getById(id),
    staleTime: 1000 * 30, // 30 seconds
    enabled: !!id, // Solo fetch si id existe
  });
}

export function useCreateAuction() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: auctionsApi.create,
    onSuccess: () => {
      // Invalidate auctions list
      queryClient.invalidateQueries({ queryKey: ['auctions'] });
    },
  });
}

export function usePlaceBid() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ auctionId, amount }: { auctionId: string; amount: number }) =>
      bidsApi.create(auctionId, { amount }),

    // Optimistic update
    onMutate: async ({ auctionId, amount }) => {
      // Cancel ongoing queries
      await queryClient.cancelQueries({ queryKey: ['auctions', auctionId] });

      // Snapshot previous value
      const previousAuction = queryClient.getQueryData(['auctions', auctionId]);

      // Optimistically update
      queryClient.setQueryData(['auctions', auctionId], (old: any) => ({
        ...old,
        currentPrice: amount,
        bidCount: old.bidCount + 1,
      }));

      return { previousAuction };
    },

    onError: (err, variables, context) => {
      // Rollback on error
      if (context?.previousAuction) {
        queryClient.setQueryData(
          ['auctions', variables.auctionId],
          context.previousAuction
        );
      }
    },

    onSettled: (data, error, variables) => {
      // Refetch to get accurate data
      queryClient.invalidateQueries({ queryKey: ['auctions', variables.auctionId] });
    },
  });
}
```

### 3. WebSocket Integration con Redux

**WebSocket middleware:**
```typescript
// src/store/middleware/websocketMiddleware.ts
import { Middleware } from '@reduxjs/toolkit';
import { io, Socket } from 'socket.io-client';

export const websocketMiddleware: Middleware = (store) => {
  let socket: Socket | null = null;

  return (next) => (action) => {
    switch (action.type) {
      case 'auth/setUser':
        // Connect WebSocket when user logs in
        const token = store.getState().auth.accessToken;
        socket = io('wss://api.artchain.com', {
          auth: { token },
        });

        socket.on('bid:new', (bid) => {
          // Update React Query cache
          const queryClient = getQueryClient();
          queryClient.setQueryData(['auctions', bid.auctionId], (old: any) => ({
            ...old,
            currentPrice: bid.amount,
            bidCount: old.bidCount + 1,
          }));

          // Show notification
          store.dispatch(
            addNotification({
              type: 'info',
              message: `New bid: $${bid.amount}`,
            })
          );
        });

        break;

      case 'auth/logout':
        // Disconnect WebSocket on logout
        socket?.disconnect();
        socket = null;
        break;

      case 'websocket/subscribe':
        socket?.emit('subscribe:auction', { auctionId: action.payload });
        break;

      case 'websocket/unsubscribe':
        socket?.emit('unsubscribe:auction', { auctionId: action.payload });
        break;
    }

    return next(action);
  };
};
```

### 4. Usage in Components

**Example component:**
```typescript
// src/pages/AuctionDetails.tsx
import { useParams } from 'react-router-dom';
import { useAuction, usePlaceBid } from '@/hooks/api/useAuctions';
import { useAppSelector, useAppDispatch } from '@/store/hooks';
import { openBidModal } from '@/store/slices/uiSlice';

export function AuctionDetails() {
  const { id } = useParams<{ id: string }>();
  const dispatch = useAppDispatch();

  // React Query (server state)
  const { data: auction, isLoading, error } = useAuction(id!);
  const placeBidMutation = usePlaceBid();

  // Redux (UI state)
  const { isOpen, auctionId } = useAppSelector((state) => state.ui.modals.bidModal);
  const { user } = useAppSelector((state) => state.auth);

  const handleBidClick = () => {
    dispatch(openBidModal(id!));
  };

  const handleBidSubmit = (amount: number) => {
    placeBidMutation.mutate(
      { auctionId: id!, amount },
      {
        onSuccess: () => {
          dispatch(closeBidModal());
          dispatch(addNotification({
            type: 'success',
            message: 'Bid placed successfully!',
          }));
        },
        onError: (error) => {
          dispatch(addNotification({
            type: 'error',
            message: error.message,
          }));
        },
      }
    );
  };

  if (isLoading) return <Spinner />;
  if (error) return <ErrorPage error={error} />;

  return (
    <div>
      <h1>{auction.title}</h1>
      <p>Current Price: ${auction.currentPrice}</p>
      <p>Bids: {auction.bidCount}</p>

      {user?.role === 'BUYER' && (
        <button onClick={handleBidClick}>Place Bid</button>
      )}

      {isOpen && auctionId === id && (
        <BidModal
          auction={auction}
          onSubmit={handleBidSubmit}
          isSubmitting={placeBidMutation.isLoading}
        />
      )}
    </div>
  );
}
```

### 5. Persistence

**Redux Persist (optional para auth):**
```typescript
import { persistStore, persistReducer } from 'redux-persist';
import storage from 'redux-persist/lib/storage'; // localStorage

const persistConfig = {
  key: 'root',
  storage,
  whitelist: ['auth'], // Solo persist auth state
};

const persistedReducer = persistReducer(persistConfig, rootReducer);

export const store = configureStore({
  reducer: persistedReducer,
});

export const persistor = persistStore(store);
```

## Consecuencias

### Positivas

1. **Clear separation of concerns**
   - Redux: UI state, auth (client-owned)
   - React Query: Server state (server-owned)
   - Evita duplicación y sincronización compleja

2. **Performance**
   - React Query: Automatic caching, deduplication, background refetch
   - Redux Toolkit: Immer for immutable updates (performance)
   - Optimistic updates para UX instantánea

3. **Developer Experience**
   - Redux DevTools: Time-travel debugging
   - React Query DevTools: Inspect cache
   - TypeScript: Type safety end-to-end
   - Less boilerplate vs vanilla Redux

4. **Predictability**
   - Redux: Single source of truth
   - Actions → Reducers → State (unidirectional)
   - Fácil rastrear cambios de estado

5. **Real-time updates**
   - WebSocket integration con Redux middleware
   - Actualización automática de cache React Query

6. **Testing**
   - Redux: Easy to test reducers (pure functions)
   - React Query: Mock queries con queryClient

### Negativas

1. **Complejidad inicial**
   - Curva de aprendizaje (Redux concepts)
   - Setup boilerplate (aunque RTK reduce)

2. **Over-engineering para apps simples**
   - Si la app fuera simple, Context API sería suficiente
   - Pero para ArtChain (complejo), justificado

3. **Bundle size**
   - Redux Toolkit + React Query + React: ~50KB gzipped
   - Pero performance benefits justify it

4. **State duplication risk**
   - Riesgo de poner server state en Redux (anti-pattern)
   - Mitigado con guidelines claros

### Neutrales

1. **Redux vs React Query boundaries**
   - Documentación clara de qué va donde
   - Code reviews para enforce

2. **Migration path**
   - Si React Query no existiera, podríamos usar Redux para todo
   - Pero sería más complejo (caching manual)

## Alternativas Consideradas

### Alternativa 1: Solo Context API
**Ventajas:**
- Nativo a React
- Zero dependencies
- Más simple

**Razones para rechazar:**
- **Performance:** Re-renders excesivos
- **DevTools:** No time-travel debugging
- **Scalability:** Difícil manejar estado complejo
- **Server state:** No caching automático

### Alternativa 2: Zustand
**Ventajas:**
- Más simple que Redux
- Menos boilerplate
- Good TypeScript support

**Razones para rechazar:**
- **Ecosystem:** Menor que Redux
- **DevTools:** Menos maduro
- **Team expertise:** Equipo conoce Redux
- **Empresa:** Redux más "enterprise-proven"

### Alternativa 3: MobX
**Ventajas:**
- Reactive (menos boilerplate)
- Performance excelente

**Razones para rechazar:**
- **Learning curve:** Más difícil que Redux para nuevos
- **Magic:** Menos explícito que Redux
- **Debugging:** Más difícil rastrear cambios

### Alternativa 4: Recoil
**Ventajas:**
- Atómico (granular subscriptions)
- Made by Facebook

**Razones para rechazar:**
- **Experimental:** Aún no 1.0
- **Ecosystem:** Pequeño
- **Uncertainty:** Futuro incierto

### Alternativa 5: Solo Redux (sin React Query)
**Ventajas:**
- Una sola solución
- Consistente

**Razones para rechazar:**
- **Caching manual:** React Query lo hace mejor
- **Boilerplate:** Más código para async logic
- **Server state:** Redux no diseñado para ello

### Alternativa 6: SWR (en lugar de React Query)
**Ventajas:**
- Más simple que React Query
- Más pequeño

**Razones para rechazar:**
- **Features:** React Query más completo (mutations, devtools)
- **Ecosystem:** React Query más grande
- **Performance:** Comparable

## Referencias

- Redux Toolkit: https://redux-toolkit.js.org/
- React Query: https://tanstack.com/query/latest
- Redux Style Guide: https://redux.js.org/style-guide/

## Notas

### State Organization

```
Redux (UI State):
├── auth (user, token, isAuthenticated)
├── ui
│   ├── modals (bidModal, loginModal, etc.)
│   ├── notifications
│   └── theme (dark/light mode)
└── websocket (connection status)

React Query (Server State):
├── auctions (list, details, cache)
├── bids (list, cache)
├── users (profile, cache)
└── payments (transactions, cache)
```

### Performance Optimization

**Memoization:**
```typescript
import { useSelector } from 'react-redux';
import { createSelector } from '@reduxjs/toolkit';

// Selector memoizado
const selectUserRole = createSelector(
  [(state: RootState) => state.auth.user],
  (user) => user?.role
);

// Uso
const role = useAppSelector(selectUserRole);
```

**Component-level optimization:**
```typescript
import { memo } from 'react';

export const AuctionCard = memo(({ auction }: Props) => {
  // Solo re-render si auction cambia
  return <div>...</div>;
});
```

## Historial

- 2025-11-07: Decisión inicial (Aceptado)
