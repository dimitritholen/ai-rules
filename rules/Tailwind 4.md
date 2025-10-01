# Tailwind CSS 4 Production Rules

Enterprise-grade best practices for Tailwind CSS v4 based on official documentation, reputable sources, and industry standards (mid-2025). Follow these rules to ensure top-quality, production-ready code.

---

## 1. Configuration & Setup

### CSS-First Configuration (v4 Paradigm)

**MUST** use `@import "tailwindcss"` instead of `@tailwind` directives → v4 uses native CSS imports, not custom directives.

**MUST** define design tokens in `@theme` directive when generating utilities → theme variables auto-create utility classes AND CSS variables.
```css
@theme {
  --color-primary: oklch(0.72 0.11 178);
  --font-display: "Inter", sans-serif;
  --spacing-gutter: 2rem;
}
```

**MUST** use `:root` for CSS variables that should NOT generate utilities → separates internal variables from design tokens.

**MUST** follow namespace conventions: `--color-*`, `--font-*`, `--spacing-*`, `--breakpoint-*`, `--container-*` → determines which utility classes are generated.

**MUST** use `--value()` function in custom utilities to reference theme tokens → enables proper token resolution.

**NEVER** use preprocessors (Sass, Less, Stylus) with v4 → v4 doesn't support preprocessor syntax; use native CSS features instead.

**NEVER** define `@theme` variables inside selectors or media queries → must be top-level declarations.

### Build Tool Integration

**MUST** prefer `@tailwindcss/vite` plugin over PostCSS plugin when using Vite → 5-100x faster builds through tight integration.

**MUST** use Node.js 20+ for v4 → required minimum version.

**MUST** configure automatic content detection with `@source` directive only when needed → v4 auto-detects files respecting `.gitignore`; manual config only for edge cases.

**MUST** verify browser support: Chrome 111+, Safari 16.4+, Firefox 128+ → v4 requires modern CSS features; use v3.4 for older browsers.

---

## 2. Component Architecture & Abstraction

### Extraction Patterns

**ALWAYS** prefer framework components (React/Vue/Svelte) over CSS abstractions → encapsulates structure + styling, more maintainable than `@apply`.

**MUST** extract to component when pattern repeats 3+ times → before that, keep utilities inline; after, create component abstraction.

**NEVER** extract components prematurely → let patterns emerge organically through actual repetition.

**MUST** use variant management libraries (tailwind-variants, CVA) for complex component APIs → type-safe, composable variant systems.
```typescript
const button = tv({
  base: "rounded-lg font-medium transition",
  variants: {
    variant: {
      primary: "bg-blue-500 text-white hover:bg-blue-600",
      secondary: "bg-gray-200 text-gray-900 hover:bg-gray-300"
    }
  }
});
```

### @apply Usage Guidelines

**NEVER** use `@apply` for "cleaner" HTML → defeats utility-first benefits; use component abstraction instead.

**MUST** limit `@apply` to: base layer styles, simple highly-reusable patterns (buttons/badges), third-party integrations requiring class names.

**NEVER** use `@apply` with `@layer components` or `@layer base` in v4 → causes errors; use `@utility` directive instead.

**MUST** use `@utility` directive for custom utilities in v4:
```css
@utility tab-* {
  tab-size: --value(--tab-size-*);
}
```

### Component Libraries

**MUST** use Headless UI for production-ready accessible components → official library from Tailwind Labs with full WAI-ARIA compliance.

**MUST** use shadcn/ui pattern (copy-paste components) for full customization → components live in your codebase, not node_modules.

---

## 3. Utility Composition & Class Management

### Class Ordering

**MUST** enforce consistent class ordering: layout → spacing → typography → color → effects → states → responsive.

**MUST** use Prettier plugin (`prettier-plugin-tailwindcss`) with `tailwindStylesheet` option → automatic sorting, zero config.
```json
{
  "plugins": ["prettier-plugin-tailwindcss"],
  "tailwindStylesheet": "./src/styles/globals.css"
}
```

**MUST** configure Tailwind CSS IntelliSense for v4: set `tailwindCSS.experimental.configFile` to CSS entry point.

### Arbitrary Values

**MUST** write complete class names including arbitrary values → JIT compiler uses regex detection; dynamic strings won't be detected.
```javascript
// ❌ NEVER: Won't be detected
<div className={`text-${size}`}>

// ✓ CORRECT
<div className={size === 'sm' ? 'text-sm' : 'text-lg'}>
```

**MUST** use underscores for spaces in arbitrary values → `grid-cols-[1fr_500px_2fr]` not `grid-cols-[1fr 500px 2fr]`.

