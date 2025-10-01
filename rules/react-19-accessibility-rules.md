# React 19 Accessibility (a11y) Best Practices - 2025

## Overview
This document contains enterprise-grade accessibility rules for React 19 applications based on W3C WCAG 2.2, WAI-ARIA Authoring Practices Guide (APG), The A11y Project, and official React documentation as of mid-2025.

## 1. Semantic HTML & ARIA Attributes

### 1.1 Prefer Semantic HTML Over Generic Elements
- **Rule**: Always use semantic HTML5 elements over generic `<div>` and `<span>` when appropriate
- **Elements**: `<header>`, `<main>`, `<nav>`, `<footer>`, `<section>`, `<article>`, `<aside>`
- **Rationale**: Semantic elements are inherently accessible and recognized by screen readers without additional ARIA attributes
- **React Pattern**:
  ```jsx
  // ❌ Avoid
  <div className="navigation">
    <div className="content">...</div>
  </div>

  // ✅ Prefer
  <nav>
    <main>...</main>
  </nav>
  ```

### 1.2 ARIA Attributes in JSX
- **Rule**: All ARIA attributes must use hyphen-case (kebab-case), not camelCase
- **Support**: All `aria-*` attributes are fully supported in JSX
- **Pattern**:
  ```jsx
  // ✅ Correct
  <button aria-label="Close dialog" aria-pressed="false">

  // ❌ Wrong
  <button ariaLabel="Close dialog" ariaPressed="false">
  ```

### 1.3 Landmark Regions
- **Rule**: Use landmark elements to indicate important content regions
- **Required Landmarks**:
  - One `<main>` element per page (contains primary content)
  - `<nav>` for navigation regions
  - `<header>` for banner (site header)
  - `<footer>` for contentinfo (site footer)
- **Multiple Landmarks**: If multiple navigation or region landmarks exist, provide unique labels using `aria-label` or `aria-labelledby`
- **Pattern**:
  ```jsx
  <nav aria-label="Primary navigation">
    {/* primary menu */}
  </nav>
  <nav aria-label="Footer navigation">
    {/* footer links */}
  </nav>
  ```

### 1.4 Avoid Generic ARIA Roles When Semantic HTML Exists
- **Rule**: Do not use ARIA roles when native HTML elements provide the same semantics
- **Example**: Use `<button>` instead of `<div role="button">`
- **W3C Principle**: "No ARIA is better than Bad ARIA"

## 2. Keyboard Navigation

### 2.1 Fundamental Keyboard Conventions
- **Tab/Shift+Tab**: Move focus between UI components (follows reading order)
- **Arrow Keys**: Navigate within composite components (grids, menus, tabs, listboxes)
- **Enter/Space**: Activate buttons, links, and controls
- **Escape**: Close dialogs, dismiss menus, cancel operations
- **Home/End**: Move to first/last element in a list or component

### 2.2 Tab Sequence Requirements
- **Rule**: Only one focusable element of a composite UI component should be in the tab sequence at a time
- **Implementation**: Use `tabIndex="0"` for one element and `tabIndex="-1"` for others
- **Exception**: Simple form controls like individual inputs should all be in the tab sequence

### 2.3 Roving Tabindex Pattern
- **Use Case**: For composite widgets (custom menus, tabs, grids, toolbars)
- **Implementation**:
  ```jsx
  const [activeIndex, setActiveIndex] = useState(0);

  items.map((item, index) => (
    <button
      key={item.id}
      tabIndex={index === activeIndex ? 0 : -1}
      onFocus={() => setActiveIndex(index)}
      onKeyDown={handleArrowKeys}
    >
      {item.label}
    </button>
  ))
  ```

### 2.4 aria-activedescendant Pattern
- **Use Case**: For composite widgets where the container retains focus
- **Implementation**: Container element remains focused; `aria-activedescendant` attribute references the active child element ID
- **Example**: Used in comboboxes, listboxes, and grids

### 2.5 Disabled Controls
- **Rule**: Remove disabled controls from tab sequence
- **Pattern**: Use `disabled` attribute for native elements, or `tabIndex="-1"` and `aria-disabled="true"` for custom controls
- **Exception**: Some composite widgets (menus, listboxes) may keep disabled items focusable for keyboard navigation

### 2.6 Avoid autofocus
- **Rule**: Do not use `autofocus` attribute as it can disorient keyboard and screen reader users
- **Exception**: Justified only in very specific cases (search pages, modals with clear context)

## 3. Focus Management

### 3.1 Visible Focus Indicators
- **WCAG Requirement**: All interactive elements must have a visible focus indicator
- **Minimum Standards (WCAG 2.2 Level AA)**:
  - Focus indicator area: at least 2 CSS pixels thick perimeter
  - Contrast ratio: at least 3:1 between focused and unfocused states
- **Rule**: Never use `outline: none` without providing an alternative focus style
- **Pattern**:
  ```css
  button:focus-visible {
    outline: 2px solid #0066cc;
    outline-offset: 2px;
  }
  ```

