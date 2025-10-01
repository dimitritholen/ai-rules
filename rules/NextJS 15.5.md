# NextJS 15.5 Production Rules

## 1. Architecture & Component Design

### App Router & Project Structure
- **MUST** use App Router for new projects—Pages Router is legacy. App Router is the standard for Next.js 15+.
- **MUST** organize with `src/` structure: place `app/`, `components/`, `lib/`, `hooks/`, `types/` inside `src/` to separate source from config files.
- **MUST** use route groups `(folder)` to organize routes logically without affecting URL structure—patterns like `(marketing)/`, `(dashboard)/`, `(admin)/`.
- **MUST** prefix private folders with `_folder` to exclude from routing and signal internal-only code.
- **SHOULD** colocate route-specific components in `_components/` subdirectories within route folders without exposing them as routes.

### Server/Client Component Boundaries
- **MUST** default to Server Components—all components are Server Components unless marked with `"use client"`.
- **MUST** push `"use client"` to leaf nodes only—add directive to specific interactive components at edges of component tree, not large UI sections.
- **NEVER** nest Server Components inside Client Components directly—pass Server Components as `children` props to Client Components to maintain boundary.
- **SHOULD** use Server Components for: data fetching, database/API access, static content, SEO-critical pages, reducing bundle size.
- **SHOULD** use Client Components for: React hooks, event handlers, browser APIs, third-party libraries requiring browser environment, real-time interactivity.

### File Organization
- **SHOULD** use special files strategically: `layout.tsx` for shared UI, `loading.tsx` for Suspense loading states, `error.tsx` for error boundaries, `page.tsx` for public routes.
- **SHOULD** use parallel routes `@folder` and intercepting routes `(.)folder` for modals with deep linking, split dashboards, conditional rendering.

---

## 2. Data Fetching & Server Actions

### Server Component Data Fetching (CRITICAL: Next.js 15 Breaking Changes)
- **MUST** explicitly cache fetch requests with `{ cache: 'force-cache' }`—fetch is NO LONGER CACHED BY DEFAULT in Next.js 15.
- **MUST** await async APIs: `cookies()`, `headers()`, `params`, `searchParams` now return Promises—use `const cookieStore = await cookies()`.
- **SHOULD** use `Promise.all()` for parallel independent requests to minimize waterfalls and reduce load time.
- **SHOULD** use `Promise.allSettled()` when parallel requests can fail independently—prevents one failure from breaking all requests.
- **MUST** call `revalidatePath()` or `revalidateTag()` after mutations to refresh cache and reflect UI changes.

### Server Actions Security & Validation
- **MUST** mark Server Actions with `'use server'` directive at function top (inline) or file top (all exports).
- **NEVER** define Server Actions inside Client Components—only in Server Components or separate files imported into Client Components.
- **MUST** validate all Server Action inputs at runtime with schema libraries (Zod, Valibot)—Server Actions are public HTTP endpoints callable by anyone, bypassing UI validation.
- **MUST** re-authenticate and re-authorize users inside every Server Action—treat as public API endpoints, not protected by middleware or component checks.
- **MUST** validate inputs before execution: current user authorization + argument integrity to prevent unauthorized access.
- **SHOULD** use `useActionState` hook for loading states and error handling when calling Server Actions from forms.

### Streaming & Caching
- **SHOULD** wrap async components with `<Suspense>` for granular loading—progressive streaming instead of blocking entire page renders.
- **NEVER** use client-side data fetching (Axios) on server if you need Next.js caching—bypasses fetch caching, memoization, revalidation. Use native `fetch` in Server Components.
- **NEVER** rely on automatic route caching—use explicit `'use cache'` directives, `revalidate`, and `cacheTag` for granular control.
- **MUST** use tag-based invalidation (`cacheTag`) for remotely clearing caches when data changes in admin panels or external systems.

---

## 3. Performance Optimization

