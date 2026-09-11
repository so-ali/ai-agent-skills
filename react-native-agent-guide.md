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
  - **Dependency Rule**: dependencies only point inward/downward (`components → hooks → queries → services → API`); a lower layer must never import from a higher layer (e.g. `services` must never import from `hooks` or `components`).
  - **SOLID**: especially Single Responsibility (one reason to change per file/component/hook) and Dependency Inversion (depend on `types`/interfaces, not concrete implementations, where it adds value).
  - **DRY**: shared logic goes into `hooks`/`utils`, shared data shapes go into `types`, shared fixtures go into `tests/factories`.
  - **KISS/YAGNI**: prefer the simplest solution that satisfies the requirement; do not add abstraction layers or configuration for hypothetical future needs.

---

## 2. Folder Structure

```
src/
├── components/       # Reusable UI components
├── context/           # React Context Providers
├── hooks/             # Generic and feature-related custom hooks
├── queries/           # Data-fetching layer (React Query), per model
├── services/          # API / database communication layer, per model
├── stores/            # Global state management (Zustand/Redux/...)
├── types/             # Type and Interface definitions, per model
├── utils/             # Pure helper functions
├── libs/              # Third-party library configs/wrappers (axios, storage, ...)
├── consts/            # Constant values (colors, routes, enums, config)
├── tests/             # Test utilities, mocks, and setup shared across the app
```

### Rules per Folder

| Folder | Responsibility | Notes |
|---|---|---|
| `components` | Pure UI, no direct API calls | Organize by feature or atomic design (`common/`, `screens/`, ...) |
| `context` | Context API + Providers | Only for state that is truly global/cross-cutting |
| `hooks` | Reusable logic | Each hook should have a single responsibility |
| `queries` | Query/mutation definitions (e.g. React Query) | Consumes `services` directly, never raw API calls |
| `services` | API/database calls (axios, fetch, SQLite) | No UI logic; pure input/output of data. **Only** layer allowed to touch SQLite directly |
| `stores` | App-wide **client** state via **Zustand** | e.g. user session, theme, UI flags — never server/DB data |
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
    └── User.types.ts          # Interfaces and types related to User

services/
└── user/
    ├── user.service.ts        # Raw API calls (CRUD)
    ├── user.service.test.ts   # Unit tests for the service
    └── index.ts                # barrel export

queries/
└── user/
    ├── useUserQuery.ts         # Read operations (get, list)
    ├── useUserMutation.ts      # Write operations (create, update, delete)
    ├── useUserQuery.test.ts    # Unit tests for the query hooks
    └── index.ts

hooks/
└── user/
    ├── useUser.ts               # Combines query/mutation + feature/form logic
    ├── useUser.test.ts          # Unit tests for the hook
    └── index.ts

components/
└── user/
    ├── UserProfile.tsx
    ├── UserProfile.test.tsx    # Component test (render + interaction)
    └── index.ts
```

### Data Flow Between Layers

```
Component  →  hooks/  →  queries/  →  services/  →  API / DB
                ↑             ↑            ↑
             types/  ←────────┴────────────┘
```

- **types**: defines the shape of the data (Model, DTO, Request/Response).
- **services**: purely raw communication with the API; output must match `types`.
- **queries**: uses `services` and manages server state/cache via React Query (or similar).
- **hooks**: implements feature-level composite logic (forms, validation, combining multiple queries) on top of `queries`.
- **components**: only consume `hooks`; never access `services` or `queries` directly.

### Example for the `User` Model

**`types/user/User.types.ts`**
```ts
export interface User {
  id: string;
  name: string;
  email: string;
  createdAt: string;
}

export interface CreateUserPayload {
  name: string;
  email: string;
}
```

**`services/user/user.service.ts`**
```ts
import { apiClient } from '@/libs';
import type { User, CreateUserPayload } from '@/types';

export const userService = {
  getUser: (id: string) => apiClient.get<User>(`/users/${id}`),
  createUser: (payload: CreateUserPayload) => apiClient.post<User>('/users', payload),
};
```

**`queries/user/useUserQuery.ts`**
```ts
import { useQuery } from '@tanstack/react-query';
import { userService } from '@/services';

export const useUserQuery = (id: string) =>
  useQuery({
    queryKey: ['user', id],
    queryFn: () => userService.getUser(id),
  });
```

**`hooks/user/useUser.ts`**
```ts
import { useUserQuery } from '@/queries';

export const useUser = (id: string) => {
  const { data, isLoading, error } = useUserQuery(id);
  return { user: data, isLoading, error };
};
```

**Usage in a Component**
```tsx
import { useUser } from '@/hooks';

