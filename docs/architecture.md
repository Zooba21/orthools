# Architecture — Orthools

## What this platform is

Orthools is a multi-module platform for speech therapists (orthophonistes). Each module proposes a distinct type of therapeutic exercise. Modules share a common shell (navigation, layout) and technical foundations (architecture, DI, testing strategy) but own their domain entirely. A new module adds a folder; it changes nothing already in place.

---

## Guiding principles

**Hexagonal architecture (Ports & Adapters):** the domain and application layers are completely framework-agnostic. Everything else adapts to them. Swapping the data source means changing one DI binding, nothing else.

**Screaming architecture:** every folder name tells you *what the app does*, not what pattern or framework it uses. A reader opening `presentation/` should see `lexical-evocation/`, `[next-module]/`, `platform/` — not `components/`, `services/`, `utils/`.

---

## What is a "port"?

In hexagonal architecture, a **port** is a TypeScript interface that lives inside the domain layer and describes a capability the application needs — without saying anything about how that capability is fulfilled.

Think of a power socket: the socket defines a standard shape (voltage, pins). Any compatible plug can fill it. The socket is the port; the plug is the adapter.

A domain port:

```
IInstructionRepository
  findAll(): Promise<Instruction[]>
  findById(id: string): Promise<Instruction | null>
  save(instruction: Instruction): Promise<void>
  delete(id: string): Promise<void>
```

The application layer calls `IInstructionRepository` everywhere. It never knows whether the actual implementation holds data in a JavaScript array, calls a REST API, or queries a SQL database. That decision lives entirely in the persistence layer and is wired up at startup via DI.

Two port directions:
- **Driven ports** (right side of the hexagon): the application *drives* them — repository interfaces, external service interfaces. These are the ports you define most often.
- **Driving ports** (left side): external actors *drive* the application — in React, the hook/facade implicitly plays this role by calling use cases directly.

---

## Layer responsibilities

```
┌────────────────────────────────────────────────────────────────────┐
│  Presentation   (React: pages, feature components, UI system)      │
│  Depends on → Application layer (via custom hooks)                 │
├────────────────────────────────────────────────────────────────────┤
│  Application    (Use cases — plain TypeScript, no framework)       │
│  Depends on → Domain ports (interfaces only)                       │
├────────────────────────────────────────────────────────────────────┤
│  Domain         (Entities, value objects, domain services, ports)  │
│  Depends on → nothing                                              │
├────────────────────────────────────────────────────────────────────┤
│  Persistence    (Adapters: in-memory → api → …)                    │
│  Implements → Domain ports; uses → Domain entities                 │
└────────────────────────────────────────────────────────────────────┘
```

Dependency arrows flow inward only. Persistence and Presentation sit at the edges; Domain is the centre.

---

## Folder structure

Each module owns a namespace in every layer. Layers are siblings at `app/` level; modules are namespaces within each layer.

```
public/                                      # Static assets — served at root URL, no path prefix
├── favicon.ico
├── fonts/
│   └── inter/                               # .woff2 files, referenced from globals.css
└── images/

src/
└── app/
    │
    ├── domain/
    │   └── lexical-evocation/               # Module domain namespace
    │       ├── instruction/
    │       │   ├── instruction.entity.ts
    │       │   ├── instruction.repository.ts  # Port: IInstructionRepository
    │       │   └── instruction-type.vo.ts
    │       ├── category/
    │       │   ├── category.entity.ts
    │       │   └── category.repository.ts
    │       └── exercise/
    │           ├── exercise-set.entity.ts
    │           ├── exercise-set.repository.ts
    │           ├── exercise-generator-config.vo.ts
    │           └── exercise-generator.domain-service.ts
    │
    ├── application/
    │   └── lexical-evocation/
    │       ├── instruction/
    │       │   ├── list-instructions.use-case.ts
    │       │   └── create-instruction.use-case.ts
    │       └── exercise/
    │           └── generate-exercise.use-case.ts
    │
    ├── persistence/
    │   └── lexical-evocation/
    │       ├── instruction.in-memory.repository.ts
    │       ├── category.in-memory.repository.ts
    │       └── exercise-set.in-memory.repository.ts
    │         └── api/                         # Phase 2 — remote adapters
    │             └── instruction.api.repository.ts
    │
    └── presentation/
        │
        ├── platform/                          # Platform shell & home
        │   ├── home/
        │   │   └── platform-home.page.tsx     # Lists all available modules
        │   └── layout/
        │       ├── shell/
        │       └── nav/
        │
        ├── lexical-evocation/                 # Module 1 presentation
        │   ├── home/
        │   ├── generator/
        │   │   ├── configurator/
        │   │   ├── preview/
        │   │   └── execution/
        │   ├── instructions/
        │   ├── categories/
        │   ├── exercises/
        │   └── components/                    # Module-scoped shared components
        │
        └── ui/                                # Design system — domain-agnostic
            ├── color-dot/
            ├── data-table/
            ├── form-field/
            └── badge/
```