### Image & Font Optimization
- **MUST** use `next/image` with `priority` prop for above-the-fold images (LCP elements like heroes) to eliminate render-blocking and achieve LCP ≤2.5s.
- **MUST** set explicit `width` and `height` on all images to prevent Cumulative Layout Shift (CLS target ≤0.1).
- **SHOULD** lazy-load below-the-fold images using `next/image` default behavior to improve initial load by 20-30%.
- **NEVER** serve images from third-party domains without optimization—TLS handshakes add latency degrading LCP.
- **MUST** use variable fonts via `next/font` for all typography—consolidates weights/styles into one file, reducing payload 50-70%.
- **SHOULD** self-host fonts with `next/font` to eliminate external requests and reduce TTFB, directly improving LCP.

### Bundle Size & Code Splitting
- **MUST** use dynamic imports with `next/dynamic` for components over 20KB not critical for initial render—reduces First Load JS by 25%+.
- **SHOULD** import only necessary functions from large libraries (`import debounce from 'lodash/debounce'`) to enable tree-shaking and reduce bundle 30%+.
- **MUST** use `next/script` with `strategy="afterInteractive"` or `"lazyOnload"` for analytics/non-critical third-party scripts to prevent INP degradation (target ≤200ms).

### Rendering Strategy
- **MUST** use Static Site Generation (SSG) or Incremental Static Regeneration (ISR) for infrequently updated content—delivers LCP <2.5s, eliminates server processing.
- **NEVER** use Client-Side Rendering (CSR) for initial page content—degrades all Core Web Vitals, increases TTI by 2-3x.
- **SHOULD** implement Partial Pre-Rendering (PPR) for routes mixing static/dynamic content—static shell loads instantly while dynamic parts stream.

---

## 4. Type Safety & Error Handling

### TypeScript Configuration
- **MUST** enable `"strict": true` in tsconfig.json to enable all strict mode checks, catching errors at compile-time.
- **MUST** use TypeScript 5.1.3+ with @types/react 18.2.8+ to avoid `'Promise<Element>' is not a valid JSX element` errors with async Server Components.
- **SHOULD** enable `typedRoutes: true` in next.config.ts for compile-time route type safety, catching invalid Links before production.

### Component Type Safety
- **MUST** type Server Component props explicitly with Promise wrappers:
  ```typescript
  type PageProps = {
    params: Promise<{ slug: string }>;
    searchParams?: Promise<{ [key: string]: string | string[] | undefined }>;
  }
  ```
- **NEVER** export both `metadata` object and `generateMetadata` from same route—Next.js will error. Choose one based on static vs dynamic needs.

### Error Handling Patterns
- **MUST** distinguish expected errors from uncaught exceptions—return expected errors (validation failures) to client, let exceptions bubble to error boundaries.
- **SHOULD** use `error.tsx` files at appropriate route hierarchy levels for granular error boundaries in Server Components.
- **NEVER** rely on error boundaries for event handler errors—use manual try/catch with useState/useReducer to handle and display these errors.
- **SHOULD** use next-safe-action for end-to-end Server Action type safety with validation library integration and proper error patterns.

---

## 5. Security

### Server Actions & API Security (CRITICAL)
- **MUST** validate all Server Action inputs at runtime—TypeScript only checks compile-time; Server Actions are public HTTP endpoints.
- **MUST** re-authenticate users in every Server Action—can be invoked directly from clients, not just through UI. Defense-in-depth required.
- **NEVER** rely solely on middleware for auth/authorization—CVE-2025-29927 demonstrated critical bypass vulnerabilities. Always implement secondary checks in routes and actions.
- **MUST** use POST method exclusively for state-changing operations—Server Actions use POST by default; GET requests must never modify data.

### Environment Variables & Secrets
- **NEVER** prefix sensitive credentials with `NEXT_PUBLIC_`—embedded in client bundle during build, exposed to anyone inspecting browser code.
- **SHOULD** verify no sensitive data leaked into client bundle after build—check `/static/chunks` directory. Use `server-only` package to guarantee code never reaches client.