### 3.2 Focus Not Obscured (WCAG 2.2)
- **Level AA (2.4.11)**: When a UI component receives keyboard focus, it must not be entirely hidden by author-created content
- **Level AAA (2.4.12)**: When a UI component receives keyboard focus, no part should be hidden
- **Common Issue**: Fixed headers, sticky navigation, or cookie banners covering focused elements

### 3.3 Focus Management with useRef
- **Rule**: Use `useRef` and `useEffect` to programmatically manage focus
- **Pattern**:
  ```jsx
  const inputRef = useRef(null);

  useEffect(() => {
    // Focus input when component mounts or specific condition
    inputRef.current?.focus();
  }, [dependency]);

  return <input ref={inputRef} type="text" />
  ```

### 3.4 React 19 Ref Simplification
- **Change**: React 19 allows passing refs as props directly, eliminating need for `forwardRef` in many cases
- **Pattern**:
  ```jsx
  // React 19+
  function CustomInput({ ref, ...props }) {
    return <input ref={ref} {...props} />
  }
  ```

### 3.5 Focus Persistence
- **Rule**: Ensure `document.activeElement` is always meaningful (not null or body element)
- **Requirement**: Manage events affecting the currently active element so focus remains visible and moves logically

## 4. Modal Dialogs & Focus Traps

### 4.1 Focus Trap Requirements
- **Rule**: When a modal/dialog opens, trap keyboard focus within the modal until it closes
- **Implementation**: Use `focus-trap-react` library or custom focus management
- **Pattern**:
  ```jsx
  import FocusTrap from 'focus-trap-react';

  <FocusTrap>
    <div role="dialog" aria-modal="true" aria-labelledby="dialog-title">
      {/* modal content */}
    </div>
  </FocusTrap>
  ```

### 4.2 Modal ARIA Attributes
- **Required**:
  - `role="dialog"` or `role="alertdialog"`
  - `aria-modal="true"` (prevents interaction with background)
  - `aria-labelledby` (references title element ID)
  - `aria-describedby` (references description element ID, optional)

### 4.3 Modal Keyboard Behavior
- **Escape Key**: Must close the modal
- **Focus Return**: When modal closes, focus must return to the element that triggered it
- **Pattern**:
  ```jsx
  const triggerRef = useRef(null);
  const [isOpen, setIsOpen] = useState(false);

  const openModal = () => {
    triggerRef.current = document.activeElement;
    setIsOpen(true);
  };

  const closeModal = () => {
    setIsOpen(false);
    triggerRef.current?.focus();
  };
  ```

### 4.4 Browser Dialog Handling
- **Rule**: Provide `handleDialog` for browser alerts, confirms, and prompts
- **Actions**: Accept or dismiss dialogs, optionally provide prompt text

## 5. Form Accessibility

### 5.1 Label Association
- **Rule**: Every form input must have an associated `<label>` element
- **Pattern**: Use `htmlFor` (not `for`) in React to connect label to input
  ```jsx
  <label htmlFor="email">Email Address</label>
  <input id="email" type="email" name="email" />
  ```
- **Alternative**: Use `aria-label` or `aria-labelledby` only when visible label is not possible

### 5.2 Fieldset and Legend
- **Rule**: Use `<fieldset>` and `<legend>` for grouping related form controls
- **Use Cases**: Radio button groups, checkbox groups, related inputs
- **Pattern**:
  ```jsx
  <fieldset>
    <legend>Shipping Method</legend>
    <label><input type="radio" name="shipping" value="standard" /> Standard</label>
    <label><input type="radio" name="shipping" value="express" /> Express</label>
  </fieldset>
  ```

### 5.3 Autocomplete Attribute
- **Rule**: Use `autocomplete` attribute for personal information inputs
- **Purpose**: Helps users with cognitive disabilities and password managers
- **Common Values**: `name`, `email`, `tel`, `address-line1`, `postal-code`, `cc-number`

### 5.4 Form Validation and Error Handling

#### 5.4.1 aria-invalid
- **Rule**: Set `aria-invalid="true"` on form controls with validation errors
- **Pattern**:
  ```jsx
  <input
    type="email"
    aria-invalid={hasError}
    aria-describedby={hasError ? "email-error" : undefined}
  />
  {hasError && <span id="email-error">Please enter a valid email</span>}
  ```

#### 5.4.2 aria-describedby
- **Rule**: Use `aria-describedby` to associate error messages with form controls
- **Behavior**: Screen readers announce the error message when input receives focus
- **Limitation**: Changes to `aria-describedby` content are not announced dynamically; use ARIA live regions for dynamic announcements

#### 5.4.3 aria-errormessage
- **Rule**: Alternative to `aria-describedby` for error messages (requires `aria-invalid="true"`)
- **Browser Support**: Check compatibility before use

#### 5.4.4 Validation Timing Patterns

