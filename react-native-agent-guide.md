# AI Agent Guide for React Native App Development

This document defines the rules, project structure, and standards that any AI agent (Claude, Cursor, Copilot, etc.) must follow when working on this React Native project.

---

## 1. General Principles (Best Practices)

- **TypeScript** is mandatory; using `any` is forbidden unless explicitly justified.
- Components must be **Functional Components** using Hooks; Class Components are not allowed.
- Each file should have a single responsibility (Single Responsibility Principle).
- Business logic must be separated from UI; components should only handle rendering and user interaction.
- Avoid deep **prop drilling**; use Context or a state management solution when needed.
- Naming conventions:
  - Components: `PascalCase` (e.g. `UserCard.tsx`)
  - Hooks: `camelCase` prefixed with `use` (e.g. `useUserProfile.ts`)
  - Type files: `PascalCase` or `camelCase.types.ts`
  - Constants: `UPPER_SNAKE_CASE`
- Every component/hook/service should be testable and decoupled from other layers (use Dependency Injection where relevant).
- Use `useMemo`, `useCallback`, and `React.memo` for render optimization where appropriate — not excessively.
- Error handling should happen in the service/query layer, not inside components.
- Use **environment variables** for secrets and values that differ between environments.
- Follow accessibility (a11y) guidelines: `accessibilityLabel`, `accessibilityRole`, etc.
- Code must be lint-clean (ESLint + Prettier) and checked before every commit.
- Each component must have **exactly one type of responsibility**: either pure **UI** (presentational) or **logic** (container/orchestration) — never both:
  - **Presentational components** (UI): receive data and callbacks via props, render markup/styles only, contain no business logic, no data fetching, no direct hook calls to `queries`/`services`.
  - **Container components** (Logic): consume `hooks`, handle state/orchestration, and pass the results down as props to presentational components.
  - If a component starts mixing rendering with business rules, split it into a container + a presentational component.
- Follow **Clean Code** principles: meaningful names, small functions/components (prefer under ~150–200 lines), avoid deep nesting, no magic numbers/strings (use `consts`), no dead code, self-documenting code over excessive comments.
- Follow **Clean Architecture** principles across the layers defined in this guide:
  - **Dependency Rule**: dependencies only point inward/downward (`components → hooks → queries → services → API/DB`); a lower layer must never import from a higher layer (e.g. `services` must never import from `hooks` or `components`).
  - **UI & Database Decoupling**: The UI layer must **never** connect or communicate directly with the database. Components only interact with data through custom hooks.
  - **In-Memory Store First & Optimistic Updates**: All data mutations (create, update, delete) are applied temporarily/optimistically to the state management system (Zustand store) first, and then persisted immediately to the local database via services.
  - **Startup Hydration & Store-Backed Reads**: At application startup, all database models are imported/hydrated from SQLite into their respective stores. Throughout the app, data is read exclusively from these stores via custom hooks, ensuring instantaneous, reactive in-memory reads.
  - **SOLID**: especially Single Responsibility (one reason to change per file/component/hook) and Dependency Inversion (depend on `types`/interfaces, not concrete implementations, where it adds value).
  - **DRY**: shared logic goes into `hooks`/`utils`, shared data shapes go into `types`, shared fixtures go into `tests/factories`.
  - **KISS/YAGNI**: prefer the simplest solution that satisfies the requirement; do not add abstraction layers or configuration for hypothetical future needs.

---

## 2. Folder Structure

```
src/
├── components/       # Reusable UI components
├── context/          # React Context Providers
├── hooks/            # Generic and feature-related custom hooks
├── queries/          # Remote data-fetching / sync layer (React Query), per model
├── services/         # API / database communication layer, per model
├── stores/           # Global state management & in-memory model stores (Zustand)
├── types/            # Type and Interface definitions, per model
├── utils/            # Pure helper functions
├── libs/             # Third-party library configs/wrappers (axios, storage, ...)
├── consts/           # Constant values (colors, routes, enums, config)
├── tests/            # Test utilities, mocks, and setup shared across the app
```

