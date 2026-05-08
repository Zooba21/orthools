# Module — Lexical Evocation (Évocation Lexicale)

## Purpose

This module lets speech therapists create and run lexical evocation exercises. The therapist defines a set of cues (visual colour circles and/or auditory sounds) each associated with an instruction (e.g. "say a word starting with K", "name a vegetable"). The module generates a sequence of slides each showing a colour and optionally playing a sound. The patient responds — or stays silent — depending on whether the cue matches a defined instruction. The therapist observes and scores results manually; the platform records nothing about patient responses.

---

## MVP scope

The first build is standalone, no backend, no authentication.

| Feature | MVP | Deferred |
|---------|-----|----------|
| Color-type instructions | ✅ seeded + inline input | — |
| Audio-type instructions | — | ✅ phase 2 |
| Category management UI | — | ✅ phase 2 |
| Instruction management UI | — | ✅ phase 2 |
| Saved exercises | — | ✅ phase 2 |
| Exercise settings (defaults) | — | ✅ phase 2 |
| Module onboarding / common trunk | — | ✅ with auth |
| Patient accessibility constraints | — | ✅ phase 2 |

**MVP data flow:** a small set of seeded color instructions lives in the in-memory repository. The therapist can also type additional instructions inline in the generator form — these are transient (not persisted, lost on page reload). The generator uses both seeded and inline instructions to produce slides. No sounds, no saved exercises.

---

## Domain model

### Entities

#### `Instruction` (Consigne)

A reusable cue, authored once and available in any exercise.

| Field | Type | Notes |
|-------|------|-------|
| `id` | string | |
| `type` | `'color' \| 'audio'` | Audio deferred in MVP |
| `name` | string | Used for search / display |
| `instruction` | string | The actual prompt shown to the therapist (e.g. "Dire un mot commençant par B") |
| `isEnabled` | boolean | Disabled instructions are excluded from generation |
| `categoryIds` | string[] | Associated categories |
| `isDefault` | boolean | `true` = system-seeded, not deletable by the user |

> Note: colour is **not** stored on `Instruction`. It is generated at exercise time based on accessibility constraints and stored on the slide snapshot. An instruction only says *what to do*, not *what colour to show*.

#### `Category`

Hierarchical classification for both instructions and audio files.

| Field | Type | Notes |
|-------|------|-------|
| `id` | string | |
| `parentId` | string \| null | Null = root category |
| `name` | string | Display name |
| `slug` | string | Normalised name for search (e.g. `objet_present_dans_la_cuisine`) |
| `isDefault` | boolean | System category, not deletable |
| `childIds` | string[] | Populated when loading the tree |

#### `AudioFile` *(deferred — phase 2)*

| Field | Type | Notes |
|-------|------|-------|
| `id` | string | |
| `name` | string | Display name (e.g. "Son de cheval") |
| `filename` | string | File name on disk |
| `filePath` | string | Full storage path (e.g. `audio/cheval.mp3`) |
| `isEnabled` | boolean | |
| `categoryIds` | string[] | Categories representing valid responses to this sound |
| `isDefault` | boolean | Pre-loaded by the platform, not deletable |

Storage limit: 1 GB per account in the MVP.

#### `ExerciseSet` *(persistence deferred — in-memory only in MVP)*

A saved exercise. Parameters are **frozen at save time** (snapshot pattern) — changing the original instruction later does not affect a saved exercise. Audio files linked to a saved exercise cannot be deleted while the exercise exists.

| Field | Type | Notes |
|-------|------|-------|
| `id` | string | |
| `name` | string | |
| `isAutoSlide` | boolean | Auto advance vs manual |
| `slideDelay` | number \| null | Milliseconds; null if manual |
| `slideCount` | number | |
| `percentageActionExpected` | number | 0–100 |
| `instructions` | `ExerciseSetInstruction[]` | Snapshot of selected instructions |
| `slides` | `ExerciseSetSlide[]` | Generated slides |

#### `ExerciseSetInstruction`

Immutable snapshot of an instruction as it existed when the exercise was created.

| Field | Type | Notes |
|-------|------|-------|
| `id` | string | |
| `type` | `'color' \| 'audio'` | |
| `name` | string | |
| `instruction` | string | |
| `color` | string \| null | Hex color captured at generation time; null for audio type |

#### `ExerciseSetSlide`

One slide in the exercise.

| Field | Type | Notes |
|-------|------|-------|
| `id` | string | |
| `color` | string | Hex color displayed on screen |
| `audioId` | string \| null | Audio file to play; null = silent slide |
| `order` | number | 0-based position |
| `isActionExpected` | boolean | Derived: true if this slide's colour or sound matches a defined instruction |

### Value objects