When a second module is added, it follows the same pattern:

```
domain/[module-name]/
application/[module-name]/
persistence/[module-name]/
presentation/[module-name]/
```

No existing file changes. No shared code is modified.

---

## UI stack

Full rationale and token reference: [Design System](../design-system/overview.md).

| Concern | Choice |
|---------|--------|
| Component library | **shadcn/ui** — composable, unstyled Radix primitives with Tailwind styling; copy-own model means full control |
| Headless primitives | **Radix UI** — focus management, live announcer, overlay, accessible comboboxes; ships inside shadcn components |
| Theming | **`src/styles/globals.css`** — CSS custom properties; uses the shadcn token naming convention (`--background`, `--primary`, etc.) |
| Custom component styles | **CSS Modules** (`.module.css` co-located) — scoped styles for components that go beyond what shadcn provides |
| Tailwind | Used as shadcn's styling engine. Utility classes appear inside shadcn component source and layout composition; domain templates reference semantic component names (`<Button>`, `<Card>`) not class chains |

### Tailwind and template readability

shadcn/ui components (`<Button>`, `<Card>`, `<FormField>`) encapsulate their Tailwind classes internally. What developers write in feature templates is still `<Button variant="outline">` — no visible class chains. Tailwind classes appear in `ui/` component implementations and layout scaffolding, not in domain feature code.

### The `ui/` layer

`presentation/ui/` holds two kinds of components:

- **shadcn/ui extensions** — thin wrappers around shadcn primitives that encode a project-wide convention (e.g., `AppButton` always defaults to `variant="outline"` with correct sizing). Only created when a convention needs enforcing; raw shadcn components are used directly otherwise.
- **Custom components** — where shadcn/Radix has no equivalent (`ColorDotComponent`) or where clinical requirements demand a fully custom implementation.

Domain components (inside module folders) compose from `ui/` components. `ui/` components never import domain types.

### Storybook

Stories are co-located with the component (`color-dot.stories.tsx` next to `color-dot.tsx`). Every `ui/` component has a story. Stories cover all visual states and accessibility states. The `@storybook/addon-a11y` WCAG check is the per-component accessibility gate — a story that fails axe is not done. Storybook's Vitest integration compiles stories into test cases, eliminating duplication between visual and unit tests.

---

## Key concepts

### "api" adapter — why not "http"?

Hexagonal convention names adapters after the *system* they connect to, not the *protocol*. Examples from other ecosystems: `TypeOrmInstructionRepository`, `MongoInstructionRepository`. None of those say "sql" or "tcp".

`HttpInstructionRepository` names the transport, not the system. If the backend later uses tRPC, gRPC, or WebSockets, the name becomes a lie. `ApiInstructionRepository` is protocol-agnostic and stable across backend technology changes.

### Pages — URL structure vs feature namespace?

In practice these are the same thing. `/lexical-evocation`, `/lexical-evocation/generator`, `/lexical-evocation/instructions` are both URL paths and domain feature names simultaneously. The URL should reflect the domain — screaming architecture flows all the way to the address bar.

