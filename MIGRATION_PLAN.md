# Migration Plan: localStorage → SQLite (sql.js)

## Overview
Migrate TodoFlow from browser localStorage to SQLite using **sql.js** (SQLite compiled to WebAssembly). This keeps the app frontend-only while adding a real SQL database.

---

## Phase 1: Setup & Dependencies

### Task 1.1: Install sql.js package
```bash
npm install sql.js
npm install -D @types/sql.js
```

### Task 1.2: Copy WASM file to public directory
```bash
cp node_modules/sql.js/dist/sql-wasm.wasm public/
```

### Task 1.3: Update Vite configuration
**File:** `vite.config.ts`
- Add sql.js to `optimizeDeps.exclude`
- Add WASM to `assetsInclude`

---

## Phase 2: Create Database Layer

### Task 2.1: Create IndexedDB persistence helpers
**File:** `src/utils/sqliteDb.ts`
- Create `saveDatabaseToIndexedDB(data: Uint8Array)` function
- Create `loadDatabaseFromIndexedDB(): Promise<Uint8Array | null>` function
- Use IndexedDB store name: `todoflow-db`

### Task 2.2: Create database initialization
**File:** `src/utils/sqliteDb.ts`
- Create `initDatabase()` function
- Load sql.js with WASM from `/sql-wasm.wasm`
- Try loading existing DB from IndexedDB
- If no existing DB, create new one with schema

### Task 2.3: Define SQLite schema
**File:** `src/utils/sqliteDb.ts`
```sql
CREATE TABLE IF NOT EXISTS todos (
    id TEXT PRIMARY KEY,
    text TEXT NOT NULL,
    completed INTEGER NOT NULL DEFAULT 0,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL
);
```

### Task 2.4: Create database singleton accessor
**File:** `src/utils/sqliteDb.ts`
- Create `getDatabase()` function returning the DB instance
- Create `saveDatabase()` function to export and persist to IndexedDB

---

## Phase 3: Create Migration Logic

### Task 3.1: Create migration module
**File:** `src/utils/migration.ts`
- Define migration flag key: `todoflow-sqlite-migrated`
- Define old storage key: `todos-app-data`

### Task 3.2: Implement migration function
**File:** `src/utils/migration.ts`
- Create `migrateFromLocalStorage()` function
- Check if already migrated (flag in localStorage)
- If not migrated:
  - Read old todos from localStorage
  - Parse JSON
  - Insert each todo into SQLite
  - Save database
  - Set migration flag

---

## Phase 4: Update Storage Layer

### Task 4.1: Add initialization function
**File:** `src/utils/storage.ts`
- Create `initStorage()` async function
- Call `initDatabase()` from sqliteDb.ts

### Task 4.2: Rewrite loadTodos function
**File:** `src/utils/storage.ts`
- Make function async: `loadTodos(): Promise<Todo[]>`
- Query SQLite: `SELECT * FROM todos ORDER BY created_at DESC`
- Map results to Todo objects (convert completed 0/1 to boolean, strings to Dates)

### Task 4.3: Rewrite saveTodos function
**File:** `src/utils/storage.ts`
- Make function async: `saveTodos(todos: Todo[]): Promise<void>`
- Clear existing todos: `DELETE FROM todos`
- Insert all todos using prepared statement
- Call `saveDatabase()` to persist to IndexedDB

---

## Phase 5: Update Hook & UI

### Task 5.1: Add loading and error states
**File:** `src/hooks/useTodos.ts`
- Add `isLoading` state (initially true)
- Add `error` state (initially null)

### Task 5.2: Make initialization async
**File:** `src/hooks/useTodos.ts`
- Update mount useEffect to be async
- Call `initStorage()` first
- Call `migrateFromLocalStorage()`
- Call `loadTodos()` and set todos
- Set `isLoading` to false when done
- Catch errors and set error state

### Task 5.3: Update save effect
**File:** `src/hooks/useTodos.ts`
- Skip saving when `isLoading` is true
- Handle async `saveTodos()` (fire-and-forget is OK for PoC)

### Task 5.4: Return loading/error from hook
**File:** `src/hooks/useTodos.ts`
- Add `isLoading` and `error` to return object

### Task 5.5: Handle loading state in UI (optional)
**File:** `src/App.tsx`
- Show loading indicator while `isLoading` is true
- Show error message if `error` is not null

---

## Phase 6: Testing & Verification

### Task 6.1: Test fresh install
- Clear browser data
- Start app
- Add todos
- Verify they appear

### Task 6.2: Test persistence
- Refresh page
- Verify todos still exist
- Check DevTools → Application → IndexedDB → `todoflow-db`

### Task 6.3: Test migration
- Set up old localStorage data manually:
  ```js
  localStorage.setItem('todos-app-data', JSON.stringify([{id:'test',text:'Old todo',completed:false,createdAt:'2024-01-01',updatedAt:'2024-01-01'}]))
  ```
- Clear IndexedDB
- Refresh app
- Verify old todo appears (migrated to SQLite)

### Task 6.4: Test CRUD operations
- Add new todo
- Toggle todo completion
- Edit todo text
- Delete todo
- Clear completed
- Verify all operations work and persist

---

## File Summary

| Phase | File | Action |
|-------|------|--------|
| 1 | `package.json` | Add sql.js dependency |
| 1 | `vite.config.ts` | Configure WASM handling |
| 1 | `public/sql-wasm.wasm` | Copy from node_modules |
| 2 | `src/utils/sqliteDb.ts` | CREATE - DB manager |
| 3 | `src/utils/migration.ts` | CREATE - Migration logic |
| 4 | `src/utils/storage.ts` | MODIFY - Use SQLite |
| 5 | `src/hooks/useTodos.ts` | MODIFY - Async init |
| 5 | `src/App.tsx` | MODIFY - Loading state (optional) |

---

## Quick Reference: SQLite ↔ TypeScript Mapping

| SQLite Column | SQLite Type | TypeScript | Conversion |
|---------------|-------------|------------|------------|
| id | TEXT | string | none |
| text | TEXT | string | none |
| completed | INTEGER (0/1) | boolean | `!!value` / `value ? 1 : 0` |
| created_at | TEXT | Date | `new Date(value)` / `.toISOString()` |
| updated_at | TEXT | Date | `new Date(value)` / `.toISOString()` |