| Name | Wraps | Invariant enforced |
|------|-------|--------------------|
| `InstructionType` | `'color' \| 'audio'` | Rejects any other string |
| `HexColor` | string | Must be a valid `#RRGGBB` value |
| `ContrastRatio` | number | WCAG minimum 4.5:1 for normal text, 3:1 for large |
| `ExerciseGeneratorConfig` | — | Input to the generation algorithm (see below) |

#### `ExerciseGeneratorConfig`

```
slideCount               number   (default: 10)
percentageActionExpected number   (default: 50, 0–100)
isAutoSlide              boolean  (default: false)
slideDelay               number   (milliseconds; default: 5000, only if isAutoSlide)
selectedInstructions     Instruction[]   (mixed seeded + inline)
```

### Domain services

#### `ExerciseGeneratorService`

Pure algorithm — no ports, no I/O. Receives `ExerciseGeneratorConfig` and returns `ExerciseSetSlide[]`.

Logic:
1. Determine how many slides need an action (`floor(slideCount × percentageActionExpected / 100)`).
2. For action slides: assign one instruction per slide (cycling through selected instructions if slideCount > instruction count).
3. For non-action slides: generate a colour that does **not** match any active instruction colour.
4. Shuffle all slides.
5. Return the ordered array.

#### `ColorGeneratorService`

Generates accessible, visually distinct hex colours. In MVP: generates random colours with sufficient WCAG contrast against white. In a later phase: accepts a list of patient-specific colour restrictions (e.g. red-green colour blindness) and excludes those ranges.

---

## Ports (repository interfaces)

```
IInstructionRepository
  findAll(): Promise<Instruction[]>
  findEnabled(): Promise<Instruction[]>
  findById(id: string): Promise<Instruction | null>
  save(instruction: Instruction): Promise<void>
  delete(id: string): Promise<void>

ICategoryRepository
  findAll(): Promise<Category[]>
  findTree(): Promise<Category[]>          # Returns root nodes with children populated
  findById(id: string): Promise<Category | null>
  save(category: Category): Promise<void>
  delete(id: string): Promise<void>

IAudioFileRepository  (deferred)
  findAll(): Promise<AudioFile[]>
  findEnabled(): Promise<AudioFile[]>
  findById(id: string): Promise<AudioFile | null>
  save(file: AudioFile): Promise<void>
  delete(id: string): Promise<void>        # Blocked if referenced by a saved exercise

IExerciseSetRepository  (persistence deferred in MVP)
  findAll(): Promise<ExerciseSet[]>
  findById(id: string): Promise<ExerciseSet | null>
  save(exercise: ExerciseSet): Promise<void>
  delete(id: string): Promise<void>
```

---

## Application layer — use cases

### MVP (active)

| Use case | Input | Output | Notes |
|----------|-------|--------|-------|
| `ListInstructionsUseCase` | — | `Instruction[]` | Returns enabled instructions only |
| `GenerateExerciseUseCase` | `ExerciseGeneratorConfig` | `ExerciseSet` | Calls domain services; does not persist |

### Phase 2 (deferred)

| Use case | Notes |
|----------|-------|
| `CreateInstructionUseCase` | Validates uniqueness, assigns categories |
| `UpdateInstructionUseCase` | Cannot change type after creation |
| `DeleteInstructionUseCase` | Soft delete (sets `deletedAt`) |
| `ListCategoriesUseCase` | Returns full tree |
| `CreateCategoryUseCase` | Validates parent exists |
| `UpdateCategoryUseCase` | |
| `DeleteCategoryUseCase` | Cascades soft-delete to children |
| `UploadAudioFileUseCase` | Checks storage quota before saving |
| `SaveExerciseUseCase` | Snapshot pattern — freezes instructions and slides |
| `ListExercisesUseCase` | |
| `DeleteExerciseUseCase` | Releases audio file locks |

---

## Persistence layer

### In-memory (phase 1)

Three repositories, all seeded with fixture data at startup.

**Seeded instructions (color type, MVP examples):**

| Name | Instruction |
|------|-------------|
| Lettre B | "Dire un mot commençant par la lettre B" |
| Lettre K | "Dire un mot commençant par la lettre K" |
| Légume | "Citer un légume" |
| Animal | "Citer un animal" |
| Objet cuisine | "Nommer un objet de cuisine" |

**Seeded categories (MVP examples):**
```
Phonologique
  └── Préfixe
  └── Suffixe
Sémantique
  └── Maison
      └── Cuisine
      └── Chambre
  └── Nature
      └── Animaux
```

### Soft delete

All repositories implement the soft-delete pattern: `delete()` sets a `deletedAt` timestamp rather than removing the record. `findAll()` and `findEnabled()` filter out soft-deleted records. System-default records (`isDefault: true`) reject delete calls entirely.