Where the two diverge is in depth. A URL like `/lexical-evocation/instructions/:id/categories/new` might suggest deep nesting; feature namespace keeps it flat: `instructions/` handles the instruction context, route params carry the id, a modal or inline form handles the sub-action. Prefer flat feature namespaces over folder trees mirroring URL depth.

### Components — feature namespace and the design system split

**Domain components** live inside a module's `components/` folder. They know the domain vocabulary (`Instruction`, `Category`, `Slide`). They compose UI components with domain data.

**UI components** live in the global `ui/` folder. They know nothing about any domain. `Card` accepts `title`, `content`, `actions` — that is all. A `SlideCard` in the module *uses* `Card` the same way a future module's component would. The design system stays coherent because `Card` never learns what a `Slide` is.

The same split applies to forms. `FormField` is a generic labeled input. `InstructionForm` assembles `FormField`s with instruction-specific labels. Reuse happens at the design system level, not the domain level.

### The hook — not "services"

The feature hook is a custom React hook that lives inside the feature folder it orchestrates. It:

1. Reads repository instances from React Context (use cases are plain TypeScript; they receive repositories by constructor).
2. Exposes reactive state via `useState`/`useReducer` that pages and domain components consume.

Named after the feature it drives: `use-generator.hook.ts`, not `exercise.service.ts`. Co-located with the feature. One per sub-feature inside a module.

```
ConfiguratorPage  →  useGenerator()  →  GenerateExerciseUseCase
                                               ↓
                                     IInstructionRepository
                                               ↑
                                 InMemoryInstructionRepository (Context)
```

### Guards beyond authentication

Route guards are implemented as wrapper components (or React Router v7 loaders) that intercept navigation:

- `UnsavedChangesGuard`: wraps a route, shows a confirm dialog when navigating away from a dirty form.
- `ExerciseReadyGuard`: wraps `/generator/execute`, redirects to `/generator` if no exercise is in hook state.
- Future `AuthGuard`: checks whether a session exists.
- Future `RoleGuard`: checks whether the user holds the required permission.

All guard components live in `presentation/auth/guards/` regardless of whether they concern authentication.

```tsx
// exercise-ready.guard.tsx
export function ExerciseReadyGuard({ children }: { children: ReactNode }) {
  const { exercise } = useGeneratorContext();
  if (!exercise) return <Navigate to="/lexical-evocation/generator" replace />;
  return <>{children}</>;
}
```

### DI binding — the swap point

`src/main.tsx` (or a dedicated `providers.tsx`) is the composition root. The only place where concrete adapters are named:

```tsx
// Phase 1
<InstructionRepositoryContext.Provider value={new InMemoryInstructionRepository()}>
  <CategoryRepositoryContext.Provider value={new InMemoryCategoryRepository()}>
    <App />
  </CategoryRepositoryContext.Provider>
</InstructionRepositoryContext.Provider>

// Phase 2 — change these bindings only
<InstructionRepositoryContext.Provider value={new ApiInstructionRepository()}>
  ...
```

Every other file in the codebase stays untouched when the adapter changes.

---

## Routing

Pages are lazy-loaded with React Router v7's `lazy`. Route groups are lazy-loaded as route module files. Each module owns its route subtree.

```tsx
// app.routes.tsx
createBrowserRouter([
  { path: '/',                       element: <PlatformHomePage /> },
  { path: '/lexical-evocation/*',    lazy: () => import('./lexical-evocation/lexical-evocation.routes') },
])

// lexical-evocation.routes.tsx
[
  { index: true,                     element: <LexicalEvocationHomePage /> },
  { path: 'generator',               element: <ConfiguratorPage /> },
  { path: 'generator/execute',       element: <ExerciseReadyGuard><ExecutionPage /></ExerciseReadyGuard> },
  { path: 'instructions',            lazy: () => import('./instructions/list') },
  { path: 'instructions/create',     lazy: () => import('./instructions/create') },
  { path: 'instructions/:id/edit',   lazy: () => import('./instructions/edit') },
  { path: 'categories',              lazy: () => import('./categories') },
  { path: 'exercises',               lazy: () => import('./exercises') },
]
```

---

## Testing strategy

### The hexagonal advantage