### Rules per Folder

| Folder | Responsibility | Notes |
|---|---|---|
| `components` | Pure UI, no direct DB or API calls | Organize by feature or atomic design (`common/`, `screens/`, ...). Only consumes `hooks` |
| `context` | Context API + Providers | Only for state that is truly global/cross-cutting |
| `hooks` | Reusable logic & data access facade | Reads from stores via selectors, triggers optimistic store updates & immediate DB persistence |
| `queries` | Remote sync & query/mutation definitions (e.g. React Query) | Consumes `services` directly for remote API sync |
| `services` | API/database calls (axios, fetch, SQLite) | Handles raw DB/API calls: hydration on boot (`getAll`) and immediate persistence on mutations. **Only** layer allowed to touch SQLite directly |
| `stores` | App-wide state & in-memory model stores via **Zustand** | Hydrated from SQLite on startup; updated optimistically on mutations; read via hooks |
| `types` | Interface/Type + **Zod** schema per model | Types are inferred from Zod schemas (single source of truth) |
| `utils` | Pure, side-effect-free functions | Date formatting, formatting helpers, etc. |
| `libs` | Third-party library setup | e.g. `libs/axios.ts`, `libs/sqlite.ts`, `libs/storage.ts` |
| `consts` | Project-wide constant values | Routes, colors, repeated strings, enums |
| `tests` | Shared test setup, mocks, factories, test utils | Per-unit tests live next to their source file, not here |

---

## 3. Layered Architecture per Database Model

For **every database model** (e.g. `User`, `Product`, `Order`), the following structure must be created across the relevant folders:

```
types/
└── user/
    ├── User.types.ts          # Interfaces and types related to User
    └── User.schema.ts         # Zod schema (source of truth for types)

stores/
└── user/
    ├── user.store.ts          # In-memory model store (hydrated on boot, optimistic updates)
    ├── user.store.test.ts     # Store unit tests
    └── index.ts

services/
└── user/
    ├── user.local.service.ts  # SQLite local operations (getAll for hydration, CRUD for persistence)
    ├── user.service.ts        # Optional remote API calls (CRUD)
    ├── user.local.service.test.ts # Local DB service tests
    └── index.ts                # barrel export

queries/
└── user/
    ├── useUserQuery.ts         # Remote sync / background queries (if remote API is used)
    └── index.ts

hooks/
└── user/
    ├── useUser.ts               # Reads from store, performs optimistic updates & immediate DB persistence
    ├── useUser.test.ts          # Unit tests for the hook
    └── index.ts

components/
└── user/
    ├── UserProfile.tsx
    ├── UserProfile.test.tsx    # Component test (render + interaction)
    └── index.ts
```

### Data Flow Between Layers

Data flow strictly adheres to three distinct pipelines:

```
1. App Startup (Hydration):
   SQLite DB  →  userLocalService.getAllUsers()  →  useUserStore.hydrateUsers()

2. Read Flow (In-Memory Reactivity):
   Component  →  useUser()  →  reads directly from useUserStore (Zustand)

3. Write Flow (Store First, Immediate DB Persistence):
   Component  →  useUser().createUser(payload)
                     ↓
         1. Apply immediately to useUserStore (optimistic / temporary update)
                     ↓
         2. Persist immediately to userLocalService (SQLite)
            (Rollback store update if DB write fails)
```

- **UI & Database Separation**: The UI (`components`) must **never** communicate directly with the database. Components only call `hooks`.
- **In-Memory Store as Single Source of Truth**: At application launch, all persistent models are fetched from SQLite and hydrated into their corresponding Zustand stores.
- **Hook-Based Reads**: Within the app, components read data exclusively from these stores via custom hooks (selectors), ensuring instantaneous UI renders without database latency.
- **Optimistic State & Immediate Persistence**:
  1. Any modification is applied immediately to the Zustand store so the UI updates instantly.
  2. Immediately following the store update, the change is written to SQLite via the service.
  3. If SQLite write fails, the store is rolled back and the error is surfaced.

