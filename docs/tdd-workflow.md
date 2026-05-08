# TDD Workflow — Orthools

## What TDD is (and what it is not)

TDD — Test-Driven Development — is a discipline where you write the test *before* you write the code that makes it pass. The test is not a safety net added after the fact: it is the design tool. You describe what the code should do, watch it fail because nothing exists yet, write the minimum code to make it pass, then clean up.

TDD is not about having 100% coverage. It is about **designing in small, verified increments** so that every piece of code you write has a clear reason to exist and a proof that it works.

---

## The cycle: Red → Green → Refactor

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   RED          Write a test that describes the next        │
│                behaviour. Run it. It must fail.            │
│                A test that passes before code exists       │
│                is not testing anything.                    │
│                                                             │
│   GREEN        Write the minimum code to make the          │
│                test pass. Not the best code — the          │
│                simplest code. Resist the urge to           │
│                generalise yet.                             │
│                                                             │
│   REFACTOR     Now improve the code. Extract duplication,  │
│                rename things, restructure. The test stays  │
│                green throughout. If it goes red during     │
│                refactor, you broke something.              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

Each cycle is short — minutes, not hours. If a cycle takes more than 15 minutes, the step you chose was too large. Split it.

---

## Before you write any code: design by intention

The first thing you write is a test that *does not compile yet*. That is intentional. Writing the call before the implementation forces you to think about the interface from the caller's perspective:

```typescript
// You write this first — nothing exists yet
const color = new HexColor('#FF0000');
expect(color.value).toBe('#FF0000');
```

This tells you: `HexColor` is a class, it takes a string, it exposes a `.value` property. Only then do you write the class. The test drives the design.

---

## Test structure: Arrange – Act – Assert

Every test has three parts. Keep them visually separated.

```typescript
it('returns only enabled instructions', async () => {
  // Arrange — set up the world
  const repo = new InMemoryInstructionRepository();
  repo.seed([
    { id: '1', isEnabled: true,  name: 'Lettre B', ... },
    { id: '2', isEnabled: false, name: 'Lettre K', ... },
  ]);
  const useCase = new ListInstructionsUseCase(repo);

  // Act — perform the operation under test
  const result = await useCase.execute();

  // Assert — verify the outcome
  expect(result).toHaveLength(1);
  expect(result[0].id).toBe('1');
});
```

Do not mix these phases. If your "Act" section requires setup, move that setup into "Arrange". If your "Assert" section calls more methods, that is two tests.

---

## Test naming: describe behaviour, not implementation

A test name is documentation. It should read as a sentence describing what the system does.

```
// Bad — describes what the code does internally
it('calls findEnabled on the repository')

// Bad — too vague
it('works correctly')

// Good — describes a user-observable behaviour
it('excludes disabled instructions from the returned list')
it('rejects a hex colour string without a leading #')
it('generates exactly the requested number of slides')
```

Use `describe` blocks to group tests by subject and `it` (or `test`) to state the specific behaviour:

```typescript
describe('HexColor', () => {
  describe('construction', () => {
    it('accepts a valid 6-digit hex string', () => { ... });
    it('rejects a string without a leading #', () => { ... });
    it('rejects a 3-digit shorthand', () => { ... });
  });
});
```

When a test fails in CI, the output reads: `HexColor > construction > rejects a string without a leading #`. That is enough to understand the failure without opening the file.

---

## One behaviour per test

Each `it` block tests one thing. Not one assertion — one *behaviour*. A behaviour may require a few assertions to verify:

```typescript
it('generates the correct action/non-action slide distribution', async () => {
  // This one behaviour requires checking multiple properties
  const result = await useCase.execute({ slideCount: 10, percentageActionExpected: 40 });

  const actionSlides    = result.slides.filter(s => s.isActionExpected);
  const nonActionSlides = result.slides.filter(s => !s.isActionExpected);

  expect(result.slides).toHaveLength(10);
  expect(actionSlides).toHaveLength(4);    // floor(10 × 40%)
  expect(nonActionSlides).toHaveLength(6);
});
```

