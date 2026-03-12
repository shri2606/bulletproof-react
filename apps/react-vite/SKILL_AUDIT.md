# Skill Audit: vercel-react-best-practices @ Bulletproof React

**Skill:** `vercel-labs/agent-skills` → `vercel-react-best-practices`
**App:** `apps/react-vite` — React 18 + Vite 5 + React Router 7 (client-side SPA)

> Rule files are in `.agents/skills/vercel-react-best-practices/rules/`.
> This document records findings only — not rule definitions.

---

## Verdicts at a Glance

| Rule | Verdict |
|------|---------|
| `async-parallel` | AGREE |
| `bundle-preload` | AGREE |
| `bundle-dynamic-imports` | REJECTED |
| `bundle-barrel-imports` | REJECTED |
| `rerender-functional-setstate` | ATTEMPTED → REVERTED |
| `rendering-conditional-render` | REJECTED |
| `async-suspense-boundaries` | REJECTED |
| `rerender-memo` | REJECTED |
| `server-*` (7 rules) | NOT APPLICABLE |
| Anonymous `ForwardRef` in DevTools | NOT ACTIONABLE |

---

## AGREE

**`async-parallel`** — `discussion.tsx` clientLoader already fetches discussion + comments via `Promise.all`. No change needed.

**`bundle-preload`** — `discussions-list.tsx` prefetches discussion data and comments on `onMouseEnter`. Already in place at two levels.

---

## ATTEMPTED → REVERTED

**`rerender-functional-setstate`** — Tried removing the arrow wrapper in `comments-list.tsx:82`:
```tsx
// attempted
<Button onClick={commentsQuery.fetchNextPage}>
```
TypeScript TS2322: `fetchNextPage` expects `FetchNextPageOptions | undefined`, but `onClick` passes `MouseEvent`. The arrow function is a **type adapter**, not a redundant closure. Reverted.

**Learning:** The rule is valid in principle. TypeScript surfaced that this instance is load-bearing. Only visible when you actually attempt the change.

---

## REJECTED

**`bundle-dynamic-imports`** — `MDPreview` (marked + dompurify) is statically imported, but the entire discussion route is already lazy-loaded at the router level. No initial bundle impact. Adding component-level `React.lazy()` adds Suspense complexity for no gain.

**`bundle-barrel-imports`** — Architecture intentionally uses barrel files (`@/components/ui/form`, `@/components/layouts`, etc.) for clean import paths. Vite + ESM handles tree-shaking correctly regardless. A large noisy refactor with no measurable benefit in this setup.

**`rendering-conditional-render`** — `comment.author` in `comments-list.tsx:65` uses `&&`. The rule guards against accidentally rendering `0` or `""`, but `comment.author` is typed `Author | undefined` — an object type. TypeScript strict mode makes the risk impossible at this call site.

**`async-suspense-boundaries`** — `throwOnError: true` is commented out in `lib/react-query.ts` deliberately. All 6+ data consumers use manual `isLoading` guards. Enabling Suspense would require coordinated changes across all consumers + MSW test setup changes. The manual pattern is intentional.

**`rerender-memo`** — No `React.memo` anywhere in the codebase by design. Comment lists are paginated (≤10 items). No measured re-render problem exists.

---

## NOT APPLICABLE

**`server-*` rules** (7 rules: `server-cache-react`, `server-parallel-fetching`, `server-auth-actions`, etc.) — This is a Vite client-side SPA with MSW mocking. No RSC, no server actions, no server-side fetch. These rules target Next.js App Router apps.

---

## NOT ACTIONABLE

**Anonymous `ForwardRef` in React DevTools** — Traced the full component path in DevTools: `CreateDiscussion → ... → Button → Plus → Anonymous (ForwardRef)`. `Button.displayName` is already set. The anonymous node is inside lucide-react's internal SVG `forwardRef` wrapper — a third-party library. Not fixable in this codebase without patching `node_modules`.

---

## Key Takeaway

The most dangerous rule was `async-suspense-boundaries`. Without `AGENTS.md`, an agent would have uncommented `throwOnError`, wrapped components in Suspense, and removed all `isLoading` guards — breaking the MSW test setup in the process.

`AGENTS.md` encoded the constraint explicitly, blocking that path before the skill could suggest it. This matches Vercel's own eval finding: project-specific context in `AGENTS.md` outperforms skills alone for exactly this reason.