### Example for the `User` Model

**`types/user/User.types.ts`**
```ts
import { z } from 'zod';

export const userSchema = z.object({
  id: z.string(),
  name: z.string().min(2),
  email: z.string().email(),
  createdAt: z.string(),
});

export const createUserSchema = userSchema.omit({ id: true, createdAt: true });

export type User = z.infer<typeof userSchema>;
export type CreateUserPayload = z.infer<typeof createUserSchema>;
```

**`stores/user/user.store.ts`**
```ts
import { create } from 'zustand';
import type { User } from '@/types';

interface UserState {
  users: Record<string, User>;
  hydrateUsers: (users: User[]) => void;
  addUser: (user: User) => void;
  updateUser: (user: User) => void;
  removeUser: (id: string) => void;
}

export const useUserStore = create<UserState>((set) => ({
  users: {},
  hydrateUsers: (userList) =>
    set({
      users: Object.fromEntries(userList.map((u) => [u.id, u])),
    }),
  addUser: (user) =>
    set((state) => ({ users: { ...state.users, [user.id]: user } })),
  updateUser: (user) =>
    set((state) => ({ users: { ...state.users, [user.id]: user } })),
  removeUser: (id) =>
    set((state) => {
      const { [id]: _, ...rest } = state.users;
      return { users: rest };
    }),
}));
```

**`services/user/user.local.service.ts`**
```ts
import { db } from '@/libs';
import { userSchema } from '@/types';
import type { User } from '@/types';

export const userLocalService = {
  getAllUsers: async (): Promise<User[]> => {
    const rows = await db.getAllAsync('SELECT * FROM users');
    return rows.map((row) => userSchema.parse(row));
  },
  insertUser: async (user: User): Promise<void> => {
    await db.runAsync(
      'INSERT INTO users (id, name, email, createdAt) VALUES (?, ?, ?, ?)',
      [user.id, user.name, user.email, user.createdAt]
    );
  },
  updateUser: async (user: User): Promise<void> => {
    await db.runAsync(
      'UPDATE users SET name = ?, email = ? WHERE id = ?',
      [user.name, user.email, user.id]
    );
  },
  deleteUser: async (id: string): Promise<void> => {
    await db.runAsync('DELETE FROM users WHERE id = ?', [id]);
  },
};
```

**`hooks/user/useUser.ts`**
```ts
import { useUserStore } from '@/stores';
import { userLocalService } from '@/services';
import type { CreateUserPayload, User } from '@/types';

export const useUser = (id?: string) => {
  // Read state reactively from Zustand store
  const user = useUserStore((state) => (id ? state.users[id] : undefined));
  const allUsers = useUserStore((state) => Object.values(state.users));

  const addUser = useUserStore((state) => state.addUser);
  const removeUser = useUserStore((state) => state.removeUser);

  const createUser = async (payload: CreateUserPayload) => {
    const newUser: User = {
      id: crypto.randomUUID(),
      ...payload,
      createdAt: new Date().toISOString(),
    };

    // 1. Temporarily/optimistically apply to state management (instant UI update)
    addUser(newUser);

    try {
      // 2. Immediately persist to SQLite database
      await userLocalService.insertUser(newUser);
    } catch (error) {
      // Rollback store if DB write fails
      removeUser(newUser.id);
      throw error;
    }
  };

  return { user, allUsers, createUser };
};
```

**App Startup Hydration (`libs/bootstrap.ts`)**
```ts
import { userLocalService } from '@/services';
import { useUserStore } from '@/stores';

export const hydrateAllStores = async () => {
  // Load all persistent models from SQLite into their stores at boot
  const users = await userLocalService.getAllUsers();
  useUserStore.getState().hydrateUsers(users);
};
```