The architecture's value for testing: each layer can be tested in isolation with zero ceremony. Use cases are plain classes — no test framework setup, no mocking framework. Domain objects are plain objects — no setup at all. The in-memory repository is a first-class test double that also verifies correct adapter behaviour via contract tests.

### Test doubles: fakes over mocks

| Double | Definition | When to use |
|--------|-----------|-------------|
| **Fake** | Working lightweight implementation of a port (the in-memory repository) | Almost everywhere — application layer, hook, page tests |
| **Stub** | Returns fixed data, no logic | When you only need one specific answer and a fake would be overkill |
| **Spy** | Wraps a real implementation to observe calls | Only when you need to assert a specific method was called with specific args |
| **Mock** | Pre-programmed with expected calls | Avoid — brittle, couples tests to implementation details |

### Testing pyramid

```
          ┌──────────────────────────────┐
          │  E2E — real backend          │  ← very few: smoke tests only
          ├──────────────────────────────┤
          │  E2E — in-memory app         │  ← some: full user flows, no infrastructure
          ├──────────────────────────────┤
          │  Integration (pages)         │  ← moderate: render + in-memory Context
          ├──────────────────────────────┤
          │  Unit (components, hooks,    │  ← many: fast, no network
          │  guards)                     │
          ├──────────────────────────────┤
          │  Unit (application, domain)  │  ← many: pure TypeScript, instant
          └──────────────────────────────┘
```

### Layer-by-layer

**Domain:** pure unit tests. No setup, no async unless explicitly needed. Test entity construction rules, value object invariants, business rules on entities, and domain service algorithms (e.g., the exercise generation algorithm).

**Application:** unit tests with fake repositories passed via constructor. No React, no mocking framework. Assert that the use case returns the correct result given known repo state, and that it mutates repo state correctly.

```typescript
const repo = new InMemoryInstructionRepository(); // fake
const useCase = new ListInstructionsUseCase(repo);
const result = await useCase.execute();
expect(result).toHaveLength(5); // seeded defaults
```

**Persistence — contract tests:** the key pattern. A shared spec (`instruction.repository.contract.ts`) defines all behavioural assertions against `IInstructionRepository`. Both `InMemoryInstructionRepository` and `ApiInstructionRepository` must pass the same suite. If both pass, the production swap is guaranteed safe.

For the API adapter: use `msw` to intercept HTTP calls. No real server needed.

**Presentation — UI components:** React Testing Library, rendering assertions only. Given these inputs, this text appears. No domain knowledge needed.

**Presentation — domain components:** same approach with domain-typed fixtures fed as props.

**Presentation — hooks:** `renderHook()` from React Testing Library with a Context wrapper providing fake repositories. Assert state returned by the hook.

```typescript
const repo = new InMemoryInstructionRepository();
const wrapper = ({ children }) => (
  <InstructionRepositoryContext.Provider value={repo}>
    {children}
  </InstructionRepositoryContext.Provider>
);
const { result } = renderHook(() => useGenerator(), { wrapper });
await act(() => result.current.generateExercise(config));
expect(result.current.exercise).not.toBeNull();
```

**Presentation — pages:** integration test with `render()` and in-memory Context providers. Assert rendered DOM, test navigation triggers.

**Guards:** render the guard component with a mock Context value. Assert `<Navigate>` is rendered when the condition is unmet, children are rendered when it is met.

### End-to-end strategy

**In-memory E2E (Playwright):** the React app built with in-memory providers. Tests full user flows — navigation, form interaction, exercise generation, execution. Fast, deterministic, no infrastructure.

**Real-backend smoke tests (few):** a handful of Playwright tests against a real backend instance. Verifies only that the DI swap from in-memory to the API adapter works end-to-end. Does not re-test business logic.

95% of tests run with no external infrastructure, ever.

### File placement