This is fine: all three assertions describe the same behaviour (distribution). What is not fine is asserting two *different* things in one `it`, because when it fails you cannot tell which thing broke.

---

## DAMP over DRY in tests

Application code should be DRY (Don't Repeat Yourself). Test code should be DAMP (Descriptive And Meaningful Phrases). A test should be readable in isolation without chasing through abstractions.

```typescript
// DRY — requires reading buildInstruction() to understand the test
it('...', () => {
  const instruction = buildInstruction({ isEnabled: false });
  ...
});

// DAMP — the data is right there; the intent is obvious
it('...', () => {
  const instruction: Instruction = {
    id: '1', type: 'color', name: 'Lettre B',
    instruction: 'Dire un mot commençant par B',
    isEnabled: false,    // ← this is the thing that matters for this test
    categoryIds: [], isDefault: false,
  };
  ...
});
```

The exception: if you genuinely need the same large fixture in many tests, extract a `buildXxx()` factory in a `*.fixtures.ts` file — but keep it simple. Factories with many optional overrides become traps.

---

## The layer order

Follow the dependency rule inward → outward. Start with the layer that depends on nothing, then work outward. Each layer's tests use real implementations of the inner layers (or the in-memory fakes that already exist).

```
1. Domain         — entities, value objects, domain services   (pure TS, no setup)
2. Application    — use cases                                  (plain TS + fake repo)
3. Persistence    — in-memory repositories, contract tests     (plain TS)
4. Presentation   — hooks                                      (renderHook + Context)
                  — UI components                              (render + assertions)
                  — pages                                      (render + Context stack)
                  — guards                                     (render + Navigate check)
```

---

## Layer 1 — Domain

### What to test

- **Value objects**: every invariant they enforce. Valid construction, rejection of invalid inputs.
- **Entities**: business rules. State transitions. Methods that enforce domain constraints.
- **Domain services**: the algorithm. Given this input, the output satisfies these properties.

### How to write domain tests

No test setup at all. Import the class, construct it, assert. The test file looks like:

```typescript
// domain/lexical-evocation/instruction/hex-color.vo.spec.ts

import { HexColor } from './hex-color.vo';

describe('HexColor', () => {
  it('stores a valid #RRGGBB string', () => {
    const color = new HexColor('#1A2B3C');
    expect(color.value).toBe('#1A2B3C');
  });

  it('normalises to uppercase', () => {
    const color = new HexColor('#ff0000');
    expect(color.value).toBe('#FF0000');
  });

  it('throws when the # prefix is missing', () => {
    expect(() => new HexColor('FF0000')).toThrow();
  });

  it('throws for a 3-digit shorthand', () => {
    expect(() => new HexColor('#F00')).toThrow();
  });
});
```

### Domain service example

```typescript
// domain/lexical-evocation/exercise/exercise-generator.domain-service.spec.ts

import { ExerciseGeneratorService } from './exercise-generator.domain-service';
import { ExerciseGeneratorConfig }  from './exercise-generator-config.vo';
import { makeInstruction }          from './__fixtures__/instruction.fixtures';

describe('ExerciseGeneratorService', () => {
  const service = new ExerciseGeneratorService();

  it('generates exactly the requested number of slides', () => {
    const config = makeConfig({ slideCount: 8, percentageActionExpected: 50 });
    const slides = service.generate(config);
    expect(slides).toHaveLength(8);
  });

  it('creates the correct number of action slides (floor)', () => {
    const config = makeConfig({ slideCount: 10, percentageActionExpected: 33 });
    const slides = service.generate(config);
    const actionSlides = slides.filter(s => s.isActionExpected);
    expect(actionSlides).toHaveLength(3); // floor(10 × 0.33)
  });

  it('assigns each instruction colour to at most one action slide per cycle', () => {
    const instructions = [makeInstruction(), makeInstruction(), makeInstruction()];
    const config = makeConfig({ slideCount: 3, percentageActionExpected: 100, selectedInstructions: instructions });
    const slides = service.generate(config);
    const actionSlides = slides.filter(s => s.isActionExpected);
    const colors = actionSlides.map(s => s.color);
    expect(new Set(colors).size).toBe(3); // no duplicates in one cycle
  });

  it('does not share colours between action and non-action slides', () => {
    const config = makeConfig({ slideCount: 10, percentageActionExpected: 50 });
    const slides = service.generate(config);
    const actionColors    = new Set(slides.filter(s =>  s.isActionExpected).map(s => s.color));
    const nonActionColors =          slides.filter(s => !s.isActionExpected).map(s => s.color);
    nonActionColors.forEach(c => expect(actionColors.has(c)).toBe(false));
  });
});

function makeConfig(overrides: Partial<ExerciseGeneratorConfig>): ExerciseGeneratorConfig {
  return {
    slideCount: 10,
    percentageActionExpected: 50,
    isAutoSlide: false,
    slideDelay: null,
    selectedInstructions: [makeInstruction(), makeInstruction()],
    ...overrides,
  };
}
```

### Domain layer specifics

- **No `async`** unless the domain model itself requires it (it rarely does at this layer).
- **No imports from `application/`, `persistence/`, or `presentation/`**. If you find yourself needing one, the test belongs in a different layer.
- Tests run in milliseconds. If a domain test is slow, something is wrong.

---

## Layer 2 — Application

### What to test

Use cases orchestrate: they call ports, apply domain logic, and return results. Test that the orchestration is correct:
- Does the use case read from the correct port?
- Does it pass the right data to the domain service?
- Does it persist the result when it should?
- Does it reject invalid input?

You do **not** test that the repository implementation works — that is the persistence layer's job. Here you trust the fake.

### How to write application tests

Construct the use case with a real `InMemoryInstructionRepository` passed by constructor. Seed the repo, call the use case, assert the output.

```typescript
// application/lexical-evocation/instruction/list-instructions.use-case.spec.ts

import { ListInstructionsUseCase }       from './list-instructions.use-case';
import { InMemoryInstructionRepository } from '../../../persistence/lexical-evocation/instruction.in-memory.repository';

describe('ListInstructionsUseCase', () => {
  it('returns only enabled instructions', async () => {
    const repo = new InMemoryInstructionRepository([
      { id: '1', isEnabled: true,  name: 'Lettre B', instruction: '...', type: 'color', categoryIds: [], isDefault: true },
      { id: '2', isEnabled: false, name: 'Lettre K', instruction: '...', type: 'color', categoryIds: [], isDefault: true },
    ]);
    const useCase = new ListInstructionsUseCase(repo);

    const result = await useCase.execute();

    expect(result).toHaveLength(1);
    expect(result[0].name).toBe('Lettre B');
  });

  it('returns an empty list when no instructions are enabled', async () => {
    const repo = new InMemoryInstructionRepository([
      { id: '1', isEnabled: false, name: 'Lettre B', instruction: '...', type: 'color', categoryIds: [], isDefault: true },
    ]);
    const useCase = new ListInstructionsUseCase(repo);

    const result = await useCase.execute();

    expect(result).toHaveLength(0);
  });
});
```

### Application layer specifics

- **No React, no testing-library, no `render()`**. Pure TypeScript.
- Use the real `InMemory*Repository` — not a jest mock. The fake is already a proper test double.
- Each test constructs fresh instances. No shared state between tests.
- `async/await` is normal here because ports return Promises. Use Vitest's built-in async support.

---

## Layer 3 — Persistence

### What to test

The in-memory repository is a first-class implementation — not a mock. It must behave exactly like the future API adapter. Test this via **contract tests**: a shared spec file that describes the required behaviour of any `IInstructionRepository`, which every adapter must pass.

### The contract pattern

```typescript
// persistence/lexical-evocation/instruction.repository.contract.ts
// No .spec suffix — this is not a test file itself; it is a shared test suite.

import type { IInstructionRepository } from '../../domain/lexical-evocation/instruction/instruction.repository';

export function runInstructionRepositoryContract(
  createRepo: () => IInstructionRepository
) {
  describe('IInstructionRepository contract', () => {
    it('findAll returns all non-deleted records', async () => {
      const repo = createRepo();
      await repo.save({ id: '1', isEnabled: true,  name: 'A', instruction: '', type: 'color', categoryIds: [], isDefault: false });
      await repo.save({ id: '2', isEnabled: false, name: 'B', instruction: '', type: 'color', categoryIds: [], isDefault: false });

      const result = await repo.findAll();

      expect(result).toHaveLength(2);
    });

    it('findEnabled excludes disabled instructions', async () => {
      const repo = createRepo();
      await repo.save({ id: '1', isEnabled: true,  name: 'A', instruction: '', type: 'color', categoryIds: [], isDefault: false });
      await repo.save({ id: '2', isEnabled: false, name: 'B', instruction: '', type: 'color', categoryIds: [], isDefault: false });

      const result = await repo.findEnabled();

      expect(result).toHaveLength(1);
      expect(result[0].id).toBe('1');
    });

    it('delete soft-deletes: record excluded from findAll but still in storage', async () => {
      const repo = createRepo();
      await repo.save({ id: '1', isEnabled: true, name: 'A', instruction: '', type: 'color', categoryIds: [], isDefault: false });

      await repo.delete('1');

      expect(await repo.findAll()).toHaveLength(0);
    });

    it('delete rejects system-default records', async () => {
      const repo = createRepo();
      await repo.save({ id: '1', isEnabled: true, name: 'A', instruction: '', type: 'color', categoryIds: [], isDefault: true });

      await expect(repo.delete('1')).rejects.toThrow();
    });

    it('save with an existing id updates the record', async () => {
      const repo = createRepo();
      await repo.save({ id: '1', isEnabled: true,  name: 'Original', instruction: '', type: 'color', categoryIds: [], isDefault: false });
      await repo.save({ id: '1', isEnabled: false, name: 'Updated',  instruction: '', type: 'color', categoryIds: [], isDefault: false });

      const result = await repo.findAll();

      expect(result).toHaveLength(1);
      expect(result[0].name).toBe('Updated');
    });
  });
}
```

Each adapter runs the contract suite in its own spec file:

```typescript
// persistence/lexical-evocation/instruction.in-memory.repository.spec.ts

import { InMemoryInstructionRepository }   from './instruction.in-memory.repository';
import { runInstructionRepositoryContract } from './instruction.repository.contract';

runInstructionRepositoryContract(() => new InMemoryInstructionRepository());
```

When the API adapter is built in phase 2:

```typescript
// persistence/lexical-evocation/api/instruction.api.repository.spec.ts

import { ApiInstructionRepository }        from './instruction.api.repository';
import { runInstructionRepositoryContract } from '../instruction.repository.contract';
import { setupServer }                     from 'msw/node';
import { http, HttpResponse }              from 'msw';

const server = setupServer(/* msw handlers */);
beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

runInstructionRepositoryContract(() => new ApiInstructionRepository('http://localhost'));
```

Both adapters must pass the same contract. If they do, the production swap is guaranteed safe.

### Persistence layer specifics

- The contract file has **no `.spec` suffix** — it is imported, not discovered by the test runner.
- Every new adapter gets its own `.spec.ts` file that calls `runXxxContract()`.
- The in-memory repository seed data (the five default instructions) should be seeded by the test itself, not relied upon as hidden state.

---

## Layer 4 — Presentation: Hooks

### What to test

A hook is the React equivalent of the facade. It wires use cases to component state. Test that:
- The correct initial state is returned.
- Calling an action updates the state correctly.
- Error states surface when the use case throws.

### How to write hook tests

Use `renderHook()` from React Testing Library with a Context wrapper that provides fake repositories.

```typescript
// presentation/lexical-evocation/generator/use-generator.hook.spec.ts

import { renderHook, act } from '@testing-library/react';
import { useGenerator }               from './use-generator.hook';
import { InstructionRepositoryContext } from '../../_di/instruction-repository.context';
import { InMemoryInstructionRepository } from '../../../persistence/lexical-evocation/instruction.in-memory.repository';

function makeWrapper(repo = new InMemoryInstructionRepository()) {
  return function Wrapper({ children }: { children: React.ReactNode }) {
    return (
      <InstructionRepositoryContext.Provider value={repo}>
        {children}
      </InstructionRepositoryContext.Provider>
    );
  };
}

describe('useGenerator', () => {
  it('starts in configuring step with no exercise', () => {
    const { result } = renderHook(() => useGenerator(), { wrapper: makeWrapper() });

    expect(result.current.step).toBe('configuring');
    expect(result.current.exercise).toBeNull();
  });

  it('advances to previewing after generateExercise resolves', async () => {
    const repo = new InMemoryInstructionRepository(); // seeded defaults
    const { result } = renderHook(() => useGenerator(), { wrapper: makeWrapper(repo) });

    await act(async () => {
      await result.current.generateExercise({
        slideCount: 5,
        percentageActionExpected: 50,
        isAutoSlide: false,
        slideDelay: null,
        selectedInstructions: [],
      });
    });

    expect(result.current.step).toBe('previewing');
    expect(result.current.exercise).not.toBeNull();
    expect(result.current.exercise!.slides).toHaveLength(5);
  });
});
```

### Hook specifics

- **`act()`** is required around any call that triggers a state update. Vitest will warn you if you forget. For async actions, use `await act(async () => { ... })`.
- **Never test implementation details.** You do not care that the hook calls `GenerateExerciseUseCase` internally. You care that calling `generateExercise()` updates `step` and `exercise`.
- The `wrapper` function must be a component (capitalised, returns JSX). That is a React requirement.
- Keep Context provision in the wrapper, not in individual tests — the tests should only vary the inputs to the hook and the assertions.

---

## Layer 4 — Presentation: UI Components

### What to test

UI components (`presentation/ui/`) have no domain knowledge. Test what the user sees and can interact with:
- Given these props, this text/element appears.
- Given these props, this style/class is applied.
- Given this interaction, this callback is called.
- The component passes the axe accessibility check.

You do **not** test implementation details: which state variables exist inside the component, or which child components are rendered internally.

### How to write component tests

Use `render()` from React Testing Library and query the DOM using semantic queries (`getByRole`, `getByLabelText`, `getByText`). Prefer `getByRole` above all others — it tests what is accessible.

```typescript
// presentation/ui/color-dot/color-dot.spec.tsx

import { render, screen } from '@testing-library/react';
import { axe }            from 'jest-axe';
import { ColorDot }       from './color-dot';

describe('ColorDot', () => {
  it('renders a visible circle with the given colour', () => {
    const { container } = render(<ColorDot color="#FF0000" size="normal" />);
    const dot = container.firstChild as HTMLElement;

    expect(dot).toBeInTheDocument();
    expect(dot.style.backgroundColor).toContain('FF0000');
  });

  it('applies the large variant class in execution mode', () => {
    render(<ColorDot color="#FF0000" size="large" />);
    const dot = screen.getByTestId('color-dot');

    expect(dot.className).toMatch(/large/);
  });

  it('has an accessible label describing the colour', () => {
    render(<ColorDot color="#FF0000" size="normal" aria-label="Red circle" />);

    expect(screen.getByRole('img', { name: 'Red circle' })).toBeInTheDocument();
  });

  it('passes the axe accessibility check', async () => {
    const { container } = render(<ColorDot color="#FF0000" size="normal" aria-label="Red circle" />);
    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });
});
```

### Query priority

Always query by what a real user would use to find an element:

| Priority | Query | When to use |
|----------|-------|-------------|
| 1st | `getByRole` | Buttons, inputs, headings, lists — always try this first |
| 2nd | `getByLabelText` | Form fields associated with a label |
| 3rd | `getByPlaceholderText` | Input placeholders (last resort for inputs) |
| 4th | `getByText` | Static text content |
| 5th | `getByTestId` | Only when nothing semantic is available |

### User interactions

Use `@testing-library/user-event` rather than `fireEvent`. It simulates real browser behaviour (focus, blur, keyboard events in sequence):

```typescript
import userEvent from '@testing-library/user-event';

it('calls onSelect when an option is chosen from the dropdown', async () => {
  const user = userEvent.setup();
  const onSelect = vi.fn();

  render(<InstructionDropdown options={seededInstructions} onSelect={onSelect} />);

  await user.click(screen.getByRole('combobox', { name: 'Instruction' }));
  await user.click(screen.getByRole('option', { name: 'Lettre B' }));

  expect(onSelect).toHaveBeenCalledWith(expect.objectContaining({ name: 'Lettre B' }));
});
```

### UI component specifics

- **No domain imports in `ui/` tests.** If you need to pass an `Instruction` type to test a `ui/` component, the component is not truly domain-agnostic. Fix the component first.
- Always include at least one axe test per `ui/` component.
- For components with many visual states, Storybook stories cover those visually; unit tests cover the interactive logic.

---

## Layer 4 — Presentation: Pages (integration tests)

### What to test

Pages are the integration point: they assemble hooks, domain components, and UI components into a full feature. Test the full user flow at the page level with in-memory Context providers:
- Does the page render the right content given in-memory data?
- When the user performs an action, does the UI respond correctly?
- Does navigation happen when expected?

### How to write page tests

Render the full page tree inside the real Context stack (in-memory repositories). Do not mock the hook — let the full hook → use case → repository chain run.

```typescript
// presentation/lexical-evocation/generator/configurator/configurator.page.spec.tsx

import { render, screen, within } from '@testing-library/react';
import userEvent                  from '@testing-library/user-event';
import { MemoryRouter }           from 'react-router-dom';
import { ConfiguratorPage }       from './configurator.page';
import { InstructionRepositoryContext } from '../../../_di/instruction-repository.context';
import { InMemoryInstructionRepository } from '../../../../persistence/lexical-evocation/instruction.in-memory.repository';

function renderConfigurator() {
  const repo = new InMemoryInstructionRepository(); // uses default seed
  return render(
    <MemoryRouter initialEntries={['/lexical-evocation/generator']}>
      <InstructionRepositoryContext.Provider value={repo}>
        <ConfiguratorPage />
      </InstructionRepositoryContext.Provider>
    </MemoryRouter>
  );
}

describe('ConfiguratorPage', () => {
  it('shows the seeded instructions in the dropdown', async () => {
    const user = userEvent.setup();
    renderConfigurator();

    await user.click(screen.getByRole('combobox', { name: /add instruction/i }));

    expect(screen.getByRole('option', { name: 'Lettre B' })).toBeInTheDocument();
    expect(screen.getByRole('option', { name: 'Légume' })).toBeInTheDocument();
  });

  it('shows the preview after Generate is clicked', async () => {
    const user = userEvent.setup();
    renderConfigurator();

    await user.click(screen.getByRole('combobox', { name: /add instruction/i }));
    await user.click(screen.getByRole('option', { name: 'Lettre B' }));
    await user.click(screen.getByRole('button', { name: /generate/i }));

    expect(await screen.findByText(/preview/i)).toBeInTheDocument();
  });
});
```

### Page specifics

- **Wrap in `MemoryRouter`** — pages use routing hooks (`useNavigate`, `useParams`). `MemoryRouter` from `react-router-dom` provides a routing context without a real browser.
- **Use `findBy*` for async content** — content that appears after a state update requires `findByRole` / `findByText` which return Promises and wait automatically. Use `getBy*` only for content that is synchronously present.
- **Do not mock the hook.** The point of page tests is to verify the integration. Mocking the hook leaves a gap between the test and reality.

---

## Layer 4 — Presentation: Guards

### What to test

Guards are simple conditional components. Test two branches:
- When the condition is **not met**, a redirect is rendered.
- When the condition **is met**, the children are rendered.

```typescript
// presentation/auth/guards/exercise-ready.guard.spec.tsx

import { render, screen } from '@testing-library/react';
import { MemoryRouter, Routes, Route } from 'react-router-dom';
import { ExerciseReadyGuard } from './exercise-ready.guard';
import { GeneratorContext }   from '../../lexical-evocation/generator/generator.context';

function renderGuard(exercise: ExerciseSet | null) {
  return render(
    <MemoryRouter initialEntries={['/lexical-evocation/generator/execute']}>
      <GeneratorContext.Provider value={{ exercise, step: 'previewing', generateExercise: vi.fn() }}>
        <Routes>
          <Route
            path="/lexical-evocation/generator/execute"
            element={
              <ExerciseReadyGuard>
                <div>Execution page</div>
              </ExerciseReadyGuard>
            }
          />
          <Route
            path="/lexical-evocation/generator"
            element={<div>Configurator page</div>}
          />
        </Routes>
      </GeneratorContext.Provider>
    </MemoryRouter>
  );
}

describe('ExerciseReadyGuard', () => {
  it('redirects to /generator when no exercise is in state', () => {
    renderGuard(null);
    expect(screen.getByText('Configurator page')).toBeInTheDocument();
  });

  it('renders children when an exercise is in state', () => {
    renderGuard(makeExercise());
    expect(screen.getByText('Execution page')).toBeInTheDocument();
  });
});
```

---

## Best practices summary

### Do

- Write the test first. Always. Even when it feels awkward.
- Run the test immediately after writing it — verify it fails for the right reason. A test that fails with `TypeError: HexColor is not a constructor` is not yet testing anything useful.
- Keep tests fast. Domain tests must be under 1 ms. A slow test is a signal that the design is wrong.
- Delete tests that no longer describe real behaviour. Dead tests create noise and false confidence.
- Name tests as sentences. Read the test list aloud — it should sound like a specification.

### Do not

- **Do not mock things you own.** Mocking `InMemoryInstructionRepository` in an application test defeats the purpose. Use the real fake.
- **Do not test private methods.** If you feel the urge, extract the logic into a public domain service and test that.
- **Do not assert on implementation details.** A test that breaks when you rename an internal variable is not a safety net — it is friction.
- **Do not share mutable state between tests.** Construct fresh instances in each `it` block. Shared mutable state causes tests to affect each other in unpredictable ways.
- **Do not write tests after the fact.** Tests written after the code was working tend to confirm what the code does, not what it should do. They miss the edge cases the implementation already silently handles wrong.

### When a test is hard to write

Difficulty writing a test is almost always a design signal:

| Difficulty | Likely cause | Solution |
|------------|-------------|----------|
| Too much setup required | Class has too many dependencies | Split responsibilities |
| Cannot test without mocking | Business logic is tangled with I/O | Extract logic into a pure domain service |
| Cannot reach the code path | Logic is buried inside a framework callback | Extract it into a plain function |
| Test breaks on refactor | Testing implementation, not behaviour | Rewrite the test to assert on outputs only |

---

## The feedback loop in practice

During active development, keep Vitest running in watch mode:

```
pnpm vitest
```

The cycle is:
1. Write a failing test → terminal shows red.
2. Write the minimum code → terminal turns green.
3. Refactor → terminal stays green.
4. Commit.

The goal is to never be more than one small step away from a passing test suite. If you have been writing code for 10 minutes and have not seen a green test, stop and write a simpler test first.