### Content Security & Input Sanitization
- **SHOULD** implement nonce-based CSP via middleware for XSS protection—prevents cross-site scripting, clickjacking, code injection. Note: forces dynamic rendering.
- **MUST** exclude `'unsafe-eval'` from production CSP—needed for React Refresh in dev but creates XSS vulnerabilities in production.
- **MUST** sanitize all user-supplied HTML and URLs with libraries like sanitize-html—implement whitelist-based access to prevent XSS and SSRF.
- **MUST** use parameterized queries or ORMs for database operations—prevents SQL injection. Modern ORMs (Prisma, Drizzle) handle automatically.

### Access Control
- **MUST** implement Data Access Layer (DAL) pattern for consistent authorization—internal library controlling data access prevents authorization bugs.
- **NEVER** return complete database objects to clients—use DTOs (Data Transfer Objects) to expose only necessary fields.
- **SHOULD** implement rate limiting on all public API routes and Server Actions—prevents abuse, DoS attacks, credential stuffing.

---

## 6. Testing Strategy

### Server Components & E2E Testing
- **MUST** use E2E testing (Playwright/Cypress) for async Server Components—Jest and Vitest do not support async Server Components.
- **SHOULD** use React Testing Library with Jest/Vitest for synchronous Server and Client Components only.
- **NEVER** attempt to unit test Server Components with data fetching or async operations—E2E tests are the recommended approach.
- **MUST** run Playwright tests against production builds (`npm run build && npm run start`) to catch production-only issues.

### Server Actions & API Testing
- **MUST** use E2E testing for Server Actions—mocking Server Actions in Playwright is challenging.
- **SHOULD** enable `experimental: { testProxy: true }` in next.config for fetch mocking with MSW and Playwright for server-side fetch requests.
- **MUST** use next-test-api-route-handler or similar for API route unit tests—standard testing requires special handling.

### Testing Configuration
- **SHOULD** configure Jest with `next/jest` for automatic transpilation, env vars, path aliases integration.
- **MUST** set `testEnvironment: 'jsdom'` for component tests in both Jest and Vitest.
- **SHOULD** aim for 80%+ test coverage with balanced pyramid—prioritize E2E over unit tests for async components; allocate 15-20% to integration tests.
- **SHOULD** use `unstable_doesMiddlewareMatch` from `next/experimental/testing/server` (Next.js 15.1+) to assert middleware execution for URLs, headers, cookies.

---

## 7. Deployment & Production

### Build Optimization
- **MUST** enable Turbopack for production builds using `next build --turbopack`—delivers 2-5x faster compilation, battle-tested with 1.2B+ requests on Vercel production.
- **MUST** set `output: 'standalone'` in next.config.js for self-hosted deployments—enables automatic sharp optimization and reduces deployment size.
- **SHOULD** implement persistent caching strategies for Turbopack to make all builds incremental—valuable for larger projects.

### Environment Configuration
- **MUST** use `VERCEL_ENV` instead of `NODE_ENV` to differentiate production vs preview—both set `NODE_ENV=production` during build.
- **MUST** scope environment variables explicitly to Production, Preview, or Development environments. Redeploy after any variable changes.
- **SHOULD** keep environment variables under 64KB total per deployment (5KB per variable for Edge Functions/Middleware).

### Monitoring & Error Tracking
- **MUST** implement `onRequestError` hook (Next.js 15) to capture server-side errors with full context before sending to observability providers.
- **MUST** integrate structured error tracking (Sentry with @sentry/nextjs@10.13.0+ for Turbopack support) with environment-aware config using `VERCEL_ENV`.
- **SHOULD** reduce `tracesSampleRate` below 1.0 in production to avoid excessive telemetry costs while maintaining visibility.
- **MUST** configure separate Sentry DSNs for client (`sentry.client.config.js`) and server (`sentry.server.config.js`) to track errors across environments.
- **NEVER** ship preview/dev error tracking configs to production—use `VERCEL_ENV` to conditionally initialize monitoring.