```
domain/lexical-evocation/instruction/
├── instruction.entity.ts
└── instruction.entity.spec.ts              # pure unit

application/lexical-evocation/instruction/
├── list-instructions.use-case.ts
└── list-instructions.use-case.spec.ts      # unit + fake repo

persistence/lexical-evocation/
├── instruction.repository.contract.ts      # shared contract (no spec suffix)
├── instruction.in-memory.repository.ts
└── instruction.in-memory.repository.spec.ts  # runs contract suite

presentation/lexical-evocation/generator/
├── use-generator.hook.spec.ts              # renderHook + in-memory Context
└── configurator/
    └── configurator.page.spec.tsx          # render() integration

e2e/
├── lexical-evocation-flow.spec.ts          # in-memory E2E (Playwright)
└── smoke/
    └── lexical-evocation-api.spec.ts       # real-backend smoke
```

### Tooling

| Tool | Purpose |
|------|---------|
| **Vitest** | Test runner |
| **@testing-library/react** | Component and page rendering |
| **@testing-library/user-event** | Realistic user input simulation |
| **Playwright** | E2E browser tests |
| **msw** (phase 2) | HTTP interception for API adapter contract tests |

---

## Code quality & git hooks

### ESLint

ESLint enforces consistent style and catches problems before they reach review. Configuration lives in `eslint.config.ts` at the project root.

Rules of note:

- **TypeScript strict rules** — `@typescript-eslint/recommended-type-checked` is the baseline. Unsafe `any`, unhandled promise returns, and missing return types are errors, not warnings.
- **Import discipline** — `eslint-plugin-import` enforces that dependency arrows never flow outward: `domain/` may not import from `application/`, `persistence/`, or `presentation/`; `application/` may not import from `persistence/` or `presentation/`. Violations are caught at lint time, not code review.
- **React hooks** — `eslint-plugin-react-hooks` enforces the rules of hooks and exhaustive deps.
- **Unused symbols** — unused variables and imports are errors. Dead code does not accumulate.

Run manually:

```bash
pnpm lint          # check
pnpm lint --fix    # auto-fix safe violations
```

### Husky + lint-staged

Husky wires git hooks so that quality gates run automatically at commit and push time. Configuration lives in `.husky/`.

#### `pre-commit` — fast, file-scoped

Runs `lint-staged`, which applies only to staged files:

```
*.{ts,tsx}  →  eslint --fix, then re-stage
*.{ts,tsx,css,md}  →  prettier --write, then re-stage
```

The commit is aborted if ESLint reports unfixable errors after auto-fix.

#### `pre-push` — full suite

Runs against the full branch before any push reaches the remote:

```bash
pnpm lint        # full lint pass (catches cross-file issues lint-staged misses)
pnpm tsc --noEmit  # type-check
pnpm test --run    # Vitest unit + integration suite
```

A push that breaks types, lint, or tests never leaves the machine.

#### Bypassing hooks

`--no-verify` is available for emergencies (e.g., WIP stash commits to a personal branch). It should not appear in any shared workflow or CI script.

### CI mirror

The same three commands (`lint`, `tsc --noEmit`, `test --run`) run in CI on every pull request. Local hooks and CI run identical checks, so a passing push means a passing pipeline.

---

## Future: monorepo & backend

```
orthools/
├── apps/
│   ├── web/            ← current src/ moves here unchanged
│   └── api/            ← NestJS or equivalent
└── libs/
    ├── domain/         ← extracted from apps/web — zero rewrite, no framework imports
    └── shared-types/   ← DTOs shared between web and api
```

`domain/` and `application/` extract cleanly because they have no framework imports today. `persistence/api/` adapters become the HTTP client layer. Auth slots into `persistence/` (token store) and `presentation/auth/` (guards, interceptors) without touching domain or application.

---

## Phase summary

| Phase | Persistence | DI binding | What changes |
|-------|-------------|------------|--------------|
| 1 — in-memory | `InMemory*Repository` | Context value = `InMemory…` | — |
| 2 — backend | `Api*Repository` | Context value = `Api…` | `*.api.repository.ts` per module |
| 3 — auth | same | add interceptor + guards | `auth.interceptor.ts`, `AuthGuard.tsx` |
| 4 — monorepo | same | extract `domain/` to `libs/` | workspace config only |

Domain, application, and all presentation components are untouched across every phase.