**Pattern 1: Validate on Submit**
- Display all errors in a list at top of form
- Use `role="alert"` on error container for immediate announcement
- Move focus to error summary
  ```jsx
  {errors.length > 0 && (
    <div role="alert" tabIndex={-1} ref={errorSummaryRef}>
      <h2>Please fix the following errors:</h2>
      <ul>
        {errors.map(error => <li key={error.field}>{error.message}</li>)}
      </ul>
    </div>
  )}
  ```

**Pattern 2: Validate on Blur**
- Validate individual inputs when user tabs away
- Set `aria-invalid` and display inline error
- Announce error using live region if needed

#### 5.4.5 Error Message Requirements (WCAG)
- Errors must not rely solely on color
- Provide clear, specific error messages
- Include error icon or visual indicator in addition to color
- Ensure error text has sufficient color contrast (4.5:1 minimum)

### 5.5 React 19 Form Actions

#### 5.5.1 useFormStatus Hook
- **Purpose**: Provides form submission status without prop drilling
- **Pattern**:
  ```jsx
  import { useFormStatus } from 'react-dom';

  function SubmitButton() {
    const { pending } = useFormStatus();
    return (
      <button type="submit" disabled={pending} aria-busy={pending}>
        {pending ? 'Submitting...' : 'Submit'}
      </button>
    );
  }
  ```
- **Accessibility**: Use `aria-busy` to indicate loading state to screen readers

#### 5.5.2 useActionState Hook
- **Purpose**: Manages form state updated by form actions
- **Pattern**:
  ```jsx
  import { useActionState } from 'react';

  function Form() {
    const [state, formAction] = useActionState(submitAction, initialState);

    return (
      <form action={formAction}>
        {state.error && (
          <div role="alert" aria-live="polite">{state.error}</div>
        )}
        {/* form fields */}
      </form>
    );
  }
  ```
- **Accessibility**: Announce state changes with `role="alert"` or `aria-live` regions

#### 5.5.3 Automatic Form Reset
- **Behavior**: React 19 form elements with actions automatically reset after submission
- **Accessibility**: Ensure focus management is handled when form resets

## 6. Screen Reader Compatibility

### 6.1 ARIA Live Regions

#### 6.1.1 Purpose
- Announce dynamic content changes to screen reader users
- Required for SPA updates, form validation, status messages, notifications

#### 6.1.2 aria-live Values
- `aria-live="polite"`: Announces update when screen reader is idle (most common)
- `aria-live="assertive"`: Announces immediately, interrupting current speech (use sparingly)
- `aria-live="off"`: Default, no announcements

#### 6.1.3 Implementation Requirements
- **Critical**: Live region must exist in DOM before content changes
- Rendering live region for first time with content will NOT be announced
- **Pattern**:
  ```jsx
  // Live region exists from start (empty)
  <div aria-live="polite" aria-atomic="true" className="sr-only">
    {announcement}
  </div>

  // Update announcement dynamically
  setAnnouncement('Item added to cart');
  ```

#### 6.1.4 role="alert"
- **Equivalent**: `role="alert"` = `aria-live="assertive"` + `aria-atomic="true"`
- **Use Case**: Important, time-sensitive messages
- **Pattern**:
  ```jsx
  {errorMessage && (
    <div role="alert">{errorMessage}</div>
  )}
  ```

#### 6.1.5 Best Practices
- Keep announcements short and concise
- Don't announce every keystroke or rapid change
- Start with empty live regions on page load
- Use `aria-atomic="true"` to announce entire region, not just changes
- Consider `aria-relevant` to control what changes are announced (default: "additions text")

### 6.2 Visually Hidden Content (Screen Reader Only)
- **Pattern**:
  ```css
  .sr-only {
    position: absolute;
    width: 1px;
    height: 1px;
    padding: 0;
    margin: -1px;
    overflow: hidden;
    clip: rect(0, 0, 0, 0);
    white-space: nowrap;
    border-width: 0;
  }
  ```
- **Usage**: Provide additional context for screen readers without visual clutter
- **Example**: "Opens in new window", "Required field", "Sort ascending"

### 6.3 Testing with Screen Readers
- **Required Testing**:
  - **VoiceOver** (macOS, iOS): Built-in Apple accessibility
  - **JAWS** (Windows): Most popular commercial screen reader
  - **NVDA** (Windows): Free, widely used
  - **TalkBack** (Android): Mobile screen reader
- **Testing Approach**: Navigate entire application using only screen reader, verify all content and interactions are accessible

### 6.4 Dynamic Routing in SPAs
- **Problem**: Screen readers don't automatically announce route changes
- **Solution**: Announce route changes using ARIA live regions
- **Pattern**:
  ```jsx
  useEffect(() => {
    announceRef.current.textContent = `Navigated to ${pageTitle}`;
  }, [location.pathname]);

  return (
    <div aria-live="assertive" aria-atomic="true" className="sr-only" ref={announceRef} />
  );
  ```

## 7. Skip Links & Navigation