**Usage in a Component**
```tsx
import { useUser } from '@/hooks';

export const UserList = () => {
  // UI reads exclusively from the store via the hook
  const { allUsers, createUser } = useUser();

  return (
    <View>
      {allUsers.map((user) => (
        <Text key={user.id}>{user.name}</Text>
      ))}
      <Button
        title="Add User"
        onPress={() => createUser({ name: 'Alice', email: 'alice@example.com' })}
      />
    </View>
  );
};
```

---

## 4. State Management, Form Validation & Local Database

### 4.1 State Management — Zustand

- **Zustand** is the only allowed library for global and app-wide state, placed in `stores/`.
- **Dual Role**: Stores manage both **client/UI state** (session, theme, active filters, UI flags) and **in-memory model stores** (entities hydrated from the database).
- Each store is scoped to a single concern or model (e.g. `stores/session/session.store.ts`, `stores/user/user.store.ts`, `stores/order/order.store.ts`) — never create a monolithic "god store".
- **Startup Model Hydration**: When the application initializes/boots, all persistent models are loaded from SQLite via their respective `services` and hydrated into their Zustand stores (`hydrate*` action). The store serves as the fast in-memory cache and single source of truth during runtime.
- **Hook-Based Reads**: Components **never** read directly from the database or services. They read from these stores exclusively via custom hooks with fine-grained selectors:
  ```ts
  const user = useUserStore((state) => state.users[userId]);
  ```
- **Store-First (Optimistic) Mutations & Immediate DB Persistence**:
  1. Any write operation (create, update, delete) is applied temporarily/optimistically to the Zustand store first, ensuring instantaneous UI reactivity with 0ms latency.
  2. The custom hook immediately persists the updated model to the local SQLite database via its `service`.
  3. If the database write fails, the hook rolls back the store modification and surfaces an error.
- Stores must expose explicit actions (`hydrate*`, `add*`, `update*`, `remove*`) — never mutate store state directly from outside.
- Naming: `useXStore` for the store hook itself (e.g. `useSessionStore`, `useUserStore`), file name `x.store.ts`.

```ts
// stores/session/session.store.ts
import { create } from 'zustand';

interface SessionState {
  userId: string | null;
  isAuthenticated: boolean;
  setSession: (userId: string) => void;
  clearSession: () => void;
}

export const useSessionStore = create<SessionState>((set) => ({
  userId: null,
  isAuthenticated: false,
  setSession: (userId) => set({ userId, isAuthenticated: true }),
  clearSession: () => set({ userId: null, isAuthenticated: false }),
}));
```

### 4.2 Form Validation — Zod

- **Zod** is the only allowed library for schema and form validation.
- Each model's Zod schema lives next to its types, e.g. `types/user/User.schema.ts`, and the TypeScript type is **inferred from the schema** (single source of truth):

```ts
// types/user/User.schema.ts
import { z } from 'zod';

export const createUserSchema = z.object({
  name: z.string().min(2),
  email: z.string().email(),
});

export type CreateUserPayload = z.infer<typeof createUserSchema>;
```

- `services` use the schema to validate/parse data at the boundary (API responses, DB rows) before returning it.
- `hooks` use the schema to validate form input (e.g. with `react-hook-form` + `@hookform/resolvers/zod`) before calling a mutation.
- Never duplicate a hand-written `interface` alongside a Zod schema for the same shape — infer the type from the schema instead.

### 4.3 Local Database — SQLite

- **SQLite** (e.g. `expo-sqlite` / `react-native-sqlite-storage` via a wrapper in `libs/sqlite.ts`) is the only allowed local persistence layer for structured/relational data.
- The SQLite client/connection is configured **once** in `libs/sqlite.ts` and is never imported outside `services/`.
- **The UI must NEVER connect to or interact with SQLite directly.** UI components interact exclusively through custom `hooks/`.
- **No layer other than `services` may ever execute a raw SQL query or call the SQLite client directly.** This applies to `components`, `hooks`, `queries`, `stores`, and `context` alike.
- **Two Touchpoints for SQLite Services**:
  1. **Startup Hydration**: Services expose `getAll*` methods used during app boot to hydrate all persistent data into Zustand stores.
  2. **Immediate Persistence**: Services expose write methods (`insert*`, `update*`, `delete*`) invoked by hooks immediately after applying optimistic in-memory updates in Zustand.