**MUST** document arbitrary values with comments; add to theme if used 2+ times → prevents design token proliferation.

**MUST** use type disambiguation for namespace collisions → `text-[length:14px]` vs `text-[color:#333]`.

### Custom Utilities

**MUST** use `@utility` directive (not `@layer utilities`) for custom utilities in v4 → better CSS placement determination.

**MUST** define custom variants with `@custom-variant` for reusable state combinations:
```css
@custom-variant hocus (&:is(:hover, :focus-visible));
```

**NEVER** create arbitrary values for repeated patterns → extract to theme or utility class.

---

## 4. Performance & Build Optimization

### Production Targets

**MUST** target <10kB compressed CSS bundle for most projects → typical production size; >20kB indicates unused classes.

**MUST** verify bundle optimization: automatic purging enabled, minification active, compression configured (Brotli > Gzip).

**MUST** use `--minify` flag for production builds → ensures CSS minification.

**MUST** monitor build times: full builds ~50-105ms, incremental ~3-8ms, no-change <1ms → v4 baselines; investigate if exceeding 500ms.

### JIT Compilation

**ALWAYS** write complete class names in source code → JIT uses regex scanning; concatenated strings bypass detection.

**MUST** use safelist for truly dynamic classes that can't be written in source → rare exceptions only.

**NEVER** rely on `recursion_limit` as primary halt mechanism → indicates architectural issues; implement explicit exit conditions.

### Content Detection

**MUST** leverage automatic content detection → v4 scans from CWD, respects `.gitignore`, uses intelligent heuristics.

**MUST** use `@source` directive only for monorepos or non-standard structures → explicit path specification when auto-detection insufficient.

---

## 5. Responsive Design & Layout

### Mobile-First Architecture

**ALWAYS** start with mobile layout (unprefixed utilities) → Tailwind is inherently mobile-first.

**NEVER** think of `sm:` as "small screens" → means "at small breakpoint (≥640px) and above".
```html
<!-- ✓ CORRECT: Mobile base, desktop enhancement -->
<div class="flex-col md:flex-row">

<!-- ❌ WRONG: Desktop base, mobile override -->
<div class="flex-row sm:flex-col">
```

**MUST** use media queries for page-level layout → header, main grid structure, overall architecture.

### Container Queries

**MUST** use container queries (`@container`) for component-level responsiveness → components adapt to parent size, not viewport; enables true portability.

**MUST** name containers with `@container/{name}` when nesting multiple containers → prevents ambiguity; target with `@md/{name}:`.

**MUST** use `@max-*` variants for max-width container queries → `@max-md:` applies styles below threshold.

**ALWAYS** document container naming conventions → critical for team maintainability.

### Breakpoint Strategy

**MUST** customize breakpoints using `--breakpoint-*` theme variables in v4:
```css
@theme {
  --breakpoint-tablet: 40rem;
  --breakpoint-desktop: 80rem;
}
```

**NEVER** add responsive variants to every utility → keep HTML readable; use strategic breakpoints (mobile, tablet, desktop).

**MUST** use arbitrary breakpoints for one-off cases → `min-[920px]:block` cleaner than custom config for single use.

**MUST** stack variants correctly: `{responsive}:{state}:{utility}` → `md:hover:bg-blue-500`.

### Touch Targets

**MUST** ensure interactive elements minimum 44×44px (AAA) or 24×24px with 8px spacing (AA) → WCAG 2.5.5 compliance.
```html
<button class="w-11 h-11 p-2"> <!-- 44px AAA -->
```

---

## 6. Theming & Design System

### Design Token Management

**MUST** implement three-layer color hierarchy: raw colors → base tokens → semantic tokens.
```css
@theme {
  /* Raw colors */
  --color-teal-500: oklch(0.65 0.15 180);

  /* Semantic tokens */
  --color-primary: var(--color-teal-500);
  --color-text-error: oklch(0.5 0.2 30);
}
```

**MUST** use OKLCH color space for all custom colors → perceptually uniform, wider gamut (P3), better gradients than HSL.
```css
--color-mint-500: oklch(0.72 0.11 178);
```

**NEVER** use HSL when OKLCH is available → HSL lacks perceptual uniformity and gamut support.

**MUST** centralize design tokens in `@theme` to prevent scattered arbitrary values → maintains consistency, enables theming.

### Dark Mode

**MUST** configure dark mode with CSS variable overrides in `@layer theme`:
```css
@theme {
  --color-background: #ffffff;
  --color-text: #000000;
}

@layer theme {
  :root, :host {
    @variant dark {
      --color-background: #000000;
      --color-text: #ffffff;
    }
  }
}
```