const UserProfile = ({ id }: { id: string }) => {
  const { user, isLoading } = useUser(id);
  if (isLoading) return <Loading />;
  return <Text>{user?.name}</Text>;
};
```

---

## 4. State Management, Form Validation & Local Database

### 4.1 State Management — Zustand

- **Zustand** is the only allowed library for global/app-wide state, placed in `stores/`.
- Each store is scoped to a single concern (e.g. `stores/session.store.ts`, `stores/theme.store.ts`, `stores/order.store.ts`) — no single "god store".
- Stores hold **client/UI state** (session, theme, filters, onboarding flags, ephemeral UI state) — never server or database data. Server/DB data always flows through `services` → `queries`/`hooks`, not through Zustand.
- Stores are consumed directly by `hooks` (and, when trivial, by presentational-adjacent logic inside container components) — never mutated directly from deep inside presentational components; expose actions from the store instead of letting components write to state directly.
- Naming: `useXStore` for the store hook itself (e.g. `useSessionStore`), file name `x.store.ts`.

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
- **No layer other than `services` may ever execute a raw SQL query or call the SQLite client directly.** This applies to `components`, `hooks`, `queries`, `stores`, and `context` alike.
- Each model has its own DB-facing service function set, e.g. `services/user/user.local.service.ts`, alongside (or instead of) its remote `user.service.ts`, both exposed through the same barrel:

```ts
// services/user/user.local.service.ts
import { db } from '@/libs';
import { userRowSchema } from '@/types';
import type { User } from '@/types';

export const userLocalService = {
  getUser: async (id: string): Promise<User> => {
    const row = await db.getFirstAsync('SELECT * FROM users WHERE id = ?', [id]);
    return userRowSchema.parse(row);
  },
  createUser: async (user: User): Promise<void> => {
    await db.runAsync(
      'INSERT INTO users (id, name, email, createdAt) VALUES (?, ?, ?, ?)',
      [user.id, user.name, user.email, user.createdAt]
    );
  },
};
```

- **Services must always be consumed through hooks, never called directly from components or from `stores`.** The mandatory chain for any data access (remote or local) is:

  ```
  Component → hooks/ → (queries/ →) services/ → SQLite / API
  ```

  - `queries/` may sit between `hooks` and `services` when caching/invalidation via React Query is useful (typical for reads).
  - For simple local writes, a `hook` may call the `service` directly without a `queries` layer, but a `component` must never import a `service` itself.
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
2. `services/order/order.service.ts` (remote) and/or `services/order/order.local.service.ts` (SQLite) → raw CRUD calls, using the Zod schema to parse/validate at the boundary + `*.service.test.ts`
3. `queries/order/useOrderQuery.ts` + `useOrderMutation.ts` + query tests (only if caching/invalidation is needed on top of the service)
4. `hooks/order/useOrder.ts` → feature-level logic, form validation via the Zod schema, and the **only** entry point components use to reach `services`/`queries` + `useOrder.test.ts`
5. Related UI components in `components/order/`, split into container (logic) + presentational (UI) as needed, + component tests
6. Update `index.ts` in every relevant folder (barrel export)
7. If global/UI state is needed → `stores/order.store.ts` using Zustand (+ store test)
8. If constant values are needed (status enums, etc.) → `consts/order.consts.ts`
9. If the model is persisted locally, add its table/columns to `libs/sqlite/migrations/`
10. If needed, add factories/mocks for `Order` in `tests/factories/` and `tests/mocks/`

---

## 8. Forbidden Practices

- Calling `fetch`/`axios` directly inside a component.
- Writing business logic inside `components`.
- Using `any` or `as any` without justification.
- Importing directly from an internal file instead of the barrel (`index.ts`).
- Duplicating the same type definition across multiple files.
- Using Class Components.
- Hardcoding secrets/tokens in code (must live in `.env`).
- Shipping a new service, query, hook, or component without a corresponding test.
- Duplicating test fixtures/mocks instead of reusing `tests/factories/` and `tests/mocks/`.
- Mixing UI rendering and business/data logic in the same component instead of splitting into container + presentational components.
- Violating the Dependency Rule (e.g. a `service` importing from a `hook` or `component`).
- Using any state library other than **Zustand** for global state, or any validation library other than **Zod** for schemas/forms.
- Storing server/DB data inside a Zustand store instead of fetching it through `services`/`queries`.
- Writing a raw SQL query or calling the SQLite client from anywhere outside `services/`.
- Calling a `service` function directly from a `component` — services must always be reached through a `hook`.
- Hand-writing a `interface`/`type` that duplicates a Zod schema's shape instead of using `z.infer`.