- The mandatory data access chains are:
  - **Reads**: `Component → hooks/ → store (Zustand)` (in-memory, instant)
  - **Writes**: `Component → hooks/ → store (optimistic update) → services/ → SQLite DB (immediate persistence)`
  - **Startup**: `App Bootstrap → services/ (getAll) → stores.hydrate()`
- Schema/migrations for SQLite live in `libs/sqlite/migrations/`, applied once at app startup — never ad hoc inside a service call.

---

## 5. Testing

Every layer must be covered by automated tests. Tests are **colocated** with the file they test (e.g. `user.service.ts` → `user.service.test.ts`), while shared, cross-cutting test infrastructure lives in `tests/`.

### `tests/` Folder

```
tests/
├── setup.ts             # Global test setup (jest-setup, RN mocks, etc.)
├── mocks/                # Shared mocks (API client, AsyncStorage, navigation, ...)
├── factories/            # Test data factories/builders per model (e.g. userFactory.ts)
├── utils/                # Test helpers (custom render with Providers, wait helpers, ...)
```

- `tests/setup.ts` is referenced from the Jest config (`setupFilesAfterEach` / `setupFiles`).
- `tests/utils/render.tsx` should export a custom `render` that wraps components with all required Providers (Context, React Query, theme, navigation) so component tests don't repeat boilerplate.
- `tests/factories/` should provide functions like `createMockUser(overrides?)` that return objects typed against `types/`, used across service/query/hook/component tests to avoid duplicated fixtures.
- `tests/mocks/` holds reusable mocks for `libs/` (axios client, storage, push notifications, etc.), injected via Jest module mocks.

### Testing Rules per Layer

| Layer | What to test | Tooling |
|---|---|---|
| `utils` | Pure function output for given inputs, edge cases | Jest |
| `services` | Correct API calls, request/response mapping, error propagation (with `libs` mocked) | Jest + mocked `apiClient` |
| `queries` | Correct query key, loading/success/error states (with `services` mocked) | Jest + `@testing-library/react-hooks` / React Query test utils |
| `hooks` | Combined logic, derived state, side effects (with `queries` mocked) | Jest + `@testing-library/react-hooks` |
| `stores` | State transitions/actions produce the expected state | Jest |
| `components` | Rendering, user interaction, accessibility (with `hooks` mocked) | `@testing-library/react-native` |
| End-to-end flows | Critical user journeys (login, checkout, etc.) | Detox / Maestro |

### Conventions

