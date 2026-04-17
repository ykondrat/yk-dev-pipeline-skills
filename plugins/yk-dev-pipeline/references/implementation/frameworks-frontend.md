# Frontend Framework Patterns — Deep Reference

Frontend patterns for React, Next.js (App Router), Vue 3, and state management.
Load this reference when the project includes a frontend or full-stack application.

---

## 1. React — Component Patterns

### Component Structure

```typescript
// Prefer function components with explicit prop types
interface UserCardProps {
  user: User;
  onEdit?: (id: string) => void;
  variant?: "compact" | "full";
}

export function UserCard({ user, onEdit, variant = "full" }: UserCardProps) {
  return (
    <article className={styles[variant]}>
      <h3>{user.name}</h3>
      {variant === "full" && <p>{user.bio}</p>}
      {onEdit && (
        <button type="button" onClick={() => onEdit(user.id)}>
          Edit
        </button>
      )}
    </article>
  );
}
```

### Composition Over Prop Drilling

```typescript
// BAD — prop drilling through 3+ levels
<Layout user={user}>
  <Sidebar user={user}>
    <UserMenu user={user} />
  </Sidebar>
</Layout>

// GOOD — composition with children
<Layout>
  <Sidebar>
    <UserMenu user={user} />
  </Sidebar>
</Layout>

// GOOD — compound components for related UI
<Tabs defaultValue="profile">
  <Tabs.List>
    <Tabs.Trigger value="profile">Profile</Tabs.Trigger>
    <Tabs.Trigger value="settings">Settings</Tabs.Trigger>
  </Tabs.List>
  <Tabs.Content value="profile"><ProfilePanel /></Tabs.Content>
  <Tabs.Content value="settings"><SettingsPanel /></Tabs.Content>
</Tabs>
```

### Container / Presenter Split

```typescript
// Container — handles data, no UI
function UserListContainer() {
  const { data, isLoading, error } = useUsers();

  if (isLoading) return <Skeleton count={5} />;
  if (error) return <ErrorMessage error={error} />;

  return <UserList users={data} />;
}

// Presenter — pure UI, easy to test and storybook
interface UserListProps {
  users: User[];
}

export function UserList({ users }: UserListProps) {
  if (users.length === 0) return <EmptyState message="No users found" />;

  return (
    <ul>
      {users.map((user) => (
        <li key={user.id}>
          <UserCard user={user} />
        </li>
      ))}
    </ul>
  );
}
```

---

## 2. React — Hook Patterns

### Data Fetching with React Query / TanStack Query

```typescript
// hooks/use-users.ts
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";
import { api } from "@/lib/api";

export function useUsers(filters?: UserFilters) {
  return useQuery({
    queryKey: ["users", filters],
    queryFn: () => api.users.list(filters),
    staleTime: 5 * 60 * 1000, // 5 minutes
  });
}

export function useCreateUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: api.users.create,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["users"] });
    },
  });
}

// Usage in component
function CreateUserForm() {
  const { mutate, isPending, error } = useCreateUser();

  function handleSubmit(data: CreateUserInput) {
    mutate(data);
  }

  return <Form onSubmit={handleSubmit} disabled={isPending} error={error} />;
}
```

### Custom Hooks

```typescript
// hooks/use-debounce.ts — Generic debounce hook
import { useState, useEffect } from "react";

export function useDebounce<T>(value: T, delay: number): T {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debounced;
}
```

```typescript
// hooks/use-local-storage.ts — Persistent state
import { useState, useCallback } from "react";

export function useLocalStorage<T>(key: string, initialValue: T) {
  const [stored, setStored] = useState<T>(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? (JSON.parse(item) as T) : initialValue;
    } catch {
      return initialValue;
    }
  });

  const setValue = useCallback(
    (value: T | ((prev: T) => T)) => {
      setStored((prev) => {
        const next = value instanceof Function ? value(prev) : value;
        window.localStorage.setItem(key, JSON.stringify(next));
        return next;
      });
    },
    [key],
  );

  return [stored, setValue] as const;
}
```

### Hook Rules Recap
- Only call hooks at the top level (not in conditions, loops, or nested functions)
- Only call hooks from React function components or custom hooks
- Custom hooks must start with `use`
- Dependency arrays must be exhaustive — use ESLint `react-hooks/exhaustive-deps`
- Prefer `useCallback` and `useMemo` only when passing to memoized children or expensive computations — don't over-memoize

---

## 3. React — Form Patterns

### React Hook Form + Zod

```typescript
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";

const schema = z.object({
  email: z.string().email("Invalid email"),
  password: z.string().min(8, "At least 8 characters"),
  role: z.enum(["admin", "user"]),
});

type FormData = z.infer<typeof schema>;

export function LoginForm({ onSubmit }: { onSubmit: (data: FormData) => void }) {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<FormData>({
    resolver: zodResolver(schema),
    defaultValues: { role: "user" },
  });

  return (
    <form onSubmit={handleSubmit(onSubmit)} noValidate>
      <div>
        <label htmlFor="email">Email</label>
        <input id="email" type="email" {...register("email")} aria-invalid={!!errors.email} />
        {errors.email && <p role="alert">{errors.email.message}</p>}
      </div>

      <div>
        <label htmlFor="password">Password</label>
        <input id="password" type="password" {...register("password")} />
        {errors.password && <p role="alert">{errors.password.message}</p>}
      </div>

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? "Signing in..." : "Sign in"}
      </button>
    </form>
  );
}
```

### Form Key Rules
- Always use `noValidate` on `<form>` when using JS validation (prevents browser popups)
- Always associate labels with inputs (`htmlFor`/`id`)
- Use `aria-invalid` and `role="alert"` for accessible error messages
- Disable submit button during submission to prevent double-submits
- Use Zod schemas shared between frontend validation and API validation

---

## 4. Next.js App Router — Frontend Patterns

### Server vs Client Components

```typescript
// app/users/page.tsx — Server Component (default, no "use client")
// Can: fetch data, access DB, read files, use async/await
// Cannot: use hooks, event handlers, browser APIs
import { UserList } from "@/components/user-list";
import { getUsers } from "@/lib/services/user.service";

export default async function UsersPage() {
  const users = await getUsers();

  return (
    <main>
      <h1>Users</h1>
      <UserList users={users} />
    </main>
  );
}
```

```typescript
// components/user-search.tsx — Client Component (needs interactivity)
"use client";

import { useState } from "react";
import { useDebounce } from "@/hooks/use-debounce";
import { useUsers } from "@/hooks/use-users";

export function UserSearch() {
  const [query, setQuery] = useState("");
  const debouncedQuery = useDebounce(query, 300);
  const { data } = useUsers({ search: debouncedQuery });

  return (
    <div>
      <input
        type="search"
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search users..."
      />
      {data?.map((user) => <UserCard key={user.id} user={user} />)}
    </div>
  );
}
```

### Server Actions

```typescript
// app/users/actions.ts
"use server";

import { revalidatePath } from "next/cache";
import { z } from "zod";
import { createUser } from "@/lib/services/user.service";

const schema = z.object({
  email: z.string().email(),
  name: z.string().min(1),
});

export async function createUserAction(formData: FormData) {
  const parsed = schema.safeParse(Object.fromEntries(formData));

  if (!parsed.success) {
    return { error: parsed.error.flatten() };
  }

  await createUser(parsed.data);
  revalidatePath("/users");
}
```

```typescript
// components/create-user-form.tsx
"use client";

import { useActionState } from "react";
import { createUserAction } from "@/app/users/actions";

export function CreateUserForm() {
  const [state, formAction, isPending] = useActionState(createUserAction, null);

  return (
    <form action={formAction}>
      <input name="email" type="email" required />
      <input name="name" required />
      {state?.error && <p role="alert">Validation failed</p>}
      <button type="submit" disabled={isPending}>Create</button>
    </form>
  );
}
```

