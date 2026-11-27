# Device Orientation Standards

This repository provides implementation guidance and reusable patterns for
detecting portrait and landscape orientation without initial paint flicker.
It is designed for:

- vanilla HTML and CSS
- SvelteKit with Svelte 5 runes
- Drupal 10 themes

The guidance prioritizes the browser's native `matchMedia` API. It avoids
orientation checks based on `window.innerWidth`, `window.innerHeight`, or
`window.resize` event listeners.

## Repository contents

- [`device-orientation/SKILL.md`](device-orientation/SKILL.md) - concise agent
  skill instructions for implementing and reviewing orientation behavior.
- [`landscape-portrait-template.md`](landscape-portrait-template.md) - detailed
  examples and framework-specific setup.

## Quick decision guide

| Requirement | Recommended approach |
| --- | --- |
| Styling only | Use `@media (orientation: portrait)` and `@media (orientation: landscape)` |
| `.portrait` or `.landscape` needed on `<html>` or `<body>` before first paint | Add a synchronous blocking inline script to the HTML shell |
| Reactive JavaScript state | Use Svelte 5 `MediaQuery` or Drupal `Drupal.behaviors` |
| Strict CSP with no inline script | Load a `header: true` library or equivalent before rendering, targeting `<html>` |

## Zero-repaint HTML pattern

When a layout depends on a class from the very first render, execute
`matchMedia` before the browser paints the page.

For a class on `<html>`:

```html
<head>
  <script>
    (function () {
      const query = window.matchMedia('(orientation: portrait)');
      const update = (event) => {
        document.documentElement.classList.toggle('portrait', event.matches);
        document.documentElement.classList.toggle('landscape', !event.matches);
      };

      update(query);
      query.addEventListener('change', update);
    })();
  </script>
</head>
```

Use `document.documentElement` for scripts in `<head>` or other execution
contexts that run before `<body>` exists.

If third-party CSS requires the class on `<body>`, place the script as the
first child inside `<body>`:

```html
<body>
  <script>
    (function () {
      const query = window.matchMedia('(orientation: portrait)');
      const update = (event) => {
        document.body.classList.toggle('portrait', event.matches);
        document.body.classList.toggle('landscape', !event.matches);
      };

      update(query);
      query.addEventListener('change', update);
    })();
  </script>
</body>
```

## Svelte 5

SvelteKit projects should use two complementary layers:

1. Put the blocking script in `src/app.html` to set the initial `<body>` or
   `<html>` classes before first paint.
2. Use `svelte/reactivity` for reactive component state.

```typescript
import { MediaQuery } from 'svelte/reactivity';

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

Store this in `src/lib/orientation.svelte.ts`, since the `.svelte.ts` extension
enables rune processing.

For markup that differs by orientation, prefer rendering both branches and
using CSS visibility to avoid SSR hydration mismatches. Alternatively, pass a
mobile-first SSR fallback to `MediaQuery`, or gate heavy client-only rendering
behind `browser` from `$app/environment`.

## Drupal 10

In a Drupal theme:

1. Add a synchronous class in `templates/layout/html.html.twig` immediately
   after `<body{{ attributes }}>`.
2. Register rotation updates through `Drupal.behaviors` and `core/once`.
3. Add the behavior library in the theme `.libraries.yml` and attach it in the
   theme `.info.yml`.

The Drupal behavior should use `window.matchMedia('(orientation: portrait)')`
and toggle both `.portrait` and `.landscape` on the existing theme element.
Using `core/once` prevents duplicate listeners after AJAX or BigPipe updates.

A strict CSP may require the initial script to run from a `header: true`
library instead of inline Twig. In that context, target
`document.documentElement`, because `<body>` is not available while `<head>`
is loading.

## Guardrails

- Never listen to `window.resize` or derive orientation from viewport dimensions.
- Never bind framework state directly to `<body>` or `<html>` during SSR.
- Never use both orientation media queries and body/html orientation classes
  for the exact same styles; choose one convention.
- Never target `document.body` from `<head>` or a CSP-compliant header library.
- Pair SSR-safe reactive state with the blocking HTML-shell script when initial
  paint consistency matters.

## Using the skill

The repository includes an agent skill at
[`device-orientation/SKILL.md`](device-orientation/SKILL.md). Copy or link that
skill into your agent/tooling configuration to give an implementation agent the
decision hierarchy and framework-specific conventions summarized above.
