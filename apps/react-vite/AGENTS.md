# AGENTS.md — Bulletproof React (react-vite)

React 18.3.1 · Vite 5.2.10 · React Router 7.0.2 (lib mode)
TanStack React Query 5 · Zustand 4.5 (notifications only) · React Hook Form 7 + Zod 3
Radix UI · Tailwind CSS 3 · CVA · Axios · MSW 2

Path alias: `@/*` → `src/*`

---

## Directory Layout

```
src/
  app/           # router.tsx, provider.tsx, routes/ (file = route module)
  components/    # layouts/, errors/, ui/ (shared only — no feature logic here)
  config/        # env.ts (Zod-validated), paths.ts (single source of truth for routes)
  features/      # {discussions,comments,users,teams,auth}/api/ + components/
  lib/           # api-client.ts, auth.tsx, authorization.tsx, react-query.ts
  types/         # api.ts (treat as backend-generated — minimal edits)
  utils/         # cn.ts, format.ts
  hooks/         # shared hooks (e.g., use-disclosure.ts)
```

New features go in `src/features/{name}/`. Never put feature logic in `src/components/`.

---

## 3-Layer API Pattern (DO NOT DEVIATE)

Every data read uses three named exports in **one file** (e.g., `get-discussions.ts`):

```ts
// Layer 1 — raw async function, typed return
export const getDiscussions = (page = 1): Promise<{ data: Discussion[]; meta: Meta }> =>
  api.get('/discussions', { params: { page } });

// Layer 2 — queryOptions factory (MUST be separate export — used by route loaders)
export const getDiscussionsQueryOptions = ({ page }: { page?: number } = {}) =>
  queryOptions({ queryKey: ['discussions', { page }], queryFn: () => getDiscussions(page) });

// Layer 3 — custom hook (thin wrapper, accepts QueryConfig override)
export const useDiscussions = ({ page, queryConfig }: UseDiscussionsOptions) =>
  useQuery({ ...getDiscussionsQueryOptions({ page }), ...queryConfig });
```

**Why Layer 2 must be a separate export:** Route `clientLoader` functions call
`queryClient.getQueryData(query.queryKey) ?? queryClient.fetchQuery(query)` using the
factory directly. Collapsing layers breaks cache-warm prefetching at the router level.

- DO NOT use `useEffect` + `fetch` or SWR for data fetching.
- DO NOT call `queryClient` inside components — use the hook.
- DO NOT collapse the three layers into a single `useQuery` call inside a hook.

---

## Route Loaders (Waterfall Prevention)

Every data-fetching route exports a `clientLoader` that warms the cache before render:

```ts
export const clientLoader =
  (queryClient: QueryClient) =>
  async ({ request }: LoaderFunctionArgs) => {
    const query = getDiscussionsQueryOptions({ page });
    return queryClient.getQueryData(query.queryKey) ?? (await queryClient.fetchQuery(query));
  };
```

The router's `convert()` utility wires `clientLoader` → `loader` automatically.
Routes are lazy by default. Components still call their hooks — the loader just pre-warms the cache.
Suspense is **NOT** used for data fetching. Manual `isLoading` guards are intentional.

- DO NOT add data-fetching routes without a `clientLoader`.
- DO NOT use `React.lazy()` for component-level splitting — use the router's `lazy()` pattern.

---

## Mutations

Mutation files follow the same pattern as queries — one file per operation:

```ts
export const createDiscussionInputSchema = z.object({ title: z.string().min(1), body: z.string().min(1) });
export type CreateDiscussionInput = z.infer<typeof createDiscussionInputSchema>;

export const createDiscussion = ({ data }: { data: CreateDiscussionInput }): Promise<Discussion> =>
  api.post('/discussions', data);

export const useCreateDiscussion = ({ mutationConfig }: UseCreateDiscussionOptions = {}) => {
  const queryClient = useQueryClient();
  return useMutation({
    onSuccess: () => queryClient.invalidateQueries({ queryKey: getDiscussionsQueryOptions().queryKey }),
    ...mutationConfig,
    mutationFn: createDiscussion,
  });
};
```

All mutation inputs **must** have a Zod schema. Invalidate via `queryOptions().queryKey`, not strings.

---

## Form Pattern

Use the custom `<Form>` component only. Never use `useForm` directly in feature components.

```tsx
<Form id="create-discussion" onSubmit={(values) => mutation.mutate({ data: values })} schema={createDiscussionInputSchema}>
  {({ register, formState }) => (
    <>
      <Input label="Title" error={formState.errors['title']} registration={register('title')} />
      <Textarea label="Body" error={formState.errors['body']} registration={register('body')} />
    </>
  )}
</Form>
```

- `schema` MUST be a Zod `z.object(...)`. No yup, no manual validation.
- All form schemas live in the feature's API file (same file as the mutation).
- For forms in modals/drawers: use `<FormDrawer>`.

---

## Authorization (RBAC)

Two patterns — pick one per use case:

```tsx
// Role-based
<Authorization allowedRoles={[ROLES.ADMIN]}>{children}</Authorization>

// Policy-based (fine-grained — e.g., "owner or admin")
<Authorization policyCheck={POLICIES['comment:delete'](user, comment)}>{children}</Authorization>
```

`POLICIES` and `ROLES` are defined in `src/lib/authorization.tsx`.

- NEVER conditionally render protected UI with an `if/else` or ternary in JSX — use `<Authorization>`.
- NEVER mix `allowedRoles` and `policyCheck` on the same `<Authorization>` instance.

---

## UI Components

All shared UI lives in `src/components/ui/{name}/`. All variants use CVA:

```ts
const buttonVariants = cva('base-classes', {
  variants: { variant: { default: '...', destructive: '...' }, size: { default: '...', sm: '...' } },
  defaultVariants: { variant: 'default', size: 'default' },
});
```

- Use Radix UI primitives for interactive components (dialog, dropdown, switch, label).
- Use Tailwind for all styling. No CSS modules, no `style={{}}` for layout.
- Use `cn()` from `@/utils/cn` for conditional class merging.
- Use `lucide-react` for all icons.
- DO NOT add new UI libraries.
- DO NOT create `src/components/ui/` entries for feature-specific components.

---

## Hard Constraints

| Constraint | Detail |
|---|---|
| `api-client.ts` auto-unwraps `response.data` | Raw API functions return the payload directly — never access `.data` on the result |
| Zustand = notifications only | `useNotifications` is the only Zustand store. Do not add more. |
| No `React.memo` / `useMemo` / `useCallback` without measurement | These are absent by design. Do not add preemptively. |
| Global query defaults | `staleTime: 60s`, `retry: false`, `refetchOnWindowFocus: false` in `lib/react-query.ts`. Do not override per-query without a documented reason. |
| `paths.ts` is the only route source | Never hardcode `/app/discussions` or any path string in components. |
| `env.ts` owns env vars | Add new vars to the Zod schema there. Never access `import.meta.env` directly. |
| MSW mocks the API | Update `src/testing/mocks/handlers/` when adding new API endpoints. |
| `throwOnError` is commented out | Suspense-based data fetching is deliberately deferred. Do not enable. |
