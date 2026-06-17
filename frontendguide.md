# Frontend Developer Guide — ENTNT Employee Portal UI

> **Who this is for:** any developer writing frontend code in `entnt.employeeportal.ui`.
> **The promise:** if you follow this guide, your code will look like everyone else's, a new teammate will find it in seconds, and reviews will be fast.
> **How to read it:** Part 1 in 5 minutes on day one. Parts 2–3 before your first PR. Part 4 is your template every time you build something.

Rule language: **MUST** = enforced (CI/review will block you). **SHOULD** = strong default, deviate only with a reason in the PR. **NEVER** = do not do this.

---

## Part 1 — Orientation

### 1.1 The 5 golden rules (memorize these)

1. **Code lives in its feature.** Belongs to one domain → `features/<domain>/`. Used by ≥2 domains → `shared/`. When unsure, start in the feature; promote later.
2. **Components render; hooks fetch and compute.** A component file **NEVER** calls `fetch`/the API directly and **NEVER** holds business rules. That goes in a hook or `lib/`.
3. **Small and focused.** Split a component when it does **more than one job** — not at an arbitrary line count. Size is a smell, not a limit: a long but cohesive component can be fine; a short one doing three things is not. One component per file.
4. **Predictable names.** Name a thing by _what it is_ (`LeaveHistoryTable`), never _what it technically is_ (`Table2`, `Helpers`, `Misc`).
5. **Client checks are UX only.** Hiding a button by permission is a convenience; the **server** is the real gate. Never assume the client protects data.

### 1.2 Tech stack (and why each exists)

| Tool                                 | Role             | Notes                                                       |
| ------------------------------------ | ---------------- | ----------------------------------------------------------- |
| **React 18 + TypeScript**            | UI, typed        | `strict` is on — honor it; no `any` at boundaries           |
| **Vite**                             | build/dev server | fast HMR; config in `vite.config.ts`                        |
| **@azure/msal-browser / msal-react** | authentication   | Azure AD; tokens held in memory (see §11)                   |
| **TanStack Query**                   | server-state     | caching, dedup, cancellation, invalidation — the data layer |
| **TailwindCSS**                      | styling          | utility-first; shared primitives in `shared/ui`             |
| **@microsoft/signalr**               | realtime         | live notifications                                          |
| **chart.js / react-chartjs-2**       | charts           | memoize `data`/`options` (see §13)                          |

### 1.3 Current state & migration (read this once, honestly)

This codebase is **mid-migration**. Two patterns coexist:

- ✅ **The standard** (this guide): `features/` slices with `page → hook → api` layering.
- ⚠️ **Legacy**: a flat `components/` folder with large multi-purpose dashboards and a single 5,000-line `WebCalls.tsx`.