### 7.1 Skip Navigation Links
- **Rule**: Provide "Skip to main content" link as first focusable element
- **Purpose**: Allows keyboard users to bypass repetitive navigation
- **Pattern**:
  ```jsx
  <a href="#main" className="skip-link">
    Skip to main content
  </a>
  {/* navigation */}
  <main id="main" tabIndex={-1}>
    {/* main content */}
  </main>
  ```

### 7.2 Skip Link Styling
- **Requirement**: Link should be visible when focused
- **Pattern**:
  ```css
  .skip-link {
    position: absolute;
    left: -10000px;
    top: auto;
    width: 1px;
    height: 1px;
    overflow: hidden;
  }

  .skip-link:focus {
    position: static;
    width: auto;
    height: auto;
  }
  ```

### 7.3 Multiple Skip Links
- **Pattern**: Provide skip links to multiple landmarks for complex pages
- **Examples**: "Skip to navigation", "Skip to search", "Skip to footer"

### 7.4 SPA Focus Management on Route Change
- **Problem**: Focus remains on clicked link after client-side navigation
- **Solution**: Move focus to main content or page heading
- **Pattern**:
  ```jsx
  const mainRef = useRef(null);

  useEffect(() => {
    mainRef.current?.focus();
  }, [location.pathname]);

  return <main ref={mainRef} tabIndex={-1}>
  ```

## 8. Images & Media

### 8.1 Image Alt Text
- **Rule**: All `<img>` elements must have `alt` attribute
- **Descriptive Alt**: Describe image purpose/content for meaningful images
- **Null Alt**: Use `alt=""` for decorative images
- **Complex Images**: Provide longer description via `aria-describedby` or adjacent text
- **Images with Text**: Include text content in alt text

### 8.2 Icon-Only Buttons
- **Rule**: Provide accessible label for icon buttons
- **Pattern**:
  ```jsx
  <button aria-label="Close dialog">
    <CloseIcon aria-hidden="true" />
  </button>
  ```

### 8.3 Media Requirements
- **Video**: Provide captions (required) and audio descriptions (recommended)
- **Audio**: Provide transcripts
- **Autoplay**: Avoid autoplay; if necessary, provide pause/stop controls
- **Flashing Content**: Never include content that flashes more than 3 times per second (seizure risk)

### 8.4 Media Controls
- **Rule**: Provide accessible, keyboard-operable controls
- **Pause Mechanism**: Allow users to pause, stop, or hide moving/animated content

## 9. Tables & Data Grids

### 9.1 Semantic Table Elements
- **Rule**: Use `<table>` for tabular data, never for layout
- **Required Elements**:
  - `<caption>` for table title/description
  - `<thead>`, `<tbody>`, `<tfoot>` for structure
  - `<th>` for header cells with `scope` attribute

### 9.2 Table Header Scope
- **Rule**: Use `scope` attribute on `<th>` elements
- **Values**: `scope="col"` for column headers, `scope="row"` for row headers
- **Pattern**:
  ```jsx
  <table>
    <caption>Sales Report Q1 2025</caption>
    <thead>
      <tr>
        <th scope="col">Product</th>
        <th scope="col">Sales</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <th scope="row">Widget A</th>
        <td>$10,000</td>
      </tr>
    </tbody>
  </table>
  ```

### 9.3 Sortable Tables
- **Rule**: Add `aria-sort` attribute to sortable column headers
- **Values**: `aria-sort="ascending"`, `aria-sort="descending"`, `aria-sort="none"`
- **Pattern**:
  ```jsx
  <th scope="col" aria-sort={sortColumn === 'name' ? sortOrder : 'none'}>
    <button onClick={() => handleSort('name')}>
      Name {sortColumn === 'name' && <SortIcon order={sortOrder} />}
    </button>
  </th>
  ```

### 9.4 Data Grid ARIA Patterns
- **Role**: Use `role="grid"` for interactive data tables
- **Child Roles**: `role="row"`, `role="columnheader"`, `role="gridcell"`
- **Roving Tabindex**: Implement roving tabindex for cell navigation
- **Arrow Key Navigation**: Support arrow keys for grid navigation
- **React Aria**: Consider using React Aria's Table component for full ARIA grid implementation

## 10. Interactive Components

### 10.1 Buttons vs Links
- **Buttons**: Use `<button>` for actions (submit, open, toggle)
- **Links**: Use `<a>` for navigation (internal pages, external URLs)
- **Rule**: Never use `<div>` or `<span>` as clickable elements
- **Pattern**:
  ```jsx
  // ✅ Correct
  <button onClick={handleClick}>Save</button>
  <a href="/about">About Us</a>

  // ❌ Wrong
  <div onClick={handleClick}>Save</div>
  <span onClick={handleClick}>About Us</span>
  ```

### 10.2 Links Opening in New Windows
- **Rule**: Indicate when links open in new windows/tabs
- **Pattern**:
  ```jsx
  <a href="https://example.com" target="_blank" rel="noopener noreferrer">
    External Site <span className="sr-only">(opens in new window)</span>
  </a>
  ```

