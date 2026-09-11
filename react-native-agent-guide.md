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
| `services` | API/database calls (axios, fetch, SDK) | No UI logic; pure input/output of data |
| `stores` | App-wide state | e.g. user session, settings, theme |
| `types` | Interface/Type per model | Single source of truth for data shape |
| `utils` | Pure, side-effect-free functions | Date formatting, validation, etc. |
| `libs` | Third-party library setup | e.g. `libs/axios.ts`, `libs/storage.ts` |
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

## 4. Testing

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
- Every new model added per the checklist in Section 6 must ship with at least: one service test, one query test, one hook test, and one component test.
- Snapshot tests should be used sparingly and only for stable, presentational components.
- Aim for meaningful coverage of business logic (services, hooks, stores, utils) over raw percentage targets.

---

## 5. Barrel Imports

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

## 6. Checklist for Adding a New Model

When adding a new database model (e.g. `Order`), the agent must follow these steps in order:

1. `types/order/Order.types.ts` → define Types/Interfaces
2. `services/order/order.service.ts` → raw CRUD calls + `order.service.test.ts`
3. `queries/order/useOrderQuery.ts` + `useOrderMutation.ts` + query tests
4. `hooks/order/useOrder.ts` → feature-level logic + `useOrder.test.ts`
5. Related UI components in `components/order/` + component tests
6. Update `index.ts` in every relevant folder (barrel export)
7. If global state is needed → `stores/order.store.ts` (+ store test)
8. If constant values are needed (status enums, etc.) → `consts/order.consts.ts`
9. If needed, add factories/mocks for `Order` in `tests/factories/` and `tests/mocks/`

---

## 7. Forbidden Practices

- Calling `fetch`/`axios` directly inside a component.
- Writing business logic inside `components`.
- Using `any` or `as any` without justification.
- Importing directly from an internal file instead of the barrel (`index.ts`).
- Duplicating the same type definition across multiple files.
- Using Class Components.
- Hardcoding secrets/tokens in code (must live in `.env`).
- Shipping a new service, query, hook, or component without a corresponding test.
- Duplicating test fixtures/mocks instead of reusing `tests/factories/` and `tests/mocks/`.