**MUST** provide three-way toggle: Light/Dark/Auto (system preference) → store in localStorage, respect `prefers-color-scheme`.

**MUST** use `:root, :host` selector syntax → Shadow DOM compatibility for web components.

### Multi-Theme Strategy

**MUST** use data attributes for multi-theme switching:
```css
@layer base {
  [data-theme='ocean'] {
    --color-primary: #aab9ff;
  }
  [data-theme='candy'] {
    --color-primary: #f9a8d4;
  }
}
```

---

## 7. Accessibility (Critical)

### Focus Management

**MUST** use `focus-visible` for keyboard-only focus indicators → avoids visual pollution for mouse users.
```html
<button class="focus-visible:ring-2 focus-visible:ring-blue-500">
```

**MUST** prefer `outline` over `ring` for accessibility-critical focus → `ring` uses `box-shadow` which becomes invisible in High Contrast Mode; `outline` is preserved.
```html
<!-- ✓ CORRECT: Visible in High Contrast Mode -->
<button class="focus-visible:outline-2 focus-visible:outline-blue-500">

<!-- ❌ PROBLEMATIC: Invisible in High Contrast Mode -->
<button class="focus-visible:ring-2 ring-blue-500">
```

**MUST** ensure focus indicators have minimum 3:1 contrast ratio → WCAG 2.1 SC 1.4.11 compliance.

**MUST** use `focus-within` for container-level focus effects → emphasizes active form sections.

### Screen Reader Support

**MUST** use `sr-only` for visually hidden, screen-reader-accessible content:
```html
<a href="/settings">
  <svg><!-- icon --></svg>
  <span class="sr-only">Settings</span>
</a>
```

**MUST** show hidden content on focus for skip links:
```html
<a href="#main" class="sr-only focus:not-sr-only focus:absolute focus:top-0">
  Skip to main content
</a>
```

### ARIA Attributes

**MUST** style based on ARIA states using `aria-*` variants:
```html
<button class="aria-checked:bg-sky-700 aria-expanded:rotate-180">
<th class="aria-[sort=ascending]:text-blue-600">
```

**MUST** properly associate error messages with `aria-errormessage` and `aria-invalid`:
```html
<input
  aria-invalid="true"
  aria-errormessage="error-id"
  class="aria-[invalid=true]:border-red-500"
/>
<span id="error-id" class="text-red-600">Error message</span>
```

**ALWAYS** use semantic HTML over ARIA roles → `<button>` instead of `<div role="button">`; ARIA is supplement, not replacement.

### Motion & Animation

**MUST** respect `prefers-reduced-motion` with `motion-safe:` and `motion-reduce:` variants:
```html
<div class="motion-safe:animate-spin motion-reduce:animate-none">
```

**NEVER** completely remove feedback for reduced motion → provide alternative (color, fade) instead of nothing.

**ALWAYS** use `motion-safe:` as default prefix rather than undoing with `motion-reduce:` → less code.

### Color & Contrast

**MUST** ensure minimum contrast ratios: 4.5:1 for normal text (AA), 3:1 for large text and UI components (AA).

**NEVER** rely on color alone for information → combine color with icons, text labels, or patterns.

### Keyboard Navigation

**MUST** ensure all interactive elements are keyboard accessible → test with Tab, Shift+Tab, Enter, Space, Arrow keys.

**MUST** use focus trapping for modals/dialogs → use Headless UI Dialog or `focus-trap-react` for custom implementations.

**MUST** implement proper ARIA attributes for dialogs: `role="dialog"`, `aria-modal="true"`, `aria-labelledby`, `aria-describedby`.

### Testing Requirements

**MUST** implement automated accessibility testing: jest-axe, cypress-axe, or @axe-core/playwright → catches ~30-50% of issues.

**MUST** conduct manual testing: keyboard navigation, screen readers (NVDA/JAWS/VoiceOver), High Contrast Mode, 200% zoom.

**MUST** test with at least one screen reader → automated tools miss 50-70% of barriers.

---

## 8. Production Readiness & Quality

### Migration to v4

**MUST** use automated upgrade tool: `npx @tailwindcss/upgrade@next` → handles dependencies, config migration, template updates.

**MUST** address breaking changes:
- `border` → now uses `currentColor` (was gray-200); explicitly set `border-gray-200` where needed
- `ring` → now 1px currentColor (was 3px blue); use `ring-2 ring-blue-500` explicitly
- `shadow-sm` → renamed to `shadow-xs`
- Opacity modifiers → use slash syntax: `bg-black/50` not `bg-opacity-50`
- Flex utilities → `flex-shrink-*` → `shrink-*`, `flex-grow-*` → `grow-*`