### 10.3 Combobox/Autocomplete
- **Rule**: Follow W3C ARIA Combobox pattern
- **Required ARIA**:
  - `role="combobox"` on input
  - `aria-autocomplete="list"` or `"both"`
  - `aria-expanded` to indicate dropdown state
  - `aria-controls` to reference listbox ID
  - `aria-activedescendant` to reference active option
- **Libraries**: Use React Aria ComboBox, Headless UI Combobox, or Material UI Autocomplete

### 10.4 Dropdown Menus
- **Rule**: Follow W3C ARIA Menu pattern
- **Keyboard**: Arrow keys navigate, Enter/Space activates, Escape closes
- **Required ARIA**:
  - `role="menu"` on container
  - `role="menuitem"` on items
  - `aria-expanded` on trigger button
  - `aria-haspopup="menu"` on trigger

### 10.5 Tabs
- **Rule**: Follow W3C ARIA Tabs pattern
- **Required ARIA**:
  - `role="tablist"` on container
  - `role="tab"` on tab elements
  - `role="tabpanel"` on panel elements
  - `aria-selected` on active tab
  - `aria-controls` linking tab to panel
  - `aria-labelledby` linking panel to tab
- **Keyboard**: Arrow keys navigate tabs, Tab enters panel content

### 10.6 Accordions
- **Rule**: Use `<button>` for accordion headers
- **Required ARIA**:
  - `aria-expanded` on button to indicate state
  - `aria-controls` linking button to panel ID
  - Optional: `role="region"` and `aria-labelledby` on panel

### 10.7 Tooltips
- **Rule**: Use `aria-describedby` to associate tooltip with trigger
- **Pattern**:
  ```jsx
  <button aria-describedby="tooltip-1" onMouseEnter={showTooltip} onFocus={showTooltip}>
    Help
  </button>
  {visible && <div id="tooltip-1" role="tooltip">Additional information</div>}
  ```
- **Keyboard**: Show tooltip on focus, hide on blur or Escape

## 11. Color & Appearance

### 11.1 Color Contrast (WCAG 2.2)
- **Level AA Requirements**:
  - Normal text: 4.5:1 contrast ratio
  - Large text (18.66px+ bold or 24px+): 3:1 contrast ratio
  - UI components and interactive elements: 3:1 contrast ratio
- **Level AAA Requirements**:
  - Normal text: 7:1 contrast ratio
  - Large text: 4.5:1 contrast ratio
- **Exceptions**: Logos, inactive components, decorative elements

### 11.2 Color as Sole Indicator
- **Rule**: Never use color alone to convey information
- **Requirement**: Provide additional visual indicators (icons, text, patterns)
- **Example**: Error states should use both red color AND error icon/text

### 11.3 Text Resizing
- **Rule**: Ensure content remains readable at 200% text size
- **Implementation**: Use relative units (rem, em) instead of fixed pixels
- **Testing**: Zoom to 200% in browser and verify layout doesn't break

### 11.4 Viewport Zoom
- **Rule**: Do not prevent viewport zooming
- **Pattern**:
  ```html
  <!-- ✅ Correct -->
  <meta name="viewport" content="width=device-width, initial-scale=1.0">

  <!-- ❌ Wrong -->
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
  ```

### 11.5 Text Alignment
- **Rule**: Left-align text for LTR languages, right-align for RTL languages
- **Avoid**: Centered or justified text for body content (reduces readability)

### 11.6 Focus Visible vs Focus
- **Rule**: Use `:focus-visible` pseudo-class for keyboard focus, not `:focus`
- **Rationale**: `:focus-visible` only shows focus ring for keyboard navigation, not mouse clicks
- **Pattern**:
  ```css
  button:focus-visible {
    outline: 2px solid #0066cc;
  }
  ```

## 12. Animation & Motion

### 12.1 prefers-reduced-motion
- **Rule**: Respect user's motion preferences
- **Implementation**: Use CSS media query or JavaScript detection
- **CSS Pattern**:
  ```css
  @media (prefers-reduced-motion: reduce) {
    * {
      animation-duration: 0.01ms !important;
      animation-iteration-count: 1 !important;
      transition-duration: 0.01ms !important;
    }
  }
  ```
- **React Hook Pattern**:
  ```jsx
  function usePrefersReducedMotion() {
    const [prefersReduced, setPrefersReduced] = useState(false);

    useEffect(() => {
      const mediaQuery = window.matchMedia('(prefers-reduced-motion: reduce)');
      setPrefersReduced(mediaQuery.matches);

      const listener = (e) => setPrefersReduced(e.matches);
      mediaQuery.addEventListener('change', listener);
      return () => mediaQuery.removeEventListener('change', listener);
    }, []);

    return prefersReduced;
  }
  ```

### 12.2 Safe Animation Alternatives
- **Rule**: Replace motion-based animations with non-motion effects for users with motion sensitivity
- **Safe Effects**: Fade in/out, dissolve, color changes
- **Avoid**: Scaling, rotating, sliding, parallax, wave effects

