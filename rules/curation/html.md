# HTML Curation Rules

## Idiomatic Patterns

What senior HTML engineers write:

- **Semantic elements**: Use `<header>`, `<nav>`, `<main>`, `<article>`, `<section>`, `<aside>`, and `<footer>` instead of `<div>` wrappers. Each element communicates meaning to browsers, assistive technologies, and search engines.
- **Correct heading hierarchy**: One `<h1>` per page, headings in order, no skipping levels (e.g., never jump from `<h2>` to `<h4>`). Heading order communicates document structure to screen readers.
- **Form accessibility**: Every `<input>`, `<select>`, and `<textarea>` has a corresponding `<label for="id">`. Use `<fieldset>` and `<legend>` to group related controls (e.g., radio button groups, address blocks).
- **Button vs anchor**: `<button>` for actions (submit, toggle, open modal), `<a href>` for navigation. Never use `<div onclick>` or `<span onclick>` — they lack keyboard focus, Enter/Space activation, and screen-reader role announcements.
- **Image accessibility**: `alt="meaningful description"` for content images, `alt=""` (empty, not omitted) for decorative images. `alt="image"` is never acceptable.
- **Lists for list content**: `<ul>`/`<ol>` with `<li>` children for sets of items (navigation links, search results, card grids). Not a series of `<div>` or `<span>` elements.
- **Tables for tabular data**: Use `<thead>`, `<tbody>`, and `<th scope="col|row">` so screen readers can associate headers with data cells. Never use tables for layout.
- **ARIA only when needed**: Native HTML semantics (`<button>`, `<nav>`, `<input type="checkbox">`) require no ARIA. Add ARIA roles and attributes only when building custom widgets that have no native equivalent.
- **Meta viewport**: `<meta name="viewport" content="width=device-width, initial-scale=1">` in every page's `<head>` to enable responsive layout.
- **`<template>` and `<slot>` for web components**: Use `<template>` for inert markup that JavaScript clones, and `<slot>` for composable web component content, rather than building DOM with concatenated strings.

## Vibe-Code Anti-Patterns

Common patterns LLM-assisted coding produces in HTML:

- **Div soup**: `<div class="header"><div class="nav"><div class="nav-item">` instead of `<header><nav><a>`. Adds markup weight, breaks assistive technology landmark navigation, and degrades SEO signals.
- **Missing or meaningless `alt`**: Omitting `alt` entirely (browsers fall back to the filename) or setting `alt="image"` everywhere. Both destroy the experience for screen-reader users.
- **ARIA overuse**: `role="button"` on `<button>`, `role="link"` on `<a>`, or `aria-label` duplicating visible text adjacent to the element. Redundant ARIA can confuse assistive technologies without adding any benefit.
- **`<br>` for spacing**: Using `<br>` tags between paragraphs or elements to create visual gaps instead of CSS `margin` or `padding`. `<br>` is for line breaks within prose (addresses, poetry), not layout.
- **Inline styles**: `style="color: red; font-size: 14px"` scattered throughout markup couples presentation to structure, prevents theming, and makes maintenance error-prone.
- **`<div onclick>` instead of `<button>`**: Interactive `<div>` and `<span>` elements with click handlers are not keyboard-focusable by default, do not fire on Enter/Space, and have no announced role for screen readers.
- **Empty elements for spacing**: `<div class="spacer"></div>` or `<div class="clearfix">` inserted purely for visual effect. Use CSS flexbox, grid, or margin instead.
- **Deprecated elements**: `<center>`, `<font>`, `<b>` (for importance; use `<strong>`), `<i>` (for stress; use `<em>`), `<marquee>`, `<blink>`. These were removed from the standard for good reason.
- **Missing `lang` attribute**: `<html>` without `lang="en"` (or the appropriate BCP 47 tag) prevents screen readers from selecting the correct voice and pronunciation rules.
- **Non-semantic class names**: `class="red-text"`, `class="big-margin"`, `class="float-left"`. Class names should describe what an element *is* or *does*, not how it looks.