**MUST** test thoroughly after migration → full builds show 3.5x performance improvement; verify visual regression.

### File Organization

**MUST** organize monorepos with shared config package:
```
packages/
├── tailwind-config/    # @repo/tailwind-config
│   ├── styles.css
│   └── package.json
└── ui/                 # Component library
```

**NEVER** import Tailwind CSS in multiple packages → prevents style duplication; each app imports once, references shared theme.

**MUST** use presets for shared configuration across packages → consistency without duplication.

### Documentation

**MUST** document: design token mappings, component usage guides, class ordering conventions, custom utilities, extraction patterns.

**MUST** maintain style guide for custom utilities and naming conventions → critical for team consistency.

**MUST** comment complex utility combinations → aids readability and maintenance.

### Code Quality

**MUST** configure linting: ESLint (`eslint-plugin-tailwindcss`), Stylelint (`stylelint-tailwindcss`).

**MUST** use VS Code extensions: Tailwind CSS IntelliSense (official), Headwind (sorting), Tailwind Fold (collapse utilities).

**MUST** implement visual regression testing (Chromatic, Percy, Playwright) → catches styling regressions during refactoring.

**NEVER** allow 20+ utility chains without component extraction → extract to component or document reason.

---

## 9. Critical Anti-Patterns

**NEVER** use arbitrary values everywhere → use design tokens; `w-[347px]` scattered throughout indicates missing theme config.

**NEVER** fight the framework → don't try to make Tailwind work like traditional CSS architectures; embrace utility-first.

**NEVER** overuse `@apply` → prefer component extraction; `@apply` reintroduces CSS naming overhead and cascade issues.

**NEVER** ignore semantic HTML → Tailwind styles elements; doesn't replace proper HTML structure, landmarks, or ARIA.

**NEVER** concatenate class strings dynamically → `className={`text-${size}`}` bypasses JIT detection; write complete class names.

**NEVER** skip accessibility → Tailwind provides tools but doesn't enforce; deliberate implementation required for WCAG compliance.

**NEVER** ignore purge/content configuration → shipping unused CSS bloats production bundles by 80%+.

**NEVER** use `@theme` in imported files → only works in main entry file Tailwind processes, not files imported via `@import`.

**NEVER** duplicate Tailwind imports in monorepo → causes style duplication; import once per app, share theme via package.

---

## 10. Quality Checklist

### Pre-Deployment Verification

- [ ] Production CSS bundle <10kB compressed
- [ ] All design tokens centralized in `@theme`
- [ ] No arbitrary values without documentation
- [ ] Class ordering automated via Prettier plugin
- [ ] Component extraction at 3+ repetition threshold
- [ ] All interactive elements have `focus-visible` indicators with 3:1 contrast
- [ ] Touch targets minimum 44×44px or 24×24px with spacing
- [ ] All animations respect `prefers-reduced-motion`
- [ ] Color contrast meets WCAG AA (4.5:1 text, 3:1 UI)
- [ ] Semantic HTML used throughout
- [ ] ARIA attributes on complex components
- [ ] "Skip to main content" link implemented
- [ ] Automated accessibility tests passing (jest-axe/cypress-axe/Playwright)
- [ ] Manual keyboard navigation tested
- [ ] Screen reader testing completed
- [ ] High Contrast Mode compatibility verified
- [ ] Visual regression tests passing
- [ ] Build times within v4 baselines (<100ms full, <10ms incremental)
- [ ] Browser compatibility verified (Chrome 111+, Safari 16.4+, Firefox 128+)
- [ ] Documentation complete (tokens, components, conventions)

---

## Sources

**Official Documentation:**
- Tailwind CSS v4.0 Documentation (tailwindcss.com)
- Tailwind CSS v4.0 Blog Announcement
- Headless UI Official Documentation (headlessui.com)

**Reputable Technical Sources:**
- W3C/WAI WCAG Guidelines & ARIA Practices
- MDN Web Docs (Mozilla Developer Network)
- CSS-Tricks
- Smashing Magazine
- LogRocket Blog
- DEV Community (verified authors)

**Framework/Tool Documentation:**
- Vite Official Documentation
- Prettier Plugin for Tailwind CSS
- tailwind-variants Official Documentation
- shadcn/ui Official Documentation

**Industry Standards:**
- WCAG 2.1 & 2.2 Guidelines
- iOS Human Interface Guidelines
- Material Design Guidelines

All practices verified against official documentation and reputable sources as of mid-2025. Follow these rules for enterprise-grade, production-ready Tailwind CSS 4 code.