### 12.3 Pause Mechanism
- **Rule**: Provide pause/stop controls for moving, blinking, or scrolling content
- **WCAG Requirement**: Content that moves for more than 5 seconds must be pausable

### 12.4 Autoplay
- **Rule**: Avoid autoplay for videos and carousels
- **Exception**: If autoplay is necessary, provide pause/stop controls immediately

## 13. Content & Language

### 13.1 Page Language
- **Rule**: Set `lang` attribute on `<html>` element
- **Pattern**: `<html lang="en">`
- **Purpose**: Helps screen readers pronounce content correctly

### 13.2 Language Changes
- **Rule**: Mark language changes within content using `lang` attribute
- **Pattern**:
  ```jsx
  <p>The French word for hello is <span lang="fr">bonjour</span>.</p>
  ```

### 13.3 Page Titles
- **Rule**: Every page must have a unique, descriptive `<title>`
- **Pattern**: `<title>Dashboard - MyApp</title>`
- **React 19**: Use `<title>` directly in components (automatically hoisted to `<head>`)
  ```jsx
  function Dashboard() {
    return (
      <>
        <title>Dashboard - MyApp</title>
        {/* component content */}
      </>
    );
  }
  ```

### 13.4 Metadata
- **React 19 Feature**: `<meta>`, `<link>`, and `<title>` tags are automatically hoisted to `<head>`
- **Pattern**:
  ```jsx
  function Page() {
    return (
      <>
        <title>Page Title</title>
        <meta name="description" content="Page description" />
        {/* content */}
      </>
    );
  }
  ```