### Layout Pattern

```typescript
// app/layout.tsx — Root layout (required)
import type { ReactNode } from "react";
import { Providers } from "@/components/providers";

export default function RootLayout({ children }: { children: ReactNode }) {
  return (
    <html lang="en">
      <body>
        <Providers>
          <header><Nav /></header>
          <main>{children}</main>
          <footer><Footer /></footer>
        </Providers>
      </body>
    </html>
  );
}

// app/(dashboard)/layout.tsx — Nested layout for authenticated routes
export default function DashboardLayout({ children }: { children: ReactNode }) {
  return (
    <div className="dashboard">
      <Sidebar />
      <div className="content">{children}</div>
    </div>
  );
}
```

### Loading & Error States

```typescript
// app/users/loading.tsx — Streaming loading UI
export default function Loading() {
  return <UserListSkeleton />;
}

// app/users/error.tsx — Error boundary
"use client";

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <div role="alert">
      <h2>Something went wrong</h2>
      <p>{error.message}</p>
      <button onClick={reset}>Try again</button>
    </div>
  );
}

// app/users/not-found.tsx
export default function NotFound() {
  return <p>User not found</p>;
}
```

### Next.js Key Rules
- **Default to Server Components** — Only add `"use client"` when you need hooks, event handlers, or browser APIs.
- **Push client boundaries down** — Keep `"use client"` on the smallest possible component, not on entire pages.
- **Don't fetch in Client Components** — Fetch in Server Components or use TanStack Query in Client Components for client-side data.
- **Use `loading.tsx` and `error.tsx`** — File-based loading and error states with Suspense boundaries.
- **Server Actions for mutations** — Prefer over API routes for form submissions in full-stack Next.js apps.
- **`revalidatePath` / `revalidateTag`** — Invalidate cached data after mutations.

---

## 5. Vue 3 — Composition API Patterns

### Component Structure

```vue
<!-- components/UserCard.vue -->
<script setup lang="ts">
interface Props {
  user: User;
  variant?: "compact" | "full";
}

const props = withDefaults(defineProps<Props>(), {
  variant: "full",
});

const emit = defineEmits<{
  edit: [id: string];
}>();
</script>

<template>
  <article :class="$style[variant]">
    <h3>{{ user.name }}</h3>
    <p v-if="variant === 'full'">{{ user.bio }}</p>
    <button v-if="$attrs.onEdit" @click="emit('edit', user.id)">Edit</button>
  </article>
</template>

<style module>
.compact { /* ... */ }
.full { /* ... */ }
</style>
```

### Composables (Vue Custom Hooks)

```typescript
// composables/useUsers.ts
import { ref, computed, watch } from "vue";
import type { Ref } from "vue";
import { api } from "@/lib/api";

export function useUsers(filters?: Ref<UserFilters>) {
  const data = ref<User[]>([]);
  const isLoading = ref(false);
  const error = ref<Error | null>(null);

  async function fetch() {
    isLoading.value = true;
    error.value = null;
    try {
      data.value = await api.users.list(filters?.value);
    } catch (e) {
      error.value = e instanceof Error ? e : new Error(String(e));
    } finally {
      isLoading.value = false;
    }
  }

  // Refetch when filters change
  if (filters) {
    watch(filters, fetch, { deep: true });
  }

  fetch();

  return { data: computed(() => data.value), isLoading, error, refetch: fetch };
}
```

```typescript
// composables/useDebounce.ts
import { ref, watch, type Ref } from "vue";

export function useDebounce<T>(source: Ref<T>, delay: number): Ref<T> {
  const debounced = ref(source.value) as Ref<T>;

  let timer: ReturnType<typeof setTimeout>;
  watch(source, (val) => {
    clearTimeout(timer);
    timer = setTimeout(() => {
      debounced.value = val;
    }, delay);
  });

  return debounced;
}
```

### Vue Router Integration

```typescript
// router/index.ts
import { createRouter, createWebHistory } from "vue-router";
import { useAuthStore } from "@/stores/auth";

const router = createRouter({
  history: createWebHistory(),
  routes: [
    { path: "/", component: () => import("@/views/Home.vue") },
    {
      path: "/dashboard",
      component: () => import("@/views/Dashboard.vue"),
      meta: { requiresAuth: true },
    },
    { path: "/:pathMatch(.*)*", component: () => import("@/views/NotFound.vue") },
  ],
});

router.beforeEach((to) => {
  if (to.meta.requiresAuth) {
    const auth = useAuthStore();
    if (!auth.isAuthenticated) {
      return { path: "/login", query: { redirect: to.fullPath } };
    }
  }
});

export default router;
```

### Vue 3 Key Rules
- **Use `<script setup>` always** — Simpler syntax, better type inference, no need for `defineComponent()`.
- **`defineProps` with TypeScript generics** — `defineProps<Props>()` for type-safe props.
- **Composables for shared logic** — Name them `use*`, return refs and computed values. They're Vue's equivalent of React hooks.
- **`ref` vs `reactive`** — Prefer `ref` for primitives and `reactive` for objects. `ref` is more explicit and works everywhere.
- **Lazy-load routes** — Use `() => import()` for code splitting in Vue Router.
- **Use Pinia for global state** — Don't use Vuex. Pinia is the official store for Vue 3.

---

## 6. State Management

### Zustand (React — Recommended for Most Apps)

```typescript
// stores/auth.ts
import { create } from "zustand";
import { persist } from "zustand/middleware";

interface AuthState {
  user: User | null;
  token: string | null;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
}

export const useAuthStore = create<AuthState>()(
  persist(
    (set) => ({
      user: null,
      token: null,
      login: async (email, password) => {
        const { user, token } = await api.auth.login({ email, password });
        set({ user, token });
      },
      logout: () => set({ user: null, token: null }),
    }),
    { name: "auth-storage" },
  ),
);

// Usage — no provider needed
function Header() {
  const user = useAuthStore((s) => s.user);
  const logout = useAuthStore((s) => s.logout);
  // ...
}
```

### Pinia (Vue — Official Store)

```typescript
// stores/auth.ts
import { defineStore } from "pinia";
import { ref, computed } from "vue";
import { api } from "@/lib/api";

export const useAuthStore = defineStore("auth", () => {
  const user = ref<User | null>(null);
  const token = ref<string | null>(null);
  const isAuthenticated = computed(() => !!token.value);

  async function login(email: string, password: string) {
    const result = await api.auth.login({ email, password });
    user.value = result.user;
    token.value = result.token;
  }

  function logout() {
    user.value = null;
    token.value = null;
  }

  return { user, token, isAuthenticated, login, logout };
});
```

### When to Use What

| Need | Solution |
|------|----------|
| Server state (API data) | TanStack Query (React) / composable + fetch (Vue) |
| Simple local state | `useState` (React) / `ref` (Vue) |
| Shared UI state (theme, sidebar) | Zustand (React) / Pinia (Vue) |
| Complex form state | React Hook Form (React) / VeeValidate (Vue) |
| Auth/session state | Zustand + persist (React) / Pinia + plugin (Vue) |
| URL state (filters, pagination) | URL search params — `nuqs` (React) / Vue Router query |

**Key rule**: Don't put server state in client stores. Use TanStack Query or equivalent. Client stores are for UI state and auth only.

---

## 7. Testing Frontend Components

### React Testing Library

```typescript
import { render, screen, waitFor } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { QueryClientProvider, QueryClient } from "@tanstack/react-query";
import { CreateUserForm } from "./create-user-form";

function renderWithProviders(ui: React.ReactElement) {
  const client = new QueryClient({
    defaultOptions: { queries: { retry: false } },
  });
  return render(<QueryClientProvider client={client}>{ui}</QueryClientProvider>);
}

describe("CreateUserForm", () => {
  it("shows validation errors on empty submit", async () => {
    const user = userEvent.setup();
    renderWithProviders(<CreateUserForm />);

    await user.click(screen.getByRole("button", { name: /create/i }));

    expect(await screen.findByText(/invalid email/i)).toBeInTheDocument();
  });

  it("submits valid data", async () => {
    const user = userEvent.setup();
    renderWithProviders(<CreateUserForm />);

    await user.type(screen.getByLabelText(/email/i), "test@example.com");
    await user.type(screen.getByLabelText(/name/i), "Test User");
    await user.click(screen.getByRole("button", { name: /create/i }));

    await waitFor(() => {
      expect(screen.queryByRole("alert")).not.toBeInTheDocument();
    });
  });
});
```

### Vue Testing Library

```typescript
import { render, screen } from "@testing-library/vue";
import userEvent from "@testing-library/user-event";
import { createTestingPinia } from "@pinia/testing";
import UserCard from "./UserCard.vue";

describe("UserCard", () => {
  it("renders user name", () => {
    render(UserCard, {
      props: { user: { id: "1", name: "Jane", bio: "Dev" } },
    });

    expect(screen.getByText("Jane")).toBeInTheDocument();
  });

  it("emits edit event on button click", async () => {
    const user = userEvent.setup();
    const { emitted } = render(UserCard, {
      props: { user: { id: "1", name: "Jane", bio: "Dev" } },
      attrs: { onEdit: () => {} }, // Make button visible
    });

    await user.click(screen.getByRole("button", { name: /edit/i }));

    expect(emitted().edit[0]).toEqual(["1"]);
  });
});
```

### MSW (Mock Service Worker) for API Mocking

```typescript
// mocks/handlers.ts
import { http, HttpResponse } from "msw";

export const handlers = [
  http.get("/api/v1/users", () => {
    return HttpResponse.json({
      data: [
        { id: "1", name: "Alice", email: "alice@example.com" },
        { id: "2", name: "Bob", email: "bob@example.com" },
      ],
    });
  }),

  http.post("/api/v1/users", async ({ request }) => {
    const body = await request.json();
    return HttpResponse.json(
      { data: { id: "3", ...body } },
      { status: 201 },
    );
  }),
];

// mocks/server.ts
import { setupServer } from "msw/node";
import { handlers } from "./handlers";

export const server = setupServer(...handlers);

// vitest.setup.ts
import { server } from "./mocks/server";

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

### Frontend Testing Key Rules
- **Query by role, label, text** — Not by test IDs or CSS classes. This tests what users actually see.
- **Use `userEvent` over `fireEvent`** — `userEvent` simulates real user interactions (typing, clicking) more accurately.
- **Mock at the network layer (MSW)** — Not at the module level. Tests stay realistic without hitting real APIs.
- **Wrap with providers** — Components using context/stores need providers in tests. Create a `renderWithProviders` helper.
- **Don't test implementation details** — Don't assert on state values or internal method calls. Assert on visible output.

---

## 8. Accessibility Checklist for Frontend

- Every `<img>` has `alt` text (empty `alt=""` for decorative images)
- Every form input has a `<label>` (use `htmlFor`/`for`)
- Interactive elements are focusable and have visible focus styles
- Color is not the sole means of conveying information
- Use semantic HTML (`<nav>`, `<main>`, `<button>`, `<article>`) over generic `<div>`
- Modals trap focus and restore it on close
- Error messages use `role="alert"` or `aria-live="polite"`
- Page has a single `<h1>`, headings are in order (`h1` → `h2` → `h3`)
- Skip navigation link for keyboard users
- Minimum contrast ratio: 4.5:1 for normal text, 3:1 for large text