**Your job:** write **all new code** to this guide. When you touch legacy code, leave it cleaner than you found it (the boy-scout rule). The refactoring epic (#397) is converging everything onto the standard — don't add to the legacy pile.

---

## Part 2 — Where things go

### 2.1 Folder structure (the map)

```
src/
├── app/                      # app wiring — you rarely edit this
│   ├── App.tsx               #   thin shell
│   ├── routes.tsx            #   the route table
│   └── providers/            #   AppProviders + every context provider
│
├── features/                 # 95% of your work happens here
│   └── <domain>/             #   e.g. employees, leaves, payroll, attendance…
│       ├── pages/            #     route-level screens (orchestrators)
│       ├── components/       #     UI specific to this domain
│       ├── hooks/            #     data + logic hooks for this domain
│       ├── api/              #     <domain>.api.ts — network calls
│       ├── types/            #     domain types
│       └── utils/            #     pure helpers for this domain
│
├── shared/                   # cross-feature ONLY (used by ≥2 domains)
│   ├── ui/                   #   dumb primitives: Button, Table, Field, StatCard…
│   ├── hooks/                #   generic hooks: useDebouncedValue, useClickOutside…
│   ├── lib/                  #   pure utils: date, currency, format
│   ├── api/                  #   apiClient.ts (the one fetch wrapper)
│   └── types/                #   cross-feature types
│
├── layouts/                  # Navbar, breadcrumbs, page shells
└── assets/
```

### 2.2 "Where does my code go?" — decision tree

```
Is it a route/screen?               → features/<domain>/pages/
Is it UI used by ONE domain?        → features/<domain>/components/
Is it UI used by 2+ domains?        → shared/ui/
Does it fetch or derive data?       → a hook in features/<domain>/hooks/ (or shared/hooks/)
Is it a network call?               → features/<domain>/api/<domain>.api.ts
Is it a pure function (no React)?   → features/<domain>/utils/ or shared/lib/
Is it a TS type/interface?          → features/<domain>/types/ or shared/types/
```

**MUST:** if you're about to create a file named `helpers`, `utils`, `common`, `misc`, `data`, or `Shared` — stop. Name it for its contents and place it by the tree above.

### 2.3 Naming conventions

| Thing                 | Convention                       | Example                           |
| --------------------- | -------------------------------- | --------------------------------- |
| Folder                | `kebab-case`                     | `features/leaves`, `shared/ui`    |
| Component file        | `PascalCase.tsx`                 | `LeaveHistoryTable.tsx`           |
| Hook file             | `useXxx.ts(x)` (lowercase `use`) | `useLeavesData.ts`                |
| Util/types file       | `camelCase.ts`                   | `formatDate.ts`, `leave.types.ts` |
| API module            | `<domain>.api.ts`                | `leaves.api.ts`                   |
| Component / type      | `PascalCase`                     | `LeaveStatus`, `EmployeeCard`     |
| Hook / function / var | `camelCase`                      | `useLeavesData`, `formatCurrency` |
| Boolean               | `is/has/can` prefix              | `isLoading`, `canApprove`         |
| Event handler         | `handleX` (def) / `onX` (prop)   | `handleSubmit` / `onSubmit`       |

**Bad → Good**

| ❌ Bad            | ✅ Good                                |
| ----------------- | -------------------------------------- |
| `Helpers.tsx`     | `shared/lib/formatDate.ts`             |
| `Model.tsx`       | `features/leaves/types/leave.types.ts` |
| `table.tsx`       | `shared/ui/Table.tsx`                  |
| `UseApiToken.tsx` | `useApiToken.ts`                       |
| `data.ts`         | `employeeDirectory.api.ts`             |
| `Modal.tsx`       | `ApplyLeaveModal.tsx`                  |

---

## Part 3 — How to write it

### 3.1 The architecture & data flow (the one diagram to remember)

```
features/<domain>/pages/<X>Page.tsx        ← orchestrates only (thin, <150 lines)
        │ uses
        ▼
features/<domain>/hooks/use<X>Data.ts      ← TanStack Query: fetch + derive
        │ calls
        ▼
features/<domain>/api/<domain>.api.ts      ← typed functions (one per endpoint)
        │ via
        ▼
shared/api/apiClient.ts                    ← the ONE fetch wrapper (token, base URL, errors, AbortSignal)

The page RENDERS features/<domain>/components/*  (feature UI)
which is BUILT FROM shared/ui/*                  (dumb primitives)
```

Every arrow is a plain import a junior can follow. **No magic, no clever abstractions.**

### 3.2 Components

- **SHOULD** keep components single-purpose. There is **no hard line limit** — judge by responsibility and cohesion, not line count. Large size is a _prompt_ to ask "is this doing more than one thing?"; if yes, split into a container + children. A long component that does exactly one well-defined job may legitimately stay long.
- **MUST** be one of two kinds:
  - **Container / Page** — orchestrates: calls hooks, wires children together, holds no business logic and no raw `fetch`.
  - **Presentational** — receives props, renders UI, no data fetching. `shared/ui` is _always_ presentational.
- **MUST** one default export = one component per file.
- **SHOULD** pass data down via props; if you're threading the same prop through 3+ levels, use a hook or context instead (no deep prop drilling).
- **NEVER** mix a form + a table + a modal + data fetching in one file. Each is its own component.

**Container shape:**

```tsx
export function LeavesPage() {
  const { leaves, isLoading, error } = useLeavesData();
  const [filters, setFilters] = useLeaveFilters();

  if (isLoading) return <Spinner />;
  if (error) return <ErrorState error={error} />;
  if (!leaves.length) return <EmptyState message="No leaves yet" />;

  return (
    <PageLayout title="Leaves">
      <LeaveFilters value={filters} onChange={setFilters} />
      <LeaveHistoryTable leaves={leaves} />
    </PageLayout>
  );
}
```

### 3.3 State management — pick the smallest tool that works

| Need                                | Use                     | Don't                              |
| ----------------------------------- | ----------------------- | ---------------------------------- |
| Local UI state (open/closed, input) | `useState`              | a context                          |
| Many related fields (>~8)           | `useReducer`            | 8 separate `useState`              |
| Server data                         | **TanStack Query hook** | `useState` + `useEffect` + `fetch` |
| Truly global (theme, auth, alerts)  | a context provider      | prop drilling                      |
| Value derived from other state      | `useMemo`               | a `useState` synced by `useEffect` |

- **MUST** memoize every context `value` with `useMemo` and wrap its callbacks in `useCallback`. An un-memoized provider value re-renders the whole app.
- **NEVER** mirror props/state into another `useState` and sync with `useEffect` — derive it with `useMemo`.

### 3.4 Hooks

- **MUST** start with lowercase `use` and follow the rules of hooks (top level, not in conditions/loops).
- **MUST** move data fetching and non-trivial derivation **out of components** into hooks.
- Two kinds:
  - **Data hooks** — `useLeavesData`, `useEmployee(id)` → wrap the api layer with TanStack Query.
  - **UI hooks** — `useClickOutside`, `useDebouncedValue` → generic, live in `shared/hooks`.
- **SHOULD** return a small, named object (`{ data, isLoading, error }`), not a positional tuple, once it has 3+ values.

```ts
// features/leaves/hooks/useLeavesData.ts
export function useLeavesData(employeeId: string) {
  return useQuery({
    queryKey: ["leaves", employeeId],
    queryFn: ({ signal }) => leavesApi.getLeaves(employeeId, signal),
    staleTime: 60_000,
  });
}
```

### 3.5 API layer

- **NEVER** call `fetch` or `WebCalls` from a component. Components use **query hooks** only.
- **MUST** route every network call through `shared/api/apiClient.ts` (handles base URL, bearer token, errors, `AbortSignal`).
- **MUST** put endpoint functions in `features/<domain>/api/<domain>.api.ts`, typed in and out.

```ts
// features/leaves/api/leaves.api.ts
import { apiClient } from "@/shared/api/apiClient";
import type { Leave, ApplyLeaveDto } from "../types/leave.types";

export const leavesApi = {
  getLeaves: (employeeId: string, signal?: AbortSignal) =>
    apiClient.get<Leave[]>(`/Leave/${employeeId}`, { signal }),
  applyLeave: (dto: ApplyLeaveDto) =>
    apiClient.post<Leave>("/Leave/apply", dto),
};
```

### 3.6 Auth & permissions

- Tokens come from **MSAL** and are held **in memory** (via the auth hooks). **NEVER** persist a token to `localStorage` or log it.
- Authorization is RBAC from the backend profile (`/auth/me`). Use the permission hook to gate UI:

```tsx
const { canApproveLeave } = usePermission();
{
  canApproveLeave && <Button onClick={onApprove}>Approve</Button>;
}
```

- **MUST** treat all `can*`/`isAdmin` checks as **UX hints only**. The corresponding API endpoint enforces the real rule on the server. **NEVER** rely on the client to protect data.
- Route protection goes through the route guard in `app/`. See `docs/RBAC.md` for the permission catalog.

### 3.7 Forms, styling, states

- **Forms:** validate before submit; disable submit while pending; surface server errors near the field. Keep form state in the form component, not the page.
- **Styling:** TailwindCSS utilities + `shared/ui` primitives. **NEVER** copy-paste a styled block twice — extract a `shared/ui` component. Reuse the design tokens (see `docs/glassMorphism.md`).
- **Every screen MUST handle three states:** loading (skeleton/spinner), error (with retry), empty ("no data yet"). Don't render a blank page.
- **MUST** wrap routes in an `ErrorBoundary` so one render error doesn't white-screen the app.

### 3.8 Performance defaults (the anti-patterns to avoid)

These mirror real issues found in this codebase — don't reintroduce them:

- **MUST** memoize chart `data` and `options` with `useMemo` (never build them inline in render).
- **MUST** wrap list-row components in `React.memo` and pass handlers via `useCallback`.
- **NEVER** put a `setInterval(setState)` clock in a big component — isolate it in a tiny leaf so only it re-renders.
- **SHOULD** lazy-load non-landing routes (`React.lazy`) and dynamically import heavy libs (`xlsx`, `react-pdf`) at the point of use.
- **SHOULD** virtualize tables with hundreds+ of rows.
- **MUST** always clean up effects: `clearInterval`, `removeEventListener`, abort requests, `connection.stop()`.

### 3.9 Accessibility & testing (baseline)

- **MUST** give icon-only buttons an `aria-label`; use semantic `<button>`/`<a>` (not `<div onClick>`).
- **SHOULD** add a colocated test for hooks with logic and for utilities (`*.test.ts(x)`). Test the behavior, not the implementation.

---

## Part 4 — The Golden Path: add a new feature end-to-end

**Scenario:** add a "Certifications" screen showing an employee's certifications, with an "Add certification" action. Follow these steps for _any_ new feature.

### Step 1 — Create the slice

```
src/features/certifications/
├── api/        certifications.api.ts
├── hooks/      useCertifications.ts
├── types/      certification.types.ts
├── components/ CertificationTable.tsx, AddCertificationModal.tsx
└── pages/      CertificationsPage.tsx
```

### Step 2 — Types

```ts
// features/certifications/types/certification.types.ts
export interface Certification {
  id: string;
  employeeId: string;
  name: string;
  issuedOn: string; // ISO date
  expiresOn: string | null;
}
export interface AddCertificationDto {
  employeeId: string;
  name: string;
  issuedOn: string;
}
```

### Step 3 — API module (typed, via apiClient)

```ts
// features/certifications/api/certifications.api.ts
import { apiClient } from "@/shared/api/apiClient";
import type {
  Certification,
  AddCertificationDto,
} from "../types/certification.types";

export const certificationsApi = {
  list: (employeeId: string, signal?: AbortSignal) =>
    apiClient.get<Certification[]>(`/Certifications/${employeeId}`, { signal }),
  add: (dto: AddCertificationDto) =>
    apiClient.post<Certification>("/Certifications", dto),
};
```

### Step 4 — Data hooks (query + mutation)

```ts
// features/certifications/hooks/useCertifications.ts
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";
import { certificationsApi } from "../api/certifications.api";
import type { AddCertificationDto } from "../types/certification.types";

export function useCertifications(employeeId: string) {
  return useQuery({
    queryKey: ["certifications", employeeId],
    queryFn: ({ signal }) => certificationsApi.list(employeeId, signal),
    staleTime: 5 * 60_000,
  });
}

export function useAddCertification(employeeId: string) {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (dto: AddCertificationDto) => certificationsApi.add(dto),
    onSuccess: () =>
      qc.invalidateQueries({ queryKey: ["certifications", employeeId] }),
  });
}
```

### Step 5 — Presentational components

```tsx
// features/certifications/components/CertificationTable.tsx
import { memo } from "react";
import type { Certification } from "../types/certification.types";

interface Props {
  certifications: Certification[];
}

export const CertificationTable = memo(function CertificationTable({
  certifications,
}: Props) {
  return (
    <Table>
      <thead>
        <tr>
          <th>Name</th>
          <th>Issued</th>
          <th>Expires</th>
        </tr>
      </thead>
      <tbody>
        {certifications.map((c) => (
          <tr key={c.id}>
            <td>{c.name}</td>
            <td>{formatDate(c.issuedOn)}</td>
            <td>{c.expiresOn ? formatDate(c.expiresOn) : "—"}</td>
          </tr>
        ))}
      </tbody>
    </Table>
  );
});
```

### Step 6 — Page (orchestrator, thin)

```tsx
// features/certifications/pages/CertificationsPage.tsx
import { useState } from "react";
import { useCertifications } from "../hooks/useCertifications";
import { usePermission } from "@/permissions/hooks/usePermission";
import { CertificationTable } from "../components/CertificationTable";
import { AddCertificationModal } from "../components/AddCertificationModal";

/**
 * CertificationsPage
 * Employee certifications screen.
 * Responsibilities: orchestrates child components; holds no business logic.
 * Data: useCertifications (features/certifications/hooks)
 */
export function CertificationsPage({ employeeId }: { employeeId: string }) {
  const { data, isLoading, error } = useCertifications(employeeId);
  const { canManageCertifications } = usePermission();
  const [isAddOpen, setAddOpen] = useState(false);

  if (isLoading) return <Spinner />;
  if (error) return <ErrorState error={error} />;

  return (
    <PageLayout title="Certifications">
      {canManageCertifications && (
        <Button onClick={() => setAddOpen(true)}>Add certification</Button>
      )}
      {data?.length ? (
        <CertificationTable certifications={data} />
      ) : (
        <EmptyState message="No certifications yet" />
      )}
      {isAddOpen && (
        <AddCertificationModal
          employeeId={employeeId}
          onClose={() => setAddOpen(false)}
        />
      )}
    </PageLayout>
  );
}
```

### Step 7 — Wire the route

```tsx
// app/routes.tsx
const CertificationsPage = lazy(() =>
  import("@/features/certifications/pages/CertificationsPage")
    .then(m => ({ default: m.CertificationsPage }))
);
// …add inside the route table:
<Route path="/employees/:employeeId/certifications" element={<CertificationsPage … />} />
```

### Step 8 — Verify (your Definition of Done)

- `npx tsc` clean · `npm run build` clean · loading/error/empty states present · permission gate on the action · no `fetch` in the component · each component stays single-purpose.

**That's the whole pattern.** Every feature in this app should look exactly like this.

---

## Part 5 — Quick reference

### Do / Don't cheat-sheet

| ✅ Do                                   | ❌ Don't                               |
| --------------------------------------- | -------------------------------------- |
| `features/<domain>/…` for domain code   | dump it in flat `components/`          |
| query hook → api module → apiClient     | `fetch` inside a component             |
| `useMemo` derived values                | `useState`+`useEffect` to mirror state |
| memoize context value + handlers        | inline `value={{…}}` on a provider     |
| `React.memo` rows + `useCallback` props | inline arrow handlers into mapped rows |
| name by contents (`LeaveTable`)         | `Helpers`, `Model`, `Table2`, `Misc`   |
| small, single-purpose components        | 2,000-line dashboards doing everything |
| treat client gates as UX                | trust the client to protect data       |
| clean up every effect                   | leave intervals/listeners running      |

### Definition of Done (every PR)

- [ ] `npx tsc` passes; `npm run build` passes
- [ ] New code follows the folder + naming conventions
- [ ] Components are single-purpose (split when doing more than one job); no `fetch`/business logic in components
- [ ] Loading / error / empty states handled
- [ ] Permission-gated where relevant (and server-enforced)
- [ ] Effects clean up; no console noise; no `any` at API boundaries
- [ ] Reviewed by someone from another domain

### Where do I find…?

| I want…                   | Look in                                                                      |
| ------------------------- | ---------------------------------------------------------------------------- |
| A reusable UI primitive   | `shared/ui/`                                                                 |
| How auth/permissions work | `docs/RBAC.md` + `app/providers/`                                            |
| Product/feature overview  | `docs/EmployeePortal.md`                                                     |
| Data model / tables       | `docs/DbTables.md`                                                           |
| Design tokens / styling   | `docs/glassMorphism.md`                                                      |
| The refactor/perf roadmap | ADO Epics #397 (refactor) & #363 (performance), `docs/FRONTEND_WORK_PLAN.md` |

---

_Keep this guide current: when a convention changes, update this file in the same PR. A guide that lies is worse than no guide._