### 13.5 Heading Structure
- **Rule**: Use one `<h1>` per page
- **Rule**: Create logical heading hierarchy (don't skip levels)
- **Correct**: h1 → h2 → h3 → h2 → h3
- **Wrong**: h1 → h3 → h5 → h2

### 13.6 Plain Language
- **Rule**: Write content at 8th grade reading level
- **Avoid**: Jargon, figures of speech, idioms, complex metaphors
- **Purpose**: Improves comprehension for users with cognitive disabilities and non-native speakers

### 13.7 Link and Button Text
- **Rule**: Make link/button text unique and descriptive
- **Avoid**: Generic text like "Click here", "Read more", "Learn more"
- **Pattern**:
  ```jsx
  // ❌ Poor
  <a href="/report">Click here</a>

  // ✅ Better
  <a href="/report">Download Q1 2025 Sales Report</a>
  ```

### 13.8 Lists
- **Rule**: Use appropriate list elements for lists
- **Elements**: `<ul>` for unordered lists, `<ol>` for ordered lists, `<dl>` for definition lists
- **Rationale**: Screen readers announce list length and position

## 14. Session Management

### 14.1 Timeout Extensions
- **Rule**: Allow users to extend session timeouts
- **WCAG Requirement**: Users must be able to turn off, adjust, or extend time limits
- **Pattern**: Warn users before timeout and provide extension option

### 14.2 Data Preservation
- **Rule**: Preserve user data when session expires
- **Implementation**: Use localStorage or sessionStorage to save form data

## 15. Testing & Tools

### 15.1 Automated Testing Tools

#### 15.1.1 ESLint Plugin
- **Tool**: `eslint-plugin-jsx-a11y`
- **Purpose**: Static analysis of JSX for accessibility issues
- **Installation**: Included by default in Create React App
- **Modes**: Recommended or strict
- **Limitation**: Only catches static code issues

#### 15.1.2 Axe-core
- **Tool**: `@axe-core/react` (development) or `axe-core` (testing)
- **Purpose**: Runtime accessibility testing of rendered DOM
- **Usage**: Add to development build for console warnings
- **Pattern**:
  ```jsx
  if (process.env.NODE_ENV !== 'production') {
    import('@axe-core/react').then(axe => {
      axe.default(React, ReactDOM, 1000);
    });
  }
  ```

#### 15.1.3 Jest-axe
- **Tool**: `jest-axe`
- **Purpose**: Accessibility testing in Jest tests
- **Pattern**:
  ```jsx
  import { axe, toHaveNoViolations } from 'jest-axe';
  expect.extend(toHaveNoViolations);

  it('should have no accessibility violations', async () => {
    const { container } = render(<MyComponent />);
    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });
  ```

#### 15.1.4 Lighthouse
- **Tool**: Chrome DevTools Lighthouse
- **Purpose**: Audits performance, accessibility, SEO
- **Usage**: Run in Chrome DevTools > Lighthouse tab

### 15.2 Manual Testing Requirements
- **Limitation**: Automated tools catch only ~30% of accessibility issues
- **Required**: Manual testing with keyboard navigation and screen readers
- **Approach**: Test entire user flow with assistive technologies

### 15.3 Keyboard Testing
- **Test**: Navigate entire application using only keyboard
- **Check**:
  - All interactive elements are focusable
  - Focus order follows logical reading order
  - Focus is visible at all times
  - No keyboard traps
  - All actions are keyboard-accessible

### 15.4 Screen Reader Testing Matrix
- **Windows**: Test with JAWS and NVDA + Chrome/Firefox
- **macOS**: Test with VoiceOver + Safari
- **iOS**: Test with VoiceOver + Safari
- **Android**: Test with TalkBack + Chrome

### 15.5 Browser Testing Tools
- **Chrome DevTools**:
  - Accessibility tree inspection
  - Contrast ratio checker
  - Lighthouse audits
- **Firefox DevTools**:
  - Accessibility inspector
  - Keyboard navigation simulation

### 15.6 CI/CD Integration
- **Rule**: Integrate accessibility tests into CI/CD pipeline
- **Tools**: axe-core with Jest or Cypress
- **Pattern**: Fail builds on accessibility violations

## 16. Component Libraries

### 16.1 React Aria (Recommended)
- **Maintainer**: Adobe
- **Scope**: 50+ accessible, unstyled components
- **Standards**: Implements WAI-ARIA specifications and W3C patterns
- **Testing**: Tested across VoiceOver, JAWS, NVDA, TalkBack
- **Components**: ComboBox, Autocomplete, Table, Menu, Tabs, Dialog, etc.
- **Usage**: Provides hooks (useComboBox, useTable) and components

### 16.2 Headless UI
- **Maintainer**: Tailwind Labs
- **Scope**: Unstyled, accessible UI components
- **Components**: Combobox, Menu, Dialog, Tabs, Disclosure, Listbox
- **Integration**: Works seamlessly with Tailwind CSS

### 16.3 Reach UI
- **Scope**: Accessible component primitives
- **Components**: Combobox, Dialog, Menu, Tabs, Tooltip, Accordion

### 16.4 Material UI
- **Scope**: Full component library with accessibility built-in
- **Standards**: Implements WAI-ARIA patterns
- **Components**: Autocomplete, DataGrid, Menu, Tabs, Dialog, etc.

### 16.5 When to Use Libraries
- **Recommendation**: Use established accessible component libraries for complex patterns (comboboxes, data grids, menus)
- **Rationale**: Accessibility is complex; well-tested libraries reduce implementation errors
- **Custom Components**: Only build custom when necessary; always follow WAI-ARIA patterns

## 17. React-Specific Patterns

### 17.1 Fragments for Semantic Markup
- **Rule**: Use React Fragments to avoid unnecessary `<div>` wrappers
- **Pattern**:
  ```jsx
  // ✅ Maintains semantic structure
  <dl>
    {items.map(item => (
      <React.Fragment key={item.id}>
        <dt>{item.term}</dt>
        <dd>{item.definition}</dd>
      </React.Fragment>
    ))}
  </dl>

  // ❌ Breaks semantic structure
  <dl>
    {items.map(item => (
      <div key={item.id}>
        <dt>{item.term}</dt>
        <dd>{item.definition}</dd>
      </div>
    ))}
  </dl>
  ```

### 17.2 Conditional Rendering and Accessibility
- **Rule**: Ensure accessibility attributes remain consistent during conditional rendering
- **Pattern**: Don't conditionally render ARIA live regions; update their content instead

### 17.3 Event Handlers for Keyboard and Mouse
- **Rule**: Support both keyboard and pointer events
- **Pattern**:
  ```jsx
  <div
    onClick={handleClick}
    onKeyDown={(e) => {
      if (e.key === 'Enter' || e.key === ' ') {
        e.preventDefault();
        handleClick();
      }
    }}
    role="button"
    tabIndex={0}
  >
    Custom Button
  </div>
  ```
- **Better**: Use `<button>` element which handles this automatically

### 17.4 HTML Validation
- **Rule**: Validate HTML structure
- **Tools**: W3C Markup Validator, HTMLHint
- **Common Issues**: Nested interactive elements, invalid ARIA, missing attributes

## 18. Enterprise Considerations

### 18.1 Accessibility Policy
- **Requirement**: Define and document accessibility standards for organization
- **Target**: WCAG 2.2 Level AA (minimum for most enterprises)
- **Compliance**: Consider legal requirements (ADA, Section 508, EAA)

### 18.2 Accessibility Statement
- **Rule**: Provide accessibility statement on website
- **Contents**: Conformance level, known issues, contact information for reporting issues

### 18.3 Training & Documentation
- **Requirement**: Train developers on accessibility best practices
- **Resources**: W3C tutorials, A11y Project, React Aria documentation

### 18.4 Design System Integration
- **Rule**: Build accessibility into design system components
- **Approach**: Create reusable, accessible components for entire organization

### 18.5 Accessibility Champions
- **Pattern**: Designate accessibility champions on each team
- **Responsibility**: Review PRs for accessibility, advocate for users

### 18.6 Budget & Timeline
- **Rule**: Include accessibility in project planning and estimates
- **Reality**: Retrofitting accessibility is more expensive than building it in

## 19. Common Pitfalls & Anti-Patterns

### 19.1 Using divs as Buttons
- **Anti-pattern**: `<div onClick={handler}>Click me</div>`
- **Issues**: Not keyboard accessible, not announced as button, no default styling
- **Fix**: Use `<button>`

### 19.2 Placeholder as Label
- **Anti-pattern**: `<input placeholder="Email" />`
- **Issues**: Placeholder disappears on input, low contrast, not a substitute for label
- **Fix**: Use `<label>` element

### 19.3 Removing Focus Outlines
- **Anti-pattern**: `* { outline: none; }`
- **Issues**: Keyboard users can't see focus
- **Fix**: Use `:focus-visible` and provide custom focus styles

### 19.4 Icon-Only Buttons Without Labels
- **Anti-pattern**: `<button><Icon /></button>`
- **Issues**: Screen readers can't determine button purpose
- **Fix**: Add `aria-label` or visually hidden text

### 19.5 Redundant ARIA Roles
- **Anti-pattern**: `<button role="button">`
- **Issues**: Unnecessary and potentially confusing
- **Fix**: Remove redundant role; native semantics are sufficient

### 19.6 Improper Heading Hierarchy
- **Anti-pattern**: Using headings for styling (h1 for large text)
- **Issues**: Breaks document structure for screen readers
- **Fix**: Use correct heading levels; style with CSS

### 19.7 Empty Links and Buttons
- **Anti-pattern**: `<a href="#">...</a>` or `<button></button>`
- **Issues**: Screen readers can't describe purpose
- **Fix**: Provide meaningful text content or aria-label

### 19.8 Non-Focusable Interactive Elements
- **Anti-pattern**: `<span onClick={handler}>Click</span>`
- **Issues**: Not keyboard accessible
- **Fix**: Use button or add `tabIndex={0}` and keyboard handlers

### 19.9 Incorrect ARIA Usage
- **Anti-pattern**: Conflicting ARIA roles with native elements
- **Issues**: Confuses assistive technology
- **Fix**: Follow "No ARIA is better than Bad ARIA" principle

## 20. Future Considerations

### 20.1 WCAG 3.0
- **Status**: Working Draft, expected ~2026
- **Changes**: New conformance model, different testing approach
- **Preparation**: Continue following WCAG 2.2; 3.0 will be backward compatible

### 20.2 React Future Features
- **React Compiler**: Automatic memoization (no accessibility impact)
- **Server Components**: Ensure accessibility in hybrid client/server architectures
- **Concurrent Rendering**: No direct accessibility impact

### 20.3 Evolving Standards
- **Rule**: Stay updated with W3C WAI-ARIA and React documentation
- **Resources**: W3C WAI, React blog, A11y Project

## Summary Checklist

### Essential Requirements
- [ ] Use semantic HTML elements
- [ ] Provide text alternatives for images
- [ ] Ensure keyboard navigation works throughout
- [ ] Provide visible focus indicators (never remove outlines)
- [ ] Associate labels with form inputs
- [ ] Use ARIA attributes correctly (or not at all)
- [ ] Ensure sufficient color contrast (4.5:1 text, 3:1 UI)
- [ ] Don't use color alone to convey information
- [ ] Provide captions for videos
- [ ] Respect prefers-reduced-motion
- [ ] Create logical heading structure
- [ ] Implement ARIA live regions for dynamic content
- [ ] Test with keyboard navigation
- [ ] Test with screen readers
- [ ] Run automated accessibility tests
- [ ] Trap focus in modals
- [ ] Return focus appropriately
- [ ] Provide skip links
- [ ] Allow viewport zoom
- [ ] Use plain language (8th grade level)

### React 19 Specific
- [ ] Use refs directly as props (no forwardRef needed)
- [ ] Implement useFormStatus for form states
- [ ] Implement useActionState for form actions
- [ ] Use native `<title>` and `<meta>` tags in components
- [ ] Announce dynamic updates with ARIA live regions
- [ ] Set aria-busy on pending form submissions
- [ ] Manage focus in client-side routing

## References

### Official Standards
- **W3C WCAG 2.2**: https://www.w3.org/TR/WCAG22/
- **W3C WAI-ARIA 1.3**: https://w3c.github.io/aria/
- **WAI-ARIA Authoring Practices Guide**: https://www.w3.org/WAI/ARIA/apg/

### React Resources
- **React Accessibility Docs**: https://react.dev/ (React 19 docs)
- **React Legacy A11y Docs**: https://legacy.reactjs.org/docs/accessibility.html
- **React Aria**: https://react-spectrum.adobe.com/react-aria/

### Community Resources
- **The A11y Project**: https://www.a11yproject.com/
- **WebAIM**: https://webaim.org/

### Tools
- **axe DevTools**: Browser extension
- **eslint-plugin-jsx-a11y**: https://github.com/jsx-eslint/eslint-plugin-jsx-a11y
- **jest-axe**: https://www.npmjs.com/package/jest-axe
- **WebAIM Contrast Checker**: https://webaim.org/resources/contrastchecker/

---

**Last Updated**: January 2025
**React Version**: 19.x
**WCAG Version**: 2.2 (Level AA focus)
**Target Audience**: Enterprise development teams
