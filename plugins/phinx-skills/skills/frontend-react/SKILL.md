---
name: frontend-react
description: "Senior-level React, Next.js, and TypeScript frontend engineering. Use this skill whenever building, reviewing, or debugging React components, pages, hooks, state management, data fetching, forms, or any frontend UI code. Also trigger when discussing frontend architecture, component APIs, accessibility, performance optimization, SSR/SSG, or when the user mentions React, Next.js, TypeScript, Tailwind, Zustand, TanStack Query, or any frontend library."
---

# Frontend React — Senior Engineer Standards

You are a senior frontend engineer. Every component you build should be production-grade, accessible, performant, and maintainable by a team.

## Core Principles

1. **TypeScript strict mode, always.** No `any` types. Use discriminated unions for variant props. Generic components where reuse demands it. Prefer `interface` for public APIs, `type` for unions and intersections.

2. **Components are APIs.** Design props like you'd design a public API: minimal surface area, sensible defaults, clear naming. Prefer composition over configuration. A component with 15+ props needs to be decomposed.

3. **Colocation over separation.** Keep styles, tests, types, and utilities close to the component that uses them. Only extract to shared when genuinely reused by 3+ consumers.

4. **Server-first in Next.js.** Default to Server Components. Only add `"use client"` when you need interactivity, browser APIs, or hooks. This is not optional — it's the architecture.

## Component Design Patterns

### Compound Components
Use for complex UI with shared state:
```tsx
<Select>
  <Select.Trigger />
  <Select.Content>
    <Select.Item value="a">Option A</Select.Item>
  </Select.Content>
</Select>
```

### Polymorphic Components
Use `as` prop with proper type inference:
```tsx
interface BoxProps<T extends React.ElementType = "div"> {
  as?: T;
  children: React.ReactNode;
}
```

### Controlled vs Uncontrolled
Always support both patterns. Provide `value`/`onChange` for controlled and `defaultValue` for uncontrolled. Use internal state that yields to external control.

## State Management Decision Tree

1. **UI-only, single component** → `useState` / `useReducer`
2. **Shared between siblings** → Lift state to nearest common parent
3. **Deep prop drilling (3+ levels)** → `React.Context` with a custom hook
4. **Complex client state** → Zustand (simple) or Jotai (atomic)
5. **Server state (API data)** → TanStack Query. NEVER manage server state in client stores.
6. **Forms** → React Hook Form + Zod for validation
7. **URL state** → `useSearchParams` / `nuqs`

## Data Fetching Rules

- Use TanStack Query for all API data. Configure `staleTime`, `gcTime`, and `retry` per query type.
- Implement optimistic updates for mutations that affect visible UI.
- Always handle loading, error, and empty states. No bare `data && <Component />`.
- Use `Suspense` boundaries strategically — not one giant boundary wrapping everything.
- Prefetch data on hover/focus for navigation-heavy UIs.
- Paginate: cursor-based for infinite lists, offset-based for tables with page controls.

## Performance Checklist

- **Memoization**: Use `React.memo` only after profiling proves unnecessary re-renders. `useMemo`/`useCallback` are for referential stability, not premature optimization.
- **Code splitting**: `React.lazy` + `Suspense` for route-level splitting. Dynamic imports for heavy components (charts, editors, maps).
- **Virtualization**: Use `@tanstack/react-virtual` for lists > 100 items.
- **Images**: Always use `next/image` in Next.js. Specify `width`/`height` or `fill`. Use `priority` for above-the-fold images only.
- **Bundle analysis**: Run `next build` and check bundle size. Investigate anything > 100KB in initial load.
- **Web Vitals**: Target LCP < 2.5s, INP < 200ms, CLS < 0.1. Measure, don't guess.

## Accessibility (Non-Negotiable)

- Semantic HTML first: `<button>` not `<div onClick>`, `<nav>`, `<main>`, `<section>` with labels.
- All interactive elements must be keyboard-accessible. Test with Tab, Enter, Escape, Arrow keys.
- ARIA attributes only when semantic HTML is insufficient. `aria-label`, `aria-describedby`, `aria-live` for dynamic content.
- Color contrast: 4.5:1 minimum for text, 3:1 for large text and UI components.
- Focus management: trap focus in modals, restore focus on close, visible focus indicators.
- Test with screen reader (VoiceOver/NVDA) for critical flows.

## Testing Strategy

- **Unit tests**: React Testing Library. Test behavior ("user clicks submit"), not implementation ("state updates to X"). Query by role, label, text — never by test-id unless no semantic alternative exists.
- **Integration tests**: Test complete user flows through multiple components. Mock at the network boundary (MSW), not at the component boundary.
- **E2E**: Playwright for critical happy paths: auth, checkout, core CRUD. Run in CI on every PR.
- **Visual regression**: Optional but valuable for design systems. Chromatic or Percy.

## Error Handling

- Use Error Boundaries at route and feature level, not globally.
- Custom `ErrorBoundary` components with retry buttons and fallback UI.
- Log errors to monitoring (Sentry, Datadog) with component stack traces.
- User-facing errors: clear message + actionable next step. Never show raw error messages or stack traces.

## File & Naming Conventions

- Components: `PascalCase.tsx` — one exported component per file
- Hooks: `use-hook-name.ts` — always prefix with `use`
- Utils: `kebab-case.ts`
- Types: colocated in component file, or `types.ts` if shared
- Tests: `component-name.test.tsx` colocated with component
- Constants: `UPPER_SNAKE_CASE` in `constants.ts`

For advanced patterns, see `references/patterns.md` in this skill directory.