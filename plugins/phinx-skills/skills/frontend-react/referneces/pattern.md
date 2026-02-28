# Advanced React Patterns Reference

## Custom Hook Patterns

### Data hook with full lifecycle
```tsx
function useApiData<T>(key: string[], fetcher: () => Promise<T>) {
  return useQuery({
    queryKey: key,
    queryFn: fetcher,
    staleTime: 5 * 60 * 1000,
    retry: (count, error) => count < 3 && !is404(error),
  });
}
```

### Debounced value hook
```tsx
function useDebouncedValue<T>(value: T, delay: number): T {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id);
  }, [value, delay]);
  return debounced;
}
```

### Media query hook
```tsx
function useMediaQuery(query: string): boolean {
  const [matches, setMatches] = useState(false);
  useEffect(() => {
    const mql = window.matchMedia(query);
    setMatches(mql.matches);
    const handler = (e: MediaQueryListEvent) => setMatches(e.matches);
    mql.addEventListener("change", handler);
    return () => mql.removeEventListener("change", handler);
  }, [query]);
  return matches;
}
```

## Next.js App Router Patterns

### Server Component data fetching
```tsx
// app/dashboard/page.tsx — Server Component (default)
async function DashboardPage() {
  const data = await fetchDashboardData(); // Direct async, no useEffect
  return <DashboardView data={data} />;
}
```

### Streaming with Suspense
```tsx
export default function Page() {
  return (
    <main>
      <h1>Dashboard</h1>
      <Suspense fallback={<MetricsSkeleton />}>
        <Metrics /> {/* Server Component, can be slow */}
      </Suspense>
      <Suspense fallback={<ChartSkeleton />}>
        <RevenueChart />
      </Suspense>
    </main>
  );
}
```

### Route handlers with validation
```tsx
// app/api/items/route.ts
import { z } from "zod";
import { NextRequest, NextResponse } from "next/server";

const CreateItemSchema = z.object({
  name: z.string().min(1).max(255),
  status: z.enum(["active", "archived"]),
});

export async function POST(req: NextRequest) {
  const body = await req.json();
  const result = CreateItemSchema.safeParse(body);
  if (!result.success) {
    return NextResponse.json(
      { error: "Validation failed", details: result.error.flatten() },
      { status: 400 }
    );
  }
  // proceed with result.data (fully typed)
}
```

## Form Patterns

### React Hook Form + Zod
```tsx
const schema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
});

type FormData = z.infer<typeof schema>;

function LoginForm() {
  const { register, handleSubmit, formState: { errors, isSubmitting } } =
    useForm<FormData>({ resolver: zodResolver(schema) });

  return (
    <form onSubmit={handleSubmit(onSubmit)} noValidate>
      <input {...register("email")} aria-invalid={!!errors.email} />
      {errors.email && <p role="alert">{errors.email.message}</p>}
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? "Signing in..." : "Sign in"}
      </button>
    </form>
  );
}
```

## State Machine Pattern (Complex UI)

For multi-step flows, modals with states, or async operations with distinct phases:

```tsx
type State =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: Result }
  | { status: "error"; error: Error; retryCount: number };

function reducer(state: State, action: Action): State {
  switch (state.status) {
    case "idle":
      if (action.type === "FETCH") return { status: "loading" };
      return state;
    case "loading":
      if (action.type === "SUCCESS") return { status: "success", data: action.data };
      if (action.type === "ERROR") return { status: "error", error: action.error, retryCount: 0 };
      return state;
    // exhaustive handling...
  }
}
```

## Optimistic Update Pattern

```tsx
const mutation = useMutation({
  mutationFn: updateItem,
  onMutate: async (newItem) => {
    await queryClient.cancelQueries({ queryKey: ["items"] });
    const previous = queryClient.getQueryData(["items"]);
    queryClient.setQueryData(["items"], (old) =>
      old.map((item) => (item.id === newItem.id ? { ...item, ...newItem } : item))
    );
    return { previous };
  },
  onError: (_err, _vars, context) => {
    queryClient.setQueryData(["items"], context?.previous);
  },
  onSettled: () => {
    queryClient.invalidateQueries({ queryKey: ["items"] });
  },
});
```

## Animation Patterns

### Staggered entry with Framer Motion
```tsx
const container = { hidden: {}, show: { transition: { staggerChildren: 0.05 } } };
const item = { hidden: { opacity: 0, y: 20 }, show: { opacity: 1, y: 0 } };

<motion.ul variants={container} initial="hidden" animate="show">
  {items.map((i) => <motion.li key={i.id} variants={item}>{i.name}</motion.li>)}
</motion.ul>
```

### Layout animations
```tsx
<LayoutGroup>
  {items.map((item) => (
    <motion.div key={item.id} layout layoutId={item.id}>
      {item.name}
    </motion.div>
  ))}
</LayoutGroup>
```