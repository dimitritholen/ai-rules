# React 19 Production Rules

## 1. Core React 19 Features & Breaking Changes

### Actions & Form Handling

**MUST** use Actions for all async data mutations—pass async functions to `action` or `formAction` props on `<form>`, `<input>`, and `<button>` elements. Actions automatically handle pending states, errors, optimistic updates, and form resets.

**MUST** use `useActionState` instead of manually combining `useState` + `useTransition` for form state management. Returns `[state, action, isPending]` tuple with automatic state handling.

**MUST** call `useFormStatus` only from child components rendered inside `<form>`—it returns status for the immediate parent form only, not nested forms or sibling components.

**MUST** use `useOptimistic` for immediate UI feedback before server confirmation (likes, votes, comments). Optimistic state automatically reverts to actual state when action completes or errors.

**NEVER** create Promises inside Client Components when using the `use` hook—Promises recreate on every render. Create Promises in Server Components and pass to Client Components as props.

### use Hook

**MUST** leverage the `use` hook's unique ability to be called conditionally, inside loops, and after early returns—unlike all other React hooks. Use for reading Promises or Context when conditional logic is required.

**PREFER** `async/await` over `use` hook in Server Components for data fetching—`async/await` is the standard pattern for Server Components, while `use` is primarily for Client Components reading resources.

### Refs & forwardRef

**NEVER** use `forwardRef` for new React 19 code—refs are now accepted as regular props. Use `ref?: Ref<T>` in props interface or `ComponentPropsWithRef<'element'>` helper type.

**MUST** provide initial value to `useRef` in React 19—all refs are mutable, `useRef()` without argument is no longer valid.

**NEVER** implicitly return non-cleanup values from ref callbacks—use explicit block statements `{ }` or return cleanup functions only. Implicit returns like `ref={current => (instance = current)}` cause TypeScript errors.

**MUST** return cleanup function from ref callbacks when setup logic requires teardown—React 19 supports cleanup pattern for refs:

```typescript
<div ref={(node) => {
  observer.observe(node);
  return () => observer.disconnect();
}} />
```

### Context API

**PREFER** simplified Context syntax `<ThemeContext value="dark">{children}</ThemeContext>` over deprecated `<ThemeContext.Provider value="dark">` in React 19.

**MUST** wrap Context values in `useMemo` and callbacks in `useCallback` to prevent unnecessary re-renders of consuming components—Context triggers re-render of all consumers when value changes.

### Component APIs Removed

**NEVER** use `PropTypes` in React 19—runtime prop validation is removed. Migrate to TypeScript interfaces for compile-time type checking.

**NEVER** use `defaultProps` on function components—removed in React 19. Use ES6 default parameters: `function Component({ name = 'Default' })`.

**NEVER** use legacy `ReactDOM.render` or `ReactDOM.hydrate`—removed in React 19. Use `createRoot().render()` and `hydrateRoot()` respectively.

**NEVER** use `react-test-renderer` in React 19—deprecated with warnings. Migrate to `@testing-library/react` for component testing.

### Document Metadata

**MUST** render `<title>`, `<meta>`, and `<link>` tags directly in components—React 19 automatically hoists them to `<head>`. No need for `react-helmet` or manual head management.

**MUST** use `precedence` attribute on stylesheet links to control loading order and prevent duplicate stylesheets:

```typescript
<link rel="stylesheet" href="/styles.css" precedence="high" />
```

### Resource Preloading

**MUST** use React 19's resource preloading APIs (`prefetchDNS`, `preconnect`, `preload`, `preinit`) from `react-dom` for performance-critical resources:

```typescript
import { preload, prefetchDNS } from 'react-dom';
preload('/fonts/font.woff2', { as: 'font', type: 'font/woff2' });
prefetchDNS('https://api.example.com');
```

---

## 2. Server Components & Client Boundaries

### Component Type Rules

**MUST** add `'use client'` directive at top of file to create Client Component boundary—all components are Server Components by default in React 19 frameworks.

**MUST** use Server Components for static content, data fetching, and database access—reduces client bundle size and executes at build/request time.

**MUST** use Client Components for interactivity requiring state (`useState`, `useReducer`), browser APIs (`localStorage`, `window`), event handlers, or hooks like `useEffect`.

**NEVER** import Server Components into Client Components—pass rendered Server Component output as `children` or props to Client Components instead:

```typescript
// ✅ Correct
function ServerWrapper() {
  return <ClientForm><ServerData /></ClientForm>;
}

// ❌ Wrong
'use client';
import ServerComponent from './ServerComponent'; // Error
```

### Data Fetching Strategy

**MUST** fetch data directly in Server Components using `async/await`—no `useEffect` needed. Server Components can access databases and filesystems directly.

**NEVER** use `useEffect` for data fetching in Server Components—hooks don't work in Server Components. Data fetching is synchronous from the Server Component's perspective.

**MUST** pass Server Actions to Client Components as props for mutations—enables client interactivity while keeping sensitive logic server-side:

```typescript
async function updateUser(formData) {
  'use server';
  await db.users.update(formData);
}

function ServerPage() {
  return <ClientForm action={updateUser} />;
}
```

---

## 3. TypeScript Integration

### Ref Typing

**MUST** use `Ref<T>` type for ref props (accepts both RefObject and RefCallback) or `ComponentPropsWithRef<'element'>` to inherit all props including ref from native elements.

**MUST** type `useRef` with element type and provide `null` as initial value for DOM refs:

```typescript
const inputRef = useRef<HTMLInputElement>(null);
// Access: inputRef.current?.focus()
```

### Props & Component Typing

**PREFER** `interface` over `type` for component props—interfaces support declaration merging and are semantically clearer for object shapes.

**MUST** use `ReactNode` for flexible children prop (accepts string, number, JSX, null, undefined, boolean), or `ReactElement` when only JSX elements are allowed.

**NEVER** use `React.FC` or `React.FunctionComponent`—deprecated pattern that adds no value. Type props directly:

```typescript
interface ButtonProps { label: string }
function Button({ label }: ButtonProps) { }
```

**MUST** mark optional props with `?` and use ES6 default parameters instead of removed `defaultProps`:

```typescript
function Component({ name = 'Guest' }: { name?: string }) { }
```

### Event Handler Typing

**MUST** use specific event types with element type parameters—`ChangeEvent<HTMLInputElement>`, `MouseEvent<HTMLButtonElement>`, `FormEvent<HTMLFormElement>`.

**MUST** use `currentTarget` (typed) instead of `target` (requires casting) for type-safe event handling:

```typescript
function handleChange(e: ChangeEvent<HTMLInputElement>) {
  console.log(e.currentTarget.value); // ✅ Typed
  console.log(e.target.value); // ❌ Requires casting
}
```

**PREFER** inline event handlers for automatic type inference, or use handler types (`React.ChangeEventHandler<T>`) when extracting functions.

### Generic Components

**MUST** use generics for reusable components handling multiple data types—constrain with `extends` for type safety:

```typescript
interface ListProps<T> {
  items: T[];
  renderItem: (item: T) => ReactNode;
}

function List<T>({ items, renderItem }: ListProps<T>) { }
```

**MUST** use `JSX.IntrinsicElements` for polymorphic "as" props to maintain type safety across element types:

```typescript
type BoxProps<T extends keyof JSX.IntrinsicElements> = {
  as: T;
} & JSX.IntrinsicElements[T];
```

### Hook Typing

**PREFER** type inference for simple `useState` (primitives), explicitly type unions, complex objects, or nullable state:

```typescript
const [count, setCount] = useState(0); // Inferred: number
const [user, setUser] = useState<User | null>(null); // Explicit
type Status = 'idle' | 'loading' | 'success' | 'error';
const [status, setStatus] = useState<Status>('idle');
```

**MUST** use discriminated unions for `useReducer` action types with exhaustiveness check:

```typescript
type Action =
  | { type: 'increment' }
  | { type: 'setCount'; value: number };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'increment': return { count: state.count + 1 };
    case 'setCount': return { count: action.value };
    default:
      const _exhaustive: never = action; // Exhaustiveness check
      return state;
  }
}
```

**MUST** use nullable type (`T | null`) for `createContext` when provider is required, and create custom hook that throws error if context is null:

```typescript
const UserContext = createContext<User | null>(null);

function useUser() {
  const user = useContext(UserContext);
  if (!user) throw new Error('useUser must be used within UserProvider');
  return user;
}
```

### Advanced Type Patterns

**MUST** use discriminated unions for mutually exclusive states—prevents invalid state combinations:

```typescript
type DataState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: string };

// TypeScript knows state.data exists only when status === 'success'
```

**MUST** use discriminated unions with `never` type for mutually exclusive props:

```typescript
type ButtonProps = { label: string } & (
  | { href: string; onClick?: never }
  | { onClick: () => void; href?: never }
);
```

**MUST** use `as const` for literal types and `satisfies` operator (TypeScript 4.9+) for precise inference with validation:

```typescript
const STATUSES = ['idle', 'loading', 'success', 'error'] as const;
type Status = typeof STATUSES[number]; // Union type

const config = { timeout: 5000 } as const satisfies AppConfig;
// TypeScript knows timeout is exactly 5000, not just number
```

### TypeScript Configuration

**MUST** enable `strict: true` in tsconfig.json—non-negotiable for React 19 type safety.

**MUST** enable `noUncheckedIndexedAccess: true` to catch array/object access errors—makes array access return `T | undefined`.

**MUST** use `jsx: "react-jsx"` for automatic JSX runtime and `moduleResolution: "bundler"` for modern bundlers.

---

## 4. State Management

### Local State

**PREFER** `useState` for simple, independent state values; use `useReducer` for complex state logic with multiple sub-values or when next state depends on previous state.

**MUST** keep state minimal—only store necessary information. Derive computed values instead of storing redundant state.

**NEVER** lift state unnecessarily—keep state as local as possible, lift only when truly shared across components.

### State Segmentation Strategy

**MUST** separate concerns by state type:
- **Server state** → TanStack Query/SWR
- **URL state** → URL search params (nuqs library)
- **Local UI state** → useState/useReducer
- **Shared app state** → Context/Zustand/Jotai
- **Complex global** → Redux Toolkit

**NEVER** mix server state with client state—server data should be cached separately (React Query/SWR), not stored in component state.

### External State Libraries

**PREFER** Zustand or Jotai for shared state in medium-to-large apps—lightweight (<1KB), minimal boilerplate, excellent performance with minimal re-renders.

**PREFER** Redux Toolkit only for truly complex global state in large applications—provides powerful dev tools and structured patterns but has steeper learning curve.

**MUST** create multiple small stores per feature rather than single monolithic store—improves modularity and prevents unnecessary coupling.

---

## 5. Performance Optimization

### React Compiler

**PREFER** automatic optimization via React Compiler over manual `useMemo`/`useCallback`/`React.memo`—compiler handles most memoization cases automatically.

**MUST** apply manual memoization only when profiling confirms performance issues—compiler optimization is sufficient for majority of use cases.

**MUST** profile before optimizing using React DevTools Profiler—identify actual bottlenecks, don't optimize prematurely.

### Code Splitting

**MUST** implement route-based code splitting as first optimization—provides maximum bundle size reduction with minimal effort:

```typescript
const Home = lazy(() => import('./routes/Home'));

<Suspense fallback={<LoadingSpinner />}>
  <Routes>
    <Route path="/" element={<Home />} />
  </Routes>
</Suspense>
```

**MUST** split large components that render only on specific user interactions—modals, admin panels, feature-gated content.

**MUST** wrap lazy components with `<Suspense>` and provide meaningful fallback UI—multiple lazy components can share single Suspense boundary.

**MUST** handle loading failures with Error Boundaries around Suspense boundaries—prevents app crash when lazy component fails to load.

### Concurrent Features

**MUST** use `useTransition` for non-blocking state updates (search filters, background refreshes, heavy list updates):

```typescript
const [isPending, startTransition] = useTransition();
startTransition(() => setSearchQuery(input));
```

**MUST** use `useDeferredValue` to defer expensive re-renders while allowing React to prioritize urgent updates:

```typescript
const deferredQuery = useDeferredValue(query);
```

### List Rendering

**MUST** virtualize lists with 100+ items using `react-window` or `react-virtualized`—renders only visible items plus buffer.

**ALWAYS** use stable, unique key props (IDs, not array indices)—prevents reconciliation bugs and performance issues.

**NEVER** use inline functions or objects as props in list items—causes unnecessary re-renders:

```typescript
// ❌ Wrong: Creates new function on every render
{items.map(item => <Item onClick={() => handle(item.id)} />)}

// ✅ Correct: Pass stable reference
{items.map(item => <Item onClick={handle} itemId={item.id} />)}
```

### Monitoring

**MUST** monitor Core Web Vitals in production—LCP, FID/INP, CLS are critical user experience metrics.

**MUST** implement Real User Monitoring (RUM)—track performance in actual user environments, set performance budgets and alerts.

---

## 6. Testing

### Testing Tools

**NEVER** use `react-test-renderer` in React 19—deprecated. Use `@testing-library/react` exclusively for component testing.

**PREFER** Vitest over Jest for new React 19 projects using Vite—better compatibility, simpler configuration, faster execution.

**MUST** import `act` from `react` package, not `react-dom/test-utils`—moved in React 19. Use codemod: `npx codemod@latest react/19/replace-act-import`.

### Query Selection

**MUST** prioritize queries in this order:
1. `getByRole` with accessible name
2. `getByLabelText` for form fields
3. `getByPlaceholderText` when labels unavailable
4. `getByText` for non-interactive content
5. `getByTestId` only as escape hatch

**NEVER** query by class names, element types, or internal component structure—tests should resemble user behavior, not implementation details.

### Async Testing

**MUST** use `findBy` queries for elements appearing after async operations—combines querying with waiting:

```typescript
await userEvent.click(button);
const result = await screen.findByText('Loaded'); // ✅ Correct
```

**NEVER** combine `getBy` with manual `waitFor` when `findBy` is sufficient—`findBy` is the idiomatic pattern for async appearance.

**MUST** use `waitForElementToBeRemoved` for testing element disappearance—more reliable than manual `waitFor` loops.

### User Interaction

**MUST** use `userEvent` from `@testing-library/user-event` instead of `fireEvent`—simulates real browser interactions more accurately.

**ALWAYS** await `userEvent` methods—all methods are async in v14+:

```typescript
await userEvent.type(input, 'value');
await userEvent.click(button);
```

### Form Testing

**MUST** test form actions holistically—verify complete flow including pending state, submission, and success/error states:

```typescript
await userEvent.type(input, 'test@example.com');
await userEvent.click(submitButton);
expect(submitButton).toBeDisabled(); // pending state
await screen.findByText('Success');
```

**MUST** use `waitFor` or `findBy` for form validation errors—validation is async, immediate assertions will fail.

### Server Components

**MUST** focus on integration tests for Server Components—they execute server-side, so test complete rendered output rather than isolated units.

**MUST** mock async data at network boundary using MSW (Mock Service Worker)—don't mock at component level.

### Test Organization

**MUST** follow 80-15-5 testing pyramid—80% unit tests, 15% integration tests, 5% E2E tests.

**NEVER** test implementation details—avoid testing internal state, private functions, or React component instances. Test user-visible behavior only.

---

## 7. Accessibility

### Semantic HTML

**MUST** use semantic HTML elements (`<button>`, `<nav>`, `<main>`, `<article>`, `<section>`) over generic `<div>`/`<span>` with ARIA roles—semantic elements provide built-in accessibility.

**MUST** use `<button>` for actions, `<a>` for navigation—never use `<div>` or `<span>` with `onClick` for clickable elements.

**MUST** use ARIA landmark roles (`role="main"`, `role="navigation"`, `role="banner"`, `role="contentinfo"`) or semantic HTML5 equivalents for page structure.

### ARIA Attributes

**MUST** use hyphen-case for ARIA attributes in JSX—`aria-label`, `aria-labelledby`, `aria-describedby` (not camelCase).

**NEVER** add ARIA roles to elements that have implicit roles—`<button>` already has `role="button"`, adding explicit role is redundant.

**MUST** use `aria-label` or `aria-labelledby` for elements lacking visible text labels—icon buttons, search inputs, custom controls.

**MUST** use `aria-describedby` to associate error messages with form fields—screen readers announce description when field receives focus.

**MUST** use `aria-invalid="true"` on form fields with validation errors—indicates invalid state to assistive technology.

### Keyboard Navigation

**MUST** ensure all interactive elements are keyboard accessible—focusable with Tab, activatable with Enter/Space.

**MUST** implement visible focus indicators meeting WCAG 2.2 requirements—minimum 2px outline with 3:1 contrast ratio against background.

**MUST** implement roving tabindex pattern for composite widgets (toolbars, menubars, grids)—only one element in group is tabbable, Arrow keys move focus within group.

**MUST** trap focus inside modals—prevent Tab from moving focus outside modal boundaries:

```typescript
import FocusTrap from 'focus-trap-react';

<FocusTrap>
  <dialog>{modalContent}</dialog>
</FocusTrap>
```

**MUST** return focus to trigger element after modal closes—maintains user context and navigation flow.

### Screen Readers

**MUST** use `aria-live="polite"` for dynamic content updates that aren't time-sensitive—screen reader announces during natural pause.

**MUST** use `aria-live="assertive"` or `role="alert"` for critical, time-sensitive announcements—interrupts current speech.

**MUST** announce SPA route changes to screen readers—add live region that updates with page title on route change.

**MUST** provide visually hidden text for context when icon-only or image-based content is used—use CSS clip pattern:

```css
.visually-hidden {
  position: absolute;
  width: 1px;
  height: 1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
}
```

### Forms

**MUST** associate labels with form controls using `htmlFor` attribute matching input `id`:

```typescript
<label htmlFor="email">Email</label>
<input id="email" type="email" />
```

**MUST** use React 19's `useFormStatus` with `aria-busy` to indicate form submission state:

```typescript
function SubmitButton() {
  const { pending } = useFormStatus();
  return <button aria-busy={pending} disabled={pending}>Submit</button>;
}
```

**MUST** validate forms on submit (not on blur)—validates complete form context, better UX for screen reader users.

---

## 8. Security

### XSS Prevention

**NEVER** use `dangerouslySetInnerHTML` without sanitization—always sanitize HTML with DOMPurify before rendering:

```typescript
import DOMPurify from 'dompurify';

<div dangerouslySetInnerHTML={{
  __html: DOMPurify.sanitize(userContent)
}} />
```

**NEVER** trust user input in URLs—validate protocol is `http:` or `https:` only, reject `javascript:`, `data:`, `vbscript:` protocols.

**NEVER** directly manipulate DOM—bypasses React's built-in XSS protection. Use JSX exclusively.

**ALWAYS** rely on JSX's automatic escaping for user content—React escapes all values by default when using `{expression}` syntax.

### Authentication & Authorization

**MUST** store authentication tokens in HttpOnly cookies—prevents JavaScript access, mitigates XSS token theft.

**NEVER** store sensitive tokens in localStorage or sessionStorage—accessible to JavaScript, vulnerable to XSS attacks.

**MUST** implement phishing-resistant Multi-Factor Authentication (MFA)—FIDO2/WebAuthn, not SMS/TOTP.

**MUST** implement Role-Based Access Control (RBAC) for authorization—use libraries like React CASL or Casbin.

**MUST** use React 19's async transitions with `useTransition` for auth flows—improves UX during authentication state changes.

### Data Fetching

**MUST** keep API keys and sensitive credentials server-side using React Server Components—never expose in client-side code.

**NEVER** trust client-side validation—always validate and sanitize inputs on server. Client validation is UX enhancement only.

**MUST** implement rate limiting, request throttling, and input validation on all API endpoints—prevent abuse and injection attacks.

**MUST** configure CORS restrictively—whitelist specific origins, avoid `Access-Control-Allow-Origin: *` in production.

### Environment Variables

**NEVER** put API keys or secrets in `.env` files for React apps—client-side env vars are visible in browser, bundled into JavaScript.

**MUST** prefix client-side env vars with `REACT_APP_` (CRA) or `VITE_` (Vite)—only use for non-sensitive configuration like feature flags.

**MUST** perform sensitive operations server-side using Server Actions or API routes—keep credentials in server environment only.

### Dependencies

**MUST** run `npm audit` or `yarn audit` regularly—identify and patch known vulnerabilities in dependencies.

**MUST** use Snyk, Dependabot, or Renovate for automated dependency updates—stay current with security patches.

**MUST** pin dependency versions after Sept 16, 2025—major supply chain attacks (Shai-Hulud worm) compromised 500+ packages. Review updates carefully.

**MUST** enable GitHub 2FA and rotate credentials—2025 supply chain attacks exploited compromised maintainer accounts.

---

## 9. Error Handling

### Error Boundaries

**MUST** implement three-level error boundary hierarchy:
1. Top-level around root (catches all uncaught errors)
2. Layer-level around major sections (isolates failures)
3. Component-level around critical components (granular recovery)

**MUST** use `react-error-boundary` package for enhanced features—reset functionality, hooks, logging integration.

**MUST** configure `onCaughtError` and `onUncaughtError` at root level in React 19:

```typescript
createRoot(container, {
  onCaughtError: (error, errorInfo) => {
    logErrorToService(error, errorInfo);
  },
  onUncaughtError: (error, errorInfo) => {
    logCriticalError(error, errorInfo);
  }
});
```

**MUST** place error boundaries around components that fetch data, perform async operations, or are critical to app functionality.

**NEVER** rely on error boundaries for event handler errors—use try/catch blocks in event handlers:

```typescript
async function handleClick() {
  try {
    await riskyOperation();
  } catch (error) {
    setError(error);
  }
}
```

---

## 10. Project Structure & Organization

### Directory Structure

**MUST** use feature-based organization for applications with 12+ components—group by business domain, not technical role:

```
src/
├── features/
│   ├── user/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── services/
│   │   └── index.ts
│   └── product/
├── components/      # Shared UI
├── hooks/          # Shared hooks
├── lib/            # Utilities
└── services/       # API clients
```

**MUST** limit folder nesting to 2-3 levels maximum—prevents navigation complexity and maintains flat, discoverable structure.

**MUST** provide single entry point per feature (`index.ts`)—controls public API, encapsulates internal implementation.

### File Conventions

**MUST** use kebab-case for file names—avoids cross-platform case-sensitivity issues.

**MUST** match file name to component name—`UserProfile.tsx` exports `UserProfile` component.

**MUST** configure absolute imports via jsconfig/tsconfig—cleaner imports, easier refactoring:

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": { "@/*": ["src/*"] }
  }
}
```

### Component Design

**MUST** keep components under 200-300 lines—split larger components into smaller, focused units.

**MUST** follow Single Responsibility Principle—one component handles one concern. If component has multiple concerns, split it.

**PREFER** composition over deep component nesting—flatten hierarchy when possible, each level adds reconciliation overhead.

**MUST** distinguish between container (logic) and presentational (display) components—separate business logic from UI rendering.

### Code Quality

**MUST** use ESLint with React plugin and React Hooks plugin—enforces hook rules and catches common errors.

**MUST** enable TypeScript `strict: true` mode—non-negotiable for production code quality and type safety.

**MUST** organize imports: external libraries → internal modules → components → styles.

---

## Summary: Critical React 19 Changes

**Breaking Changes:**
- forwardRef deprecated (ref as prop)
- PropTypes removed (use TypeScript)
- defaultProps removed for functions (ES6 defaults)
- ReactDOM.render removed (use createRoot)
- react-test-renderer deprecated (use Testing Library)

**New Features:**
- Actions & form hooks (useActionState, useFormStatus, useOptimistic)
- use hook (conditional resource reading)
- Document metadata hoisting
- Resource preloading APIs
- Ref cleanup functions
- Server Components (full support)

**Key Practices:**
- TypeScript strict mode mandatory
- Testing Library over test renderer
- Accessibility-first queries
- Server Components for data, Client for interactivity
- Security: HttpOnly cookies, server-side secrets
- Error boundaries at multiple levels
- Feature-based organization
- Profile before optimizing