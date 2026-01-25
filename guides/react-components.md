# React Conventions (Framework-Agnostic)

Use functional components and arrow functions.

## Definition and Typing

- Use `FC` types e.g. `FC<PropsWithChildren<SidebarProps>>`.
- Define prop types explicitly; prefer `interface` over `type` for props.
- Destructure props in the function signature for clarity.

## Tailwind + Styling

- Use Tailwind utility classes as the default.
- Use a `cn(...)` helper for class composition and Tailwind conflict resolution (classnames + tailwind-merge).

## Component Patterns

Common patterns (use intentionally, not by default):

- Compound components (static properties or object export), e.g. `Modal.Header`, `Sidebar.ContextProvider`.
- Suspense wrappers with skeleton fallbacks (for data fetches and lazy boundaries)
- `useTransition` for server-action calls from client components; show non-blocking pending UI
- `data-testid` attributes for stable test selectors in UI components
- if there is complex internal state management, consider using `useReducer` over `useState`
- use container + render props patterns for complex components, that expose internal state to children & allow custom rendering
- context providers for shared state (theming, modals, sidebars, etc.)
- use portals for modals, tooltips, dropdowns, and other overlay components
