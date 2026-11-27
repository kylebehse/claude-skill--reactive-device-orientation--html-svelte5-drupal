---
name: device-orientation
description: Implement zero-repaint, SSR-safe device orientation detection (portrait/landscape) across vanilla HTML/CSS, SvelteKit (Svelte 5 runes), and Drupal 10. Use when developing for device rotation, screen orientation classes, matchMedia listeners, or fixing orientation hydration flicker.
allowed-tools: Read, Edit, Write, Glob, Grep
---

# Device Orientation & Rotation Standards

When implementing screen rotation or `portrait` / `landscape` states, follow this decision hierarchy to prevent initial page repaints, avoid layout thrashing, and eliminate SSR hydration flicker.

## 1. Decision Tree

1. **Styling-only changes (no JS logic or legacy `<body>` class needed):**
   - Use native CSS `@media (orientation: portrait)` and `@media (orientation: landscape)`.
   - **Never** add JavaScript listeners if pure CSS solves the requirement.
2. **Legacy or third-party CSS requiring `.portrait` / `.landscape` on `<body>` or `<html>`:**
   - Use a synchronous blocking inline `<script>` in the initial HTML shell via `window.matchMedia('(orientation: portrait)')`.
   - **Never** attach listeners to `window.resize` or read `window.innerWidth` / `innerHeight`.
3. **Reactive JavaScript state required (Canvas, gesture thresholds, conditional logic):**
   - Use framework-native primitives (`MediaQuery` in Svelte 5) or `Drupal.behaviors` (Drupal 10), paired with the blocking HTML shell script when initial paint consistency is required.

---

## 2. Vanilla HTML / Zero-Repaint Pattern

Execute `matchMedia` synchronously before the browser paints the DOM to prevent a Flash of Unstyled Content (FOUC):

- **Preferred (`<html>` target inside `<head>`):**
  ```html
  <head>
    <script>
      (function () {
        const query = window.matchMedia('(orientation: portrait)');
        const update = (e) => {
          document.documentElement.classList.toggle('portrait', e.matches);
          document.documentElement.classList.toggle('landscape', !e.matches);
        };
        update(query);
        query.addEventListener('change', update);
      })();
    </script>
  </head>
  ```
- **When `<body>` is strictly required:** Place the `<script>` as the **very first child inside `<body>`** (never in `<head>`, where `document.body` is `null`).

---

## 3. SvelteKit & Svelte 5 (Runes)

Inspect `package.json` to confirm Svelte 5 (`"svelte": "^5.0.0"`), then apply a two-layer setup:

### Step A: `src/app.html` (Zero-Repaint CSS Class)
Place the blocking script immediately inside `<body>` above `%sveltekit.body%`:
```html
<body data-sveltekit-preload-data="hover">
  <script>
    (function () {
      const query = window.matchMedia('(orientation: portrait)');
      const update = (e) => {
        document.body.classList.toggle('portrait', e.matches);
        document.body.classList.toggle('landscape', !e.matches);
      };
      update(query);
      query.addEventListener('change', update);
    })();
  </script>
  <div style="display: contents">%sveltekit.body%</div>
</body>
```
- In component `<style>` blocks, wrap body selectors in `:global(body.portrait)` and `:global(body.landscape)`.

### Step B: Reactive Rune Module (`src/lib/orientation.svelte.ts`)
Use the `.svelte.ts` extension and `MediaQuery` from `svelte/reactivity`:
```typescript
import { MediaQuery } from 'svelte/reactivity';

// Second argument sets SSR fallback boolean (defaults to false)
const portraitQuery = new MediaQuery('(orientation: portrait)');

export const orientation = {
  get isPortrait() {
    return portraitQuery.current;
  },
  get isLandscape() {
    return !portraitQuery.current;
  }
};
```

### Step C: Preventing SSR Hydration Flicker
Choose the mitigation based on what is being rendered:
1. **SEO / Layout blocks:** Render both branches and toggle visibility with CSS (`:global(body.landscape) .portrait-only { display: none; }`).
2. **Mobile-first apps:** Pass `true` as the SSR fallback (`new MediaQuery('(orientation: portrait)', true)`).
3. **Heavy client-only components (Canvas/WebGL):** Gate rendering behind `import { browser } from '$app/environment'` (`{#if browser}...{/if}`).

---

## 4. Drupal 10

Respect Drupal's deferred asset aggregation and BigPipe/AJAX lifecycle:

### Step A: `html.html.twig` (Zero-Repaint Initial Class)
In the active custom theme's `templates/layout/html.html.twig`, inject the initial class synchronously right after `<body{{ attributes }}>`:
```twig
<body{{ attributes }}>
  <script>
    (function () {
      const isPortrait = window.matchMedia('(orientation: portrait)').matches;
      document.body.classList.add(isPortrait ? 'portrait' : 'landscape');
    })();
  </script>
  {{ page_top }}
  {{ page }}
  {{ page_bottom }}
</body>
```
- **CSP Fallback:** If the site enforces a strict Content Security Policy blocking inline scripts, move this into a JS file attached via a library with `header: true` and target `document.documentElement` (`<html>`) instead of `document.body`.

### Step B: `Drupal.behaviors` + `core/once`
Register a library in `<theme>.libraries.yml` with dependencies `core/drupal` and `core/once` (never `jQuery.once`):
```javascript
(function (Drupal, once) {
  'use strict';

  Drupal.behaviors.deviceOrientation = {
    attach: function (context) {
      once('device-orientation', 'body', context).forEach(function (body) {
        const query = window.matchMedia('(orientation: portrait)');
        const updateClass = function (e) {
          body.classList.toggle('portrait', e.matches);
          body.classList.toggle('landscape', !e.matches);
        };
        updateClass(query);
        query.addEventListener('change', updateClass);
      });
    }
  };
})(Drupal, once);
```

---

## 5. Guardrails
- **Never** bind reactive framework classes directly to `<body>` or `<html>` during SSR (e.g., `<svelte:body>`), as it causes initial hydration flicker.
- **Never** mix `@media (orientation: ...)` and `body.portrait` / `body.landscape` rules for the same component styles.
- **Never** reference `document.body` inside `<head>` or a Drupal `header: true` library; always use `document.documentElement`.