### API adapter (phase 2)

One `*.api.repository.ts` file per entity, implementing the same interface. Both adapters run the same contract test suite — see the testing strategy in `architecture.md`.

---

## Presentation layer

### Route structure

```
/lexical-evocation                      → home (module description)
/lexical-evocation/generator            → generator wizard (configure + preview)
/lexical-evocation/generator/execute    → execution mode (guard: exercise-ready)
/lexical-evocation/instructions         → instruction list          (deferred)
/lexical-evocation/instructions/create  → create instruction        (deferred)
/lexical-evocation/instructions/:id/edit→ edit instruction          (deferred)
/lexical-evocation/categories           → category tree management  (deferred)
/lexical-evocation/exercises            → saved exercises list      (deferred)
/lexical-evocation/settings             → module settings           (deferred)
```

### Folder structure

```
presentation/lexical-evocation/
│
├── home/
│   └── home.page.tsx                    # Module landing: description, entry CTA
│
├── generator/                           # Core MVP flow
│   │
│   ├── configurator/                    # Step 1: define instructions + parameters
│   │   └── configurator.page.tsx
│   │
│   ├── preview/                         # Step 2: review generated slides
│   │   └── preview.page.tsx
│   │
│   ├── execution/                       # Step 3: run with patient (full screen, no nav)
│   │   └── execution.page.tsx
│   │
│   └── use-generator.hook.ts            # Drives: GenerateExercise + ListInstructions
│
├── instructions/                        # Deferred
│   ├── list/
│   ├── create/
│   ├── edit/
│   └── use-instructions.hook.ts
│
├── categories/                          # Deferred
│   ├── list/
│   ├── create/
│   ├── edit/
│   └── use-categories.hook.ts
│
├── exercises/                           # Deferred
│   ├── list/
│   └── use-exercises.hook.ts
│
└── components/                          # Module-scoped shared components
    ├── color-dot/                        # The coloured circle visual cue
    │   └── color-dot.tsx
    ├── instruction-row/                  # One row in the instruction repeater
    │   └── instruction-row.tsx
    ├── slide-card/                       # One slide in the preview grid
    │   └── slide-card.tsx
    └── category-selector/               # Multi-select category dropdown (deferred)
        └── category-selector.tsx
```

### Generator flow detail

The generator is a two-step wizard on a single route (`/lexical-evocation/generator`). Internal state managed by `use-generator.hook.ts` drives which step is shown via a `step` state value. No route change between step 1 (configurator) and step 2 (preview) — only hook state advances. Execution gets its own route so the browser URL is shareable and the back button works naturally.

```
/lexical-evocation/generator
  │
  ├── [step: 'configuring']  → <ConfiguratorPage />
  └── [step: 'previewing']   → <PreviewPage />
         │
         └── "Launch exercise" → navigate('/lexical-evocation/generator/execute')
```

**Configurator step:**
- Instruction repeater: each row lets the therapist pick a seeded instruction from a dropdown, or type a custom one inline (transient — not saved to the repository).
- Generation parameters: slide count, action percentage, auto/manual mode, slide delay.
- "Generate" button calls `generateExercise()` from `useGenerator()`.

**Preview step:**
- Grid of `SlideCard` instances showing colour and audio indicator per slide.
- "Regenerate" resets to configuring state and re-runs generation with same config.
- "Save exercise" (deferred — shows as disabled in MVP).
- "Launch exercise" navigates to `/generator/execute`.

**Execution step:**
- Full-screen, no shell/nav — rendered outside the shell layout route.
- Displays `ColorDot` centred.
- Auto mode: countdown timer, space = pause/resume.
- Manual mode: any key = next slide.
- Mouse move / click: reveals a discreet "Back" button (hides after 2s of inactivity).
- `ExerciseReadyGuard` wraps this route and redirects to `/lexical-evocation/generator` if no exercise is in hook state.

### Accessibility in the UI

Colour cues must satisfy WCAG AA contrast (4.5:1 against the background). The `ColorGeneratorService` in the domain layer is responsible for generating compliant colours. The `ColorDot` component in the presentation layer is responsible for rendering them at the correct size. Patient-specific colour restrictions (colour blindness, pathologies) are a phase 2 addition — the domain service interface is designed to accept a constraint list today so the signature does not need to change later.

---

## DI configuration

`src/main.tsx` is the composition root. Repository Context providers wrap the app:

```tsx
// Phase 1
root.render(
  <InstructionRepositoryContext.Provider value={new InMemoryInstructionRepository()}>
    <CategoryRepositoryContext.Provider value={new InMemoryCategoryRepository()}>
      <ExerciseSetRepositoryContext.Provider value={new InMemoryExerciseSetRepository()}>
        <App />
      </ExerciseSetRepositoryContext.Provider>
    </CategoryRepositoryContext.Provider>
  </InstructionRepositoryContext.Provider>
);

// Phase 2 — change these three lines only
root.render(
  <InstructionRepositoryContext.Provider value={new ApiInstructionRepository()}>
    <CategoryRepositoryContext.Provider value={new ApiCategoryRepository()}>
      <ExerciseSetRepositoryContext.Provider value={new ApiExerciseSetRepository()}>
        <App />
      </ExerciseSetRepositoryContext.Provider>
    </CategoryRepositoryContext.Provider>
  </InstructionRepositoryContext.Provider>
);
```

Domain, application, and all presentation components are untouched.

The `use-generator.hook.ts` reads repositories from Context:

```typescript
export function useGenerator() {
  const instructionRepo = useContext(InstructionRepositoryContext)!;
  const [step, setStep] = useState<'configuring' | 'previewing'>('configuring');
  const [exercise, setExercise] = useState<ExerciseSet | null>(null);

  const generateExercise = async (config: ExerciseGeneratorConfig) => {
    const useCase = new GenerateExerciseUseCase(instructionRepo);
    const result = await useCase.execute(config);
    setExercise(result);
    setStep('previewing');
  };

  return { step, exercise, generateExercise };
}
```

---

## Testing this module

### Domain

- `ExerciseGeneratorService`: unit test the algorithm — given N instructions and M% action rate, assert the correct slide distribution. Pure function, no setup.
- `ColorGeneratorService`: assert generated colours meet the WCAG contrast threshold against white. Pure function.
- `Instruction` entity: assert that invalid types are rejected. Assert `isEnabled` filtering logic.
- `HexColor` value object: assert valid/invalid hex strings.
- `ContrastRatio` value object: assert ratio calculation.

### Application

- `ListInstructionsUseCase`: inject an `InMemoryInstructionRepository` seeded with known data. Assert the use case returns only enabled instructions.
- `GenerateExerciseUseCase`: inject a seeded fake repo. Assert the returned `ExerciseSet` has the correct slide count and action distribution.

### Persistence

Contract test (`instruction.repository.contract.ts`) asserts `IInstructionRepository` behaviour:
- `findAll` returns all non-deleted records.
- `findEnabled` excludes disabled records.
- `save` with a new id inserts; with an existing id updates.
- `delete` soft-deletes (record still present in raw storage but excluded from `findAll`).
- Default records reject `delete` calls.

Both `InMemoryInstructionRepository` and (phase 2) `ApiInstructionRepository` run this suite.

### Presentation

- `ColorDot`: given a `HexColor` prop, assert the rendered element has the correct background colour.
- `SlideCard`: given a slide prop, assert colour and audio indicator render correctly.
- `InstructionRow`: assert dropdown renders seeded options; assert inline input produces a transient instruction value.
- `configurator.page.spec.tsx`: integration test with `render()`. Provide `InMemoryInstructionRepository` via Context. Assert the repeater renders seeded instructions. Fill the form, click Generate, assert hook state advances to preview.
- `use-generator.hook.spec.ts`: `renderHook()` with in-memory Context. Assert `generateExercise()` populates the exercise state and advances the step.
- `exercise-ready.guard.spec.tsx`: render the guard with a mock Context. When no exercise: assert `<Navigate>` renders. When exercise exists: assert children render.

### E2E

`e2e/lexical-evocation-flow.spec.ts` (Playwright, in-memory build):
1. Navigate to `/lexical-evocation`.
2. Click "Start exercise".
3. Select two seeded instructions, set slide count to 5.
4. Click "Generate", assert preview shows 5 slides.
5. Click "Launch exercise", assert navigation to `/execute`.
6. Assert execution page renders a colour dot.
7. Press a key, assert slide advances.

---

## Deferred features checklist

- [ ] Audio instruction type: `AudioFile` entity, `IAudioFileRepository`, upload UI, storage quota enforcement
- [ ] Category management UI: tree view, create/edit/delete with hierarchy
- [ ] Instruction management UI: list, create with category assignment, edit, soft delete
- [ ] Saved exercises: `SaveExerciseUseCase`, exercise list page, audio file lock enforcement
- [ ] Module settings: `Param` entity, settings page, default injection into configurator
- [ ] Common trunk: super-admin seeds initial data on first access; reset-to-defaults option
- [ ] Onboarding tooltips: guided tour on first access
- [ ] Patient accessibility constraints: colour blindness / pathology restrictions fed to `ColorGeneratorService`
- [ ] WCAG contrast enforcement with patient profile: phase 2 extension of `ColorGeneratorService` signature