- Test file naming: `*.test.ts` / `*.test.tsx`, placed next to the source file.
- Follow the **Arrange–Act–Assert** pattern inside each test.
- Mock only the layer directly below the unit under test (e.g. when testing `hooks`, mock `queries` — don't mock `services` two layers down).
- Every new model added per the checklist in Section 7 must ship with at least: one service test, one query test, one hook test, and one component test.
- Snapshot tests should be used sparingly and only for stable, presentational components.
- Aim for meaningful coverage of business logic (services, hooks, stores, utils) over raw percentage targets.

---

## 6. Barrel Imports

Every folder (and every model sub-folder) must have an `index.ts` that re-exports its contents.

**`types/index.ts`**
```ts
export * from './user/User.types';
export * from './product/Product.types';
```

**`services/index.ts`**
```ts
export * from './user';
export * from './product';
```

**`hooks/index.ts`**
```ts
export * from './user';
export * from './product';
```

Rules:
- Imports throughout the project must always come from the root of each layer, never from an internal file:
  ```ts
  // ✅ Correct
  import { useUser, userService } from '@/hooks';

  // ❌ Wrong
  import { useUser } from '@/hooks/user/useUser';
  ```
- Use **path aliases** for module resolution (`tsconfig.json` / `babel.config.js`):
  ```json
  {
    "paths": {
      "@/components": ["src/components"],
      "@/hooks": ["src/hooks"],
      "@/queries": ["src/queries"],
      "@/services": ["src/services"],
      "@/types": ["src/types"],
      "@/utils": ["src/utils"],
      "@/stores": ["src/stores"],
      "@/context": ["src/context"],
      "@/libs": ["src/libs"],
      "@/consts": ["src/consts"]
    }
  }
  ```
- Prefer **named exports** over `export default` so barrel imports stay clean and consistent.
- `index.ts` files must contain no logic — exports only.

---

## 7. Checklist for Adding a New Model

When adding a new database model (e.g. `Order`), the agent must follow these steps in order:

1. `types/order/Order.types.ts` + `types/order/Order.schema.ts` → define the Zod schema and infer the Type from it
2. `services/order/order.local.service.ts` (SQLite) → `getAllOrders()` for startup hydration, plus CRUD operations (`insertOrder`, `updateOrder`, `deleteOrder`), using the Zod schema to parse/validate at the boundary + `*.local.service.test.ts`
3. If remote sync is needed: `services/order/order.service.ts` (remote) + `queries/order/useOrderQuery.ts` + query tests
4. `stores/order/order.store.ts` → Zustand store holding the in-memory entity map/list, `hydrateOrders`, and optimistic mutation actions + `order.store.test.ts`
5. Register model in app startup bootstrap (`libs/bootstrap.ts`) so `getAllOrders()` hydrates `useOrderStore` on app launch
6. `hooks/order/useOrder.ts` → custom hook exposing store selectors for reading data, plus mutation functions that apply immediate store updates and immediately persist to `orderLocalService` (with rollback on error) + `useOrder.test.ts`
7. Related UI components in `components/order/`, split into container (logic) + presentational (UI) as needed, consuming `useOrder()` only + component tests
8. Update `index.ts` in every relevant folder (barrel export)
9. If constant values are needed (status enums, etc.) → `consts/order.consts.ts`
10. If the model is persisted locally, add its table/columns to `libs/sqlite/migrations/`
11. If needed, add factories/mocks for `Order` in `tests/factories/` and `tests/mocks/`

---

## 8. Forbidden Practices

- Direct communication between the UI (`components`) and the database (SQLite or raw API).
- Reading data directly from SQLite or services inside UI components instead of reading from stores via custom hooks.
- Persisting to SQLite without first applying the change to the state management system (Zustand).
- Mutating the store without immediately persisting changes to the local database (when data is persistent).
- Omitting startup model hydration from SQLite into stores on app launch.
- Calling `fetch`/`axios` directly inside a component.
- Writing business logic inside `components`.
- Using `any` or `as any` without justification.
- Importing directly from an internal file instead of the barrel (`index.ts`).
- Duplicating the same type definition across multiple files.
- Using Class Components.
- Hardcoding secrets/tokens in code (must live in `.env`).
- Shipping a new service, query, hook, store, or component without a corresponding test.
- Duplicating test fixtures/mocks instead of reusing `tests/factories/` and `tests/mocks/`.
- Mixing UI rendering and business/data logic in the same component instead of splitting into container + presentational components.
- Violating the Dependency Rule (e.g. a `service` importing from a `hook` or `component`).
- Using any state library other than **Zustand** for state management, or any validation library other than **Zod** for schemas/forms.
- Writing a raw SQL query or calling the SQLite client from anywhere outside `services/`.
- Calling a `service` function directly from a `component` — services must always be reached through a `hook`.
- Hand-writing a `interface`/`type` that duplicates a Zod schema's shape instead of using `z.infer`.
