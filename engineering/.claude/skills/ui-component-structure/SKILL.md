---
name: ui-component-structure
description: >
  Enforces and scaffolds a feature-first React component folder structure.
  Use this skill whenever a UI component is created, updated, moved, or reorganised — even if the user
  just says "create a component", "add a new feature", "scaffold UserProfile", or "new hook for X".
  Also triggers on structural audits: "check component structure", "audit UI folders",
  "are my components organised correctly", "fix folder structure", or "reorganise components".
  Applies to any React component tree in the project — adjust the target paths below to match the repo.
user-invocable: true
argument-hint: "<optional: component or feature name, or 'check' to audit>"
---

# UI Component Structure

This skill enforces a feature-first folder convention for React components.
The guiding principle is **co-location** — every piece of code lives as close as possible to where
it is used, and only gets promoted upward when genuinely shared.

**Primary targets:** the project's component/page directories (e.g. `src/pages/`, `src/components/` —
adjust to match wherever this repo keeps its feature folders).

---

## The structure at a glance

The feature folder **is** the root component boundary. Component files sit directly at the feature
root — no nested folder with the same name.

```
FeatureName/
  FeatureName.tsx          # Root component implementation
  index.ts                 # Public API (re-export)
  types.ts                 # Props interface and related types
  constants.ts             # Constants used only by the root component
  __tests__/
    FeatureName.test.tsx
  components/               # Sub-components (each follows the same structure recursively)
    SubComponent/
      SubComponent.tsx
      index.ts
      types.ts
      __tests__/
        SubComponent.test.tsx
  hooks/                    # Shared hooks (used by 2+ sub-components)
    useHookName/
      useHookName.ts
      index.ts
      __tests__/
        useHookName.test.ts
  utils/                    # Shared utils (used by 2+ sub-components)
    utilName/
      utilName.ts
      index.ts
      __tests__/
        utilName.test.ts
  constants/                # Shared constants (used by 2+ sub-components)
    constantGroupName.ts    # No barrel index.ts needed
```

---

## Co-location rule

This is the most important rule in the entire structure and the one that prevents the codebase
from drifting into a "dump everything in a shared folder" pattern.

**Hooks, utils, and constants live inside the component that uses them.**
They only get promoted to the feature-level folder when they are imported by 2 or more sub-components.

### Allowed placement — only two levels

Constants, hooks, and utils can only exist at exactly two levels:

1. **Component level** — inside a component folder, as a direct child
2. **Feature root level** — when shared by 2+ sub-components

They must **never** be nested inside each other. A constant does not go inside a hook folder.
A util does not go inside another util folder. A hook does not go inside a utils folder.
Each is a sibling within the component or feature folder — they sit alongside each other, not
inside each other.

**Correct — constants and hooks are siblings inside the component folder:**
```
TableHeader/
  TableHeader.tsx
  index.ts
  types.ts
  constants.ts                     # sits in the component folder, NOT inside hooks/
  hooks/
    useSortColumn/
      useSortColumn.ts
      index.ts
      __tests__/
        useSortColumn.test.ts
  utils/
    formatHeaderLabel/
      formatHeaderLabel.ts
      index.ts
      __tests__/
        formatHeaderLabel.test.ts
  __tests__/
    TableHeader.test.tsx
```

**Wrong — constant nested inside a hook folder:**
```
TableHeader/
  hooks/
    useSortColumn/
      useSortColumn.ts
      constants.ts               # WRONG — constants never go inside hooks/
```

**Wrong — util nested inside a hook folder:**
```
TableHeader/
  hooks/
    useSortColumn/
      utils/
        formatSomething.ts       # WRONG — utils never go inside hooks/
```

When a hook/util/constant is shared across sub-components, it lives at the feature root:

```
DataTable/
  hooks/
    usePagination/          # shared between DataTable and Pagination
      usePagination.ts
      index.ts
      __tests__/
        usePagination.test.ts
  constants/
    tableConfig.ts          # shared between Pagination and TableRow
```

The reverse also matters: during an audit, if a feature-level hook/util/constant is only imported
by one sub-component, flag it for demotion back into that component's folder.

---

## File responsibilities

| File | Purpose |
|---|---|
| `ComponentName.tsx` | The React component implementation. Imports from sibling `types.ts`, local hooks/utils, and child components. |
| `index.ts` | Public API. Re-exports the component and any types consumers need. Nothing else — no logic. |
| `types.ts` | All TypeScript interfaces and types for the component. Props must use `Readonly<>`. |
| `constants.ts` | Static values — enums, config objects, magic strings. No runtime logic. |
| `__tests__/` | Co-located test folder. Test file mirrors the source file name with `.test.tsx` / `.test.ts` suffix. |

---

## Mode 1: Scaffold a new component

When the user asks to create a new component or feature, generate the full folder structure
following the convention above.

### Steps

1. **Determine scope.** Is this a standalone component (simple) or a feature with sub-components?
   Ask if unclear.

2. **Create the folder tree.** Use the structure above. Every component gets at minimum:
   - `ComponentName.tsx` with a basic functional component
   - `index.ts` with `export { default } from './ComponentName'` and named type exports
   - `types.ts` with a `Readonly<ComponentNameProps>` interface
   - `__tests__/ComponentName.test.tsx` with a render smoke test

3. **Place hooks/utils/constants correctly.** If the user describes helpers during creation,
   decide whether they belong at the component level or feature level based on intended usage.
   When in doubt, start local (inside the component) — it is easier to promote later than to demote.

4. **Wire up exports.** If the feature has a public API (used outside the feature boundary),
   the top-level `index.ts` should re-export what consumers need.

