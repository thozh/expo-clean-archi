# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a React Native + Expo application demonstrating Clean Architecture, Event-Driven Architecture (EDA), Test-Driven Development (TDD), and Vertical Slices organization. The architecture decouples business logic from the UI layer using Redux/RTK as the application core, not just for state management.

**Key Philosophy**: React is used only for UI. All business logic, state transitions, and application flow reside in use cases (Redux thunks), reducers, and the Redux store.

## Common Commands

### Development
```bash
npm install              # Install dependencies
npm run start           # Start Expo development server
npm run android         # Run on Android
npm run ios             # Run on iOS
npm run web             # Run on web
npm run lint            # Run ESLint
```

### Testing
```bash
npm run test            # Run all tests in watch mode
npm test -- --watchAll=false  # Run tests once without watch mode
```

To run a single test file:
```bash
npm test -- <path-to-test-file>
# Example: npm test -- src/exercice/features/create-exercice/create-exercice.use-case.spec.ts
```

## Architecture

### Core Principles

1. **Clean Architecture**: Dependencies point inward. UI → Use Cases → Repository (via interface/DIP)
2. **Redux as Application Core**: Redux lives in the domain/application layer, not infrastructure
3. **Event-Driven Architecture**: Redux actions are treated as "events" that reducers listen to
4. **Vertical Slices**: Features are organized by capability, not by technical layer
5. **TDD with Social Tests**: Tests assert against state using selectors, not implementation details

### Redux Usage (4 Purposes)

1. **State Management**: Global application state
2. **Synchronous Event Bus**: Actions are events; reducers listen and update state
3. **Dependency Injection**: `extraArgument` in store configuration injects repositories into thunks
4. **Middleware for Side Effects**: Thunks handle async operations (API calls)

### Execution Flow

```
UI Component → Use Case (Thunk) → Repository (via DIP)
                 ↓ dispatch
              Events ← Reducer listens
                 ↓
           State Model updated
                 ↓
             Selectors
                 ↓
         UI re-renders (useSelector)
```

### Vertical Slice Structure

Each feature directory contains:
- `*.state.model.ts` - State shape, initial state, and selectors
- `*.events.ts` - Redux actions (created with `createAction`)
- `*.reducer.ts` - Listens to events and updates state
- `*.use-case.ts` - Business logic as Redux thunk
- `*.use-case.spec.ts` - TDD acceptance tests
- `*.state-machine.md` - Mermaid state transition diagram
- `*-validator.service.ts` - Validation logic (if needed)

Shared across features (in `features/shared/`):
- Repository interface and implementations
- Domain model types
- Combined reducers

### Directory Structure

```
src/
  <domain>/              # e.g., exercice, notification, muscle
    features/
      <feature-name>/    # e.g., create-exercice, list-exercices
        *.state.model.ts
        *.events.ts
        *.reducer.ts
        *.use-case.ts
        *.use-case.spec.ts
        *.state-machine.md
        *-validator.service.ts
      shared/
        <domain>.repository.interface.ts
        <domain>.state.model.ts
        <domain>.reducer.ts
        infrastructure/
          <domain>.repository.in-memory.ts
        test/
          <domain>-*.repository.fake.ts
    ui/
      <component>/
        Component.tsx
        component.view-model.ts
  shared/
    application/
      root.store.ts
      root.reducer.ts
      test/test.store.ts
```

### Testing Approach

**Social/Acceptance Tests** (Not Unit Tests):
- Test use cases from the outside (call use case → assert state with selectors)
- Use fake repositories to simulate success/error/loading scenarios
- Tests are structured as BDD scenarios: Given/When/Then
- Avoid testing implementation details (reducers directly, internal methods)
- Tests serve as specifications and should only break when behavior changes

**Test Store Setup**:
```typescript
const testStore = createTestStore(preloadedState, {
  exerciceRepository: new ExerciceSuccessRepositoryFake()
});
```

### Dependency Injection via Redux

The main store ([src/shared/application/root.store.ts](src/shared/application/root.store.ts)) injects dependencies via `extraArgument`:

```typescript
configureStore({
  reducer: rootReducer,
  middleware: (getDefaultMiddleware) => getDefaultMiddleware({
    thunk: {
      extraArgument: {
        exerciceRepository: new ExerciceRepositoryInMemory(),
      },
    },
  }),
});
```

Use cases access injected dependencies as the third parameter:
```typescript
export const createExerciceUseCase = (command: Command): Thunk =>
  async (dispatch, _, {exerciceRepository}) => {
    // Use exerciceRepository
  };
```

### Important Conventions

- **No DDD Tactical Patterns**: Business rules are handled by backend. No entities/aggregates/value objects in frontend
- **Reducers Don't Call Use Cases**: If one event triggers another, dispatch the second event within the same use case
- **UI Doesn't Manage Flow**: React components only call use cases; they don't orchestrate multi-step flows
- **State Not Normalized**: State structure is kept simple (not using normalizr)
- **No Reselect**: Selectors are simple getters without derived/transformed data
- **Events Over Actions**: Redux actions are named and treated as events (past tense, e.g., `exerciceCreated`)

### Key Files

- [src/shared/application/root.store.ts](src/shared/application/root.store.ts) - Main Redux store with DI configuration
- [src/shared/application/root.reducer.ts](src/shared/application/root.reducer.ts) - Root reducer combining all features
- [src/shared/application/test/test.store.ts](src/shared/application/test/test.store.ts) - Test store factory for testing
- [src/shared/application/thunk.type.ts](src/shared/application/thunk.type.ts) - Thunk type definition with Container

### TypeScript Path Alias

The project uses `@/*` as an alias for the root directory:
```typescript
import {createTestStore} from "@/src/shared/application/test/test.store";
```

## Development Workflow (TDD)

1. **Define User Story**: Write scenarios (Given/When/Then)
2. **Create State Machine Diagram**: Use Mermaid to visualize state transitions
3. **Write First Test**: Red (test fails)
4. **Implement Use Case**: Green (baby steps)
5. **Refactor**: Clean up code
6. **Repeat**: Add next test scenario

State machine diagrams are embedded in `*.state-machine.md` files using Mermaid syntax and can be viewed directly in WebStorm/IDEs supporting Mermaid.

## Notes

- React Native screens and components are intentionally kept simple (may contain TODOs or TypeScript warnings)
- No tests for React components (focus is on business logic)
- The in-memory repository allows frontend development without a backend
- Fake repositories in tests allow testing different scenarios (success/error/loading) without real API calls
