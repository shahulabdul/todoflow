# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

HabitFlow is a frontend-only React habit tracking application with browser-based localStorage persistence. No backend server or database exists - all data is stored locally in the user's browser.

## Workflow Preferences

- **Always ask for approval before deleting any file** - Never delete files without explicit user permission first.

## Development Commands

```bash
# Start development server (http://localhost:5173)
npm run dev

# Build for production
npm run build

# Preview production build locally
npm run preview

# Run linter
npm run lint
```

## Architecture

### Data Flow (Frontend-Only)

This is a **client-side only** application with the following data flow:

```
User Interface → React Components → useTodos Hook → Browser localStorage
```

There is no backend API, server, or database. All data persists in the browser's localStorage using the key `'habits-app-data'`.

### State Management Pattern

The application uses a **centralized custom hook pattern** (`src/hooks/useTodos.ts`) instead of Redux or Context API:

- **Single source of truth**: `useTodos` hook manages all todo state and operations
- **Automatic persistence**: React's `useEffect` automatically saves to localStorage whenever todos change (line 14-16)
- **Automatic loading**: On mount, todos are loaded from localStorage (line 9-12)
- **Filtered view**: The hook handles filtering logic internally and returns pre-filtered todos based on current filter state

### Key Architecture Points

1. **Component Hierarchy**:
   - `App.tsx` is the root component that orchestrates all features
   - All child components are "dumb" presentational components that receive props
   - All business logic lives in `useTodos` hook

2. **Data Persistence** (`src/utils/storage.ts`):
   - `saveTodos()`: Serializes todos array to JSON and stores in localStorage
   - `loadTodos()`: Deserializes from localStorage and converts date strings back to Date objects
   - Error handling ensures app doesn't crash if localStorage fails

3. **Component Communication**:
   - Unidirectional data flow: parent passes callbacks down, children call them
   - No prop drilling beyond one level - all components receive props directly from App.tsx
   - No component-to-component communication - everything flows through App via useTodos

4. **State Structure**:
   ```typescript
   interface Todo {
     id: string;          // Generated via crypto.randomUUID()
     text: string;
     completed: boolean;
     createdAt: Date;
     updatedAt: Date;     // Updates on edit or toggle
   }
   ```

5. **Inline Editing Pattern** (`TodoItem.tsx`):
   - Double state: `isEditing` (boolean) + `editText` (string)
   - Keyboard shortcuts: Enter to save, Escape to cancel
   - Auto-focus on edit mode, blur to save

## Tech Stack

- **React 18.3.1** - UI library
- **TypeScript 5.5.3** - Type safety
- **Vite 5.4.2** - Build tool and dev server
- **Tailwind CSS 3.4.1** - Utility-first styling
- **Lucide React 0.344.0** - Icon components

## File Structure

```
src/
├── components/          # Presentational components (receive all state via props)
│   ├── TodoForm.tsx    # Add new todo input with submit button
│   ├── TodoItem.tsx    # Individual todo with inline edit/delete/toggle
│   ├── TodoFilter.tsx  # Filter tabs (All/Active/Completed) with counts
│   └── TodoStats.tsx   # Progress bar and bulk actions
├── hooks/
│   └── useTodos.ts     # Central state management and business logic
├── utils/
│   └── storage.ts      # localStorage abstraction layer
├── types/
│   └── index.ts        # Todo interface and FilterType union
├── App.tsx             # Root component, orchestrates all features
└── main.tsx            # Entry point, mounts React app
```

## Important Implementation Notes

### When Adding Features

- **Do not add a backend** unless explicitly requested - this is a local-first app by design
- New todo operations should be added to `useTodos.ts` hook, not scattered across components
- All components should remain "dumb" - they receive props and call callbacks, no business logic
- Any new data fields on Todo interface must be handled in `storage.ts` date serialization logic

### localStorage Constraints

- Browser localStorage limit: ~5-10MB (sufficient for thousands of todos)
- Data is per-origin: only accessible to this app on this domain
- Cleared when user clears browser data
- Synchronous API - no async/await needed for storage operations

### Styling Convention

- Uses Tailwind utility classes exclusively
- Gradient backgrounds: `bg-gradient-to-br from-blue-50 via-indigo-50 to-purple-50`
- Glassmorphism: `bg-white/80 backdrop-blur-sm`
- Interactive states handled via hover/focus modifiers
- No custom CSS files beyond `index.css` (which imports Tailwind directives)