## Simplification Strategies

### 1. Replace div soup with semantic landmarks

Before — LLM-generated structural divs:

```html
<div class="header">
  <div class="logo"><img src="logo.png"></div>
  <div class="nav">
    <div class="nav-item"><a href="/">Home</a></div>
    <div class="nav-item"><a href="/about">About</a></div>
  </div>
</div>
<div class="main">
  <div class="article">
    <div class="article-title"><h1>Welcome</h1></div>
    <div class="article-body"><p>Content here.</p></div>
  </div>
  <div class="sidebar"><p>Related links</p></div>
</div>
<div class="footer"><p>© 2024</p></div>
```

After — semantic HTML with no wrapper divs:

```html
<header>
  <img src="logo.png" alt="Acme Co">
  <nav>
    <ul>
      <li><a href="/">Home</a></li>
      <li><a href="/about">About</a></li>
    </ul>
  </nav>
</header>
<main>
  <article>
    <h1>Welcome</h1>
    <p>Content here.</p>
  </article>
  <aside><p>Related links</p></aside>
</main>
<footer><p>© 2024</p></footer>
```

### 2. Replace interactive div with button

Before — click handler on a non-interactive element:

```html
<div class="btn" onclick="submitForm()" style="cursor:pointer; padding:8px; background:#007bff; color:#fff;">
  Submit
</div>
```

After — native button with external styles:

```html
<button type="submit" class="btn-primary">
  Submit
</button>
```

The native `<button>` is keyboard-focusable, activates on Enter and Space, announces its role to screen readers, and participates in form submission — all without extra JavaScript or ARIA.

### 3. Add proper form labels

Before — unlabeled inputs with placeholder-as-label:

```html
<form>
  <input type="text" placeholder="First name" name="first_name">
  <input type="email" placeholder="Email" name="email">
  <div onclick="submitForm()">Submit</div>
</form>
```

After — explicitly labeled inputs and a semantic button:

```html
<form>
  <div>
    <label for="first-name">First name</label>
    <input type="text" id="first-name" name="first_name" autocomplete="given-name">
  </div>
  <div>
    <label for="email">Email</label>
    <input type="email" id="email" name="email" autocomplete="email">
  </div>
  <button type="submit">Submit</button>
</form>
```

Placeholders disappear on input and have low contrast; explicit labels are always visible and provide a larger click target.

## Dead Code Signals

HTML-specific indicators that markup or resources are no longer used:

- **Unused CSS classes**: Class names on elements that match no rule in any linked or embedded stylesheet and are not referenced by JavaScript. Common after iterative restyling where old class names are not cleaned up.
- **Hidden elements never shown**: `<div hidden>`, `<div style="display:none">`, or `<div class="hidden">` that no JavaScript ever removes or toggles. Often left over from feature flags or A/B tests that have concluded.
- **Unused `<script>` includes**: `<script src="...">` tags loading libraries (jQuery plugins, old analytics, deprecated polyfills) that nothing in the current codebase calls.
- **Orphaned `<link>` stylesheets**: `<link rel="stylesheet" href="old-theme.css">` referencing files whose selectors no longer match any element in the document.
- **Commented-out HTML blocks**: Large swaths of HTML in `<!-- ... -->` comments that have not been removed after a refactor. They inflate page size and confuse future readers about intent.
- **Unused `id` attributes**: `id` values not referenced by any `<label for>`, anchor link (`href="#id"`), JavaScript `getElementById`, or ARIA attribute (`aria-labelledby`, `aria-describedby`). Safe to remove after confirming no external deep links rely on them.
- **Dead `<template>` elements**: `<template id="...">` blocks whose `id` is never queried by JavaScript and that were not cloned into the document. Left behind when a component was refactored to a framework or removed.