### Scaffold template — Component.tsx

```tsx
import type { ComponentNameProps } from './types';

const ComponentName = ({ ...props }: ComponentNameProps) => {
  return (
    <div>
      {/* implementation */}
    </div>
  );
};

export default ComponentName;
```

### Scaffold template — types.ts

```ts
export interface ComponentNameProps {
  readonly className?: string;
}
```

### Scaffold template — index.ts

```ts
export { default } from './ComponentName';
export type { ComponentNameProps } from './types';
```

### Scaffold template — __tests__/ComponentName.test.tsx

```tsx
import { render, screen } from '@testing-library/react';
import ComponentName from '..';

describe('ComponentName', () => {
  it('renders without crashing', () => {
    render(<ComponentName />);
  });
});
```

### Scaffold template — hook (useHookName.ts)

```ts
import { useState } from 'react';

export const useHookName = () => {
  // implementation
};
```

### Scaffold template — hook index.ts

```ts
export { useHookName } from './useHookName';
```

---

## Mode 2: Audit and reorganise

When the user asks to check, audit, or fix the component structure, scan the target directory
and report violations.

### Audit steps

1. **Scan the target.** Default to the project's component/page directories (e.g. `src/pages/`,
   `src/components/`). If the user specifies a different path or a specific feature, use that instead.

2. **Check each feature folder against the rules:**

   | Check | What to look for |
   |---|---|
   | Missing files | Component folder without `index.ts`, `types.ts`, or `__tests__/` |
   | Flat files | `.tsx` files sitting directly in the feature root instead of inside a named component folder |
   | Misplaced hooks/utils | Feature-level `hooks/` or `utils/` entry imported by only 1 sub-component (should be demoted) |
   | Orphaned local helpers | A hook/util inside a component folder that is imported by a sibling component (should be promoted) |
   | Missing test folders | Any component, hook, or util folder without `__tests__/` |
   | Barrel issues | `index.ts` containing logic instead of pure re-exports |
   | Constants without a home | Shared constants sitting in the feature root as loose `.ts` files instead of inside `constants/` |
   | Naming mismatches | Folder name does not match the main export file name |
   | Illegal nesting | Constants, hooks, or utils nested inside each other (e.g. `constants.ts` inside a `hooks/` folder) |

3. **Report findings.** Group by feature, show the violation and the fix. Example:

   ```
   DataTable/
     FAIL  hooks/useSortColumn — only imported by TableHeader
           -> move to DataTable/components/TableHeader/hooks/useSortColumn/

     FAIL  components/TableRow — missing __tests__/ folder
           -> create DataTable/components/TableRow/__tests__/TableRow.test.tsx

     PASS  components/Pagination — structure OK
     PASS  constants/tableConfig.ts — imported by Pagination + TableRow (correctly shared)
   ```

4. **Offer to fix.** After reporting, ask: "Want me to reorganise the files?" If yes, move files
   and update all affected imports automatically. Run the TypeScript compiler afterwards to verify
   nothing broke.

---

## Mode 3: Update an existing component

When modifying an existing component, respect the structure that is already in place:

- Adding a new hook? Place it inside the component folder unless it is already needed by a sibling.
- Extracting logic from a component into a util? Same rule — start local.
- Adding a sub-component? Create it under `components/` with full structure.
- If the change causes a local hook/util/constant to now be shared, promote it in the same PR.

After any structural change, do a quick local audit of the affected feature to catch anything
that drifted.

---

## Full example — complex feature

For reference, here is a complete DataTable feature showing all patterns in action:

```
DataTable/
  DataTable.tsx                # Root component — sits directly at feature root
  index.ts
  types.ts
  constants.ts                 # Constants only DataTable.tsx uses
  __tests__/
    DataTable.test.tsx
  components/
    TableHeader/
      TableHeader.tsx
      index.ts
      types.ts
      constants.ts             # SORT_DIRECTIONS — only TableHeader uses it
      hooks/
        useSortColumn/
          useSortColumn.ts
          index.ts
          __tests__/
            useSortColumn.test.ts
      __tests__/
        TableHeader.test.tsx
    TableRow/
      TableRow.tsx
      index.ts
      types.ts
      utils/
        formatCellValue/
          formatCellValue.ts
          index.ts
          __tests__/
            formatCellValue.test.ts
      __tests__/
        TableRow.test.tsx
    Pagination/
      Pagination.tsx
      index.ts
      types.ts
      __tests__/
        Pagination.test.tsx
  hooks/                       # Shared — promoted to feature level
    usePagination/
      usePagination.ts
      index.ts
      __tests__/
        usePagination.test.ts
  utils/                       # Shared — promoted to feature level
    calculatePageRange/
      calculatePageRange.ts
      index.ts
      __tests__/
        calculatePageRange.test.ts
  constants/                   # Shared — promoted to feature level
    tableConfig.ts
```

Why things are where they are:
- `DataTable.tsx` sits directly in `DataTable/` — the feature folder is the root component boundary, no double nesting.
- `useSortColumn` is inside `TableHeader` because only TableHeader uses it.
- `formatCellValue` is inside `TableRow` because only TableRow uses it.
- `usePagination` is at feature level because both `DataTable` and `Pagination` import it.
- `calculatePageRange` is at feature level because both `Pagination` and `TableRow` import it.
- `tableConfig.ts` (shared constants) is at feature level because Pagination and TableRow both reference `DEFAULT_PAGE_SIZE`.
- `constants.ts` inside `TableHeader` holds `SORT_DIRECTIONS` which only TableHeader needs.