### Cache Management
- **MUST** call `router.refresh()` to invalidate Router Cache and force server re-render for current route.
- **NEVER** assume Full Route Cache persists across deployments—cleared on every deployment, unlike Data Cache.
- **SHOULD** configure custom cache handlers in next.config.js when self-hosting with Kubernetes to prevent stale data across pod restarts.

### Self-Hosting
- **MUST** disable in-memory caching and configure distributed cache handlers (Redis, etc.) when running multiple instances behind load balancers.
- **SHOULD** use PM2, Grafana, Prometheus, or New Relic for production monitoring when self-hosting—Next.js doesn't include built-in observability.
- **SHOULD** implement CI/CD pipelines with GitHub Actions including linting, type checking, bundle size tracking before deployment.

---

## 8. SEO & Metadata

### Metadata Configuration
- **MUST** use `generateMetadata` function for dynamic routes—static metadata objects cannot access route parameters or external data.
- **NEVER** export both `metadata` object and `generateMetadata` from same route—Next.js errors. File-based metadata (opengraph-image.js) overrides both.
- **MUST** include canonical URLs in metadata (`alternates.canonical`) for duplicate content—critical for pagination, filtered pages, multi-language sites.

### Structured Data & Discoverability
- **MUST** implement JSON-LD structured data for content types (articles, products, events, organizations)—test with Google Rich Results Test before deployment.
- **MUST** create dynamic sitemap.xml using sitemap.js Route Handler in app root—use `generateSitemaps` function for large sites requiring split sitemaps.
- **MUST** create robots.txt using robots.js Route Handler in app root—return Robots object with User-agent, Allow, Disallow, Sitemap properties.

### Social & Image Optimization
- **MUST** set complete Open Graph metadata for social sharing—openGraph object with title, description, images (1200x630px min), url, type. Impacts CTR from social channels.
- **MUST** use Next.js Image component with descriptive alt text for accessibility and image search rankings—automatic optimization improves Core Web Vitals.
- **MUST** deploy with HTTPS enabled—Google prioritizes secure sites; non-HTTPS receives ranking penalties and browser warnings.

### Rendering for SEO
- **SHOULD** use SSG or ISR over client-side rendering for search engines—pre-rendered pages reindexed more frequently by Google.
- **MUST** use semantic HTML in Server Components—proper heading hierarchy (h1-h6), nav, main, article, section tags improve crawlability and ranking.
- **SHOULD** optimize Core Web Vitals using built-in Next.js features—Google uses site speed as ranking factor (LCP, FID, CLS metrics).

---

## Implementation Priority

**Priority 1 (Breaking Changes & Critical Security):**
- Async APIs (cookies, headers, params, searchParams)
- No default caching in fetch()
- Server Action input validation
- Server Action re-authentication
- Environment variable security

**Priority 2 (Architecture Fundamentals):**
- Server/Client component boundaries
- App Router patterns
- Data fetching strategies
- Type safety configuration

**Priority 3 (Performance & Quality):**
- Image/font optimization (Core Web Vitals)
- Bundle size management
- Testing strategies
- Error handling patterns

**Priority 4 (Production Readiness):**
- Monitoring/logging setup
- SEO fundamentals
- Deployment configuration
- Cache management

---

**Version:** Next.js 15.5 (Mid-2025)
**Sources:** Vercel official documentation, Next.js docs, OWASP guidelines, Google SEO guidelines, web.dev, established TypeScript and React patterns.

These rules address production-critical concerns: security vulnerabilities (OWASP Top 10), performance benchmarks (Core Web Vitals), type safety (compile-time error prevention), testing coverage (80%+ with E2E focus), and SEO fundamentals (crawlability, indexing, rankings). Following these rules ensures code quality meets enterprise standards for Next.js 15.5 applications.
