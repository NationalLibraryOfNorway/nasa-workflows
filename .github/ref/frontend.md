# Frontend Reference

Framework-agnostic rules for browser-facing code. Apply on top of [`../AGENTS.md`](../AGENTS.md). For TS/React specifics see [`typescript.md`](typescript.md). For the API contract side, see [`backend-api.md`](backend-api.md).

## Architecture

- Separate concerns: rendering, state, data fetching, routing.
- Co-locate code by feature, not by technical layer (`/features/checkout/...` over `/components`, `/hooks`, `/utils` for everything).
- Lift state only as far as needed. Global state is a last resort, not a default.
- Server state (cache of API data) and client state (UI toggles, form drafts) are different. Use a server-state library (TanStack Query, SWR, RTK Query) for the former.

## Components

- Small, single-purpose. If a component does two unrelated things, split.
- Props are the API. Keep the surface narrow. Prefer composition (children, slots) over many boolean props.
- No prop drilling more than 2–3 levels — extract context or compose differently.
- Presentational vs. container split is a guideline, not a rule. Use it when it clarifies; ignore when it doesn't.

## State management

- Local state first (`useState` / signals).
- Lift to a parent when siblings need it.
- Context for cross-cutting (theme, auth, locale) — not for high-frequency updates (causes re-renders).
- Reach for Redux / Zustand / Pinia / a store only when justified by app size and team familiarity.

## Data fetching

- Don't `fetch` in random `useEffect`s. Use a query library so you get caching, dedup, retries, and loading/error states for free.
- Handle the four states explicitly: loading, empty, error, success. Don't render `undefined.foo`.
- Mutations: optimistic updates with rollback on failure; invalidate related queries.
- Validate API responses at the boundary (`zod`, `valibot`) — don't trust shape just because TypeScript says so.

## Forms

- Controlled inputs by default. Uncontrolled is fine for simple cases or perf.
- Use a form library (React Hook Form, Formik, Felte) for non-trivial forms — validation, dirty tracking, submission.
- Validate on blur or submit, not on every keystroke (annoying).
- Show server errors next to the field, not just a banner.

## Accessibility (a11y)

- Semantic HTML first. `<button>` for actions, `<a>` for navigation. Don't `<div onClick>` interactive elements.
- Every form control has a `<label>` (visible or `aria-label`).
- Keyboard navigable: tab order makes sense, focus visible, Esc closes modals.
- Color contrast meets WCAG AA. Don't convey state by color alone.
- Test with a screen reader at least once for new interactive features.
- Respect `prefers-reduced-motion`.

## Performance

- Measure before optimizing. Use the Performance panel and Web Vitals (LCP, INP, CLS).
- Code-split by route. Lazy-load heavy components (`React.lazy`, dynamic `import()`).
- Images: correct `width`/`height` to prevent CLS, `loading="lazy"` below the fold, modern formats (WebP/AVIF), `srcset` for responsive.
- Avoid re-renders: stable refs for callbacks/objects passed to memoized children.
- Bundle: watch with `source-map-explorer` / `vite-bundle-visualizer`. Tree-shake; avoid full lodash imports.
- Network: cache, compress (br/gzip), HTTP/2+, fewer round trips.

## Styling

- Pick one approach per project: CSS Modules, Tailwind, CSS-in-JS (Emotion/Styled), or vanilla. Don't mix without reason.
- Design tokens (color, spacing, type) in one place. No magic hex codes scattered through components.
- Mobile-first responsive. Use `rem`/`em` for type, `clamp()` for fluid sizes.
- Don't fight the browser — use system focus rings, native `<dialog>`, etc., when they fit.

## Routing

- File-based or config-based — match the framework convention.
- Code-split per route by default.
- Reflect important state in the URL (filters, tabs, pagination) so it's shareable and bookmarkable.

## i18n / l10n

- Externalize all user-visible strings from day one if i18n is on the roadmap. Retrofitting is painful.
- Don't concatenate translated fragments — use ICU MessageFormat or equivalent for plurals/genders.
- Format dates/numbers with `Intl.*`, not hand-rolled.

## Security

- Treat all backend data as untrusted. Escape on render. React/Vue/Svelte do this by default — don't bypass with `dangerouslySetInnerHTML` / `v-html` on untrusted input.
- Sanitize HTML (DOMPurify) when you must render rich text.
- Auth tokens: prefer HttpOnly cookies over `localStorage` (XSS-exposed).
- CSP headers from the backend; report violations.
- No secrets in frontend code or env vars prefixed for the client (`VITE_*`, `NEXT_PUBLIC_*`) — they're shipped to the browser.

## Testing

- **Unit**: pure logic, hooks (with `@testing-library/react-hooks` or framework equivalent).
- **Component**: Testing Library — query by role/label, not by class/id. Test behavior, not implementation.
- **E2E**: Playwright (preferred) or Cypress for critical user flows. Don't E2E what a unit/component test can cover.
- Mock the network at the boundary (MSW). Don't mock your own modules.
- Visual regression: Chromatic / Percy for design-system-heavy projects.

## Common pitfalls

- Effect dependency arrays missing values — enable `react-hooks/exhaustive-deps`.
- Updating state in render → infinite loop.
- Treating `null`/`undefined`/empty array as the same — they often aren't.
- Memoizing everything "to be safe" — `useMemo`/`useCallback` have their own cost.
- Hidden `Date` timezone bugs — display in user TZ, store/transport in UTC.
- Forgetting `key` on lists, or using array index as key for reorderable lists.
- Shipping dev-only code (mock data, debug logs) to production — gate by `NODE_ENV` and verify.
