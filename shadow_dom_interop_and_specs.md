# Shadow DOM, Interop, and Web Performance Specifications

This document analyzes Shadow DOM from the perspective of **browser interoperability, web performance monitoring (RUM), and W3C/WHATWG specification design**:
1. How `open`, `closed`, and `User-Agent (UA)` Shadow DOM work in the web platform.
2. Concrete demos showing how **normal DOM events** and the **normal rendering lifecycle** inside Shadow DOM (including `closed` shadow trees) are *already* observable without any explicit opt-in.
3. The **core platform principles** that follow from these mechanics.
4. Which Web Performance specs touch DOM content, where they adhere to or violate those principles, and **exact Chromium implementation mismatches**.
5. **Stretch Goal:** History and design of an optional capability mechanism allowing developers to pass a `ShadowRoot` (or `Node` reference) to `PerformanceObserver`.

---

## 1. Overview: How Shadow DOM Works (`open` vs. `closed` vs. `UA`)

In the WHATWG DOM and CSS specifications, a document with Shadow DOM is represented by two parallel structures:

1. **The "Tree of Trees" (DOM Encapsulation Realm):**
   - The main `Document` is the root of the **light DOM** tree.
   - An `Element` (the **shadow host**) can host a `ShadowRoot` (`DocumentFragment` subclass), which is the root of a separate **shadow tree**.
   - DOM tree traversals (`parentNode`, `children`, `querySelector`, CSS selector matching, and `id` scoping) stay strictly within a single tree scope (`TreeScope` in Blink). A `ShadowRoot`'s `parentNode` is always `null`; it is linked to the outer tree via `shadowRoot.host`.
2. **The Flat Tree / Composed Tree (Layout, Rendering & Input Realm):**
   - For style calculation, layout, painting, hit-testing, scrolling, and accessibility, the browser resolves `<slot>` assignments and flattens the tree of trees into a single **flat tree** ([CSS Scoping §3.1](https://drafts.csswg.org/css-scoping/#flat-tree)).
   - The rendering pipeline (`requestAnimationFrame` -> Style -> Layout -> `ResizeObserver` -> `IntersectionObserver` -> Paint -> Composite) runs once per `Document` across the entire flat tree.

### Comparing the Three Shadow Root Modes

| Property / Behavior | `mode: "open"` (Author) | `mode: "closed"` (Author) | User-Agent (`kUserAgent`, Browser-Internal) |
| :--- | :--- | :--- | :--- |
| **Who creates it?** | Author script (`attachShadow({mode: 'open'})`) or HTML (`<template shadowrootmode="open">`) | Author script (`attachShadow({mode: 'closed'})`) or HTML (`<template shadowrootmode="closed">`) | Browser engine internally (e.g., `<video controls>`, `<input type="range">`, `<details>`, `<select>`) |
| **`host.shadowRoot` getter** | Returns `ShadowRoot` | Returns `null` | Returns `null` |
| **`element.assignedSlot` getter** | Returns `<slot>` | Returns `null` | Returns `null` |
| **`event.composedPath()` (from light DOM)** | Includes inner shadow nodes | Hides inner shadow nodes (stops at `host`) | Hides inner shadow nodes (stops at `host`) |
| **`event.target` (in light DOM)** | **Retargeted to `host`** | **Retargeted to `host`** | **Retargeted to `host`** |
| **`document.activeElement` / `elementFromPoint()`** | **Retargeted to `host`** | **Retargeted to `host`** | **Retargeted to `host`** |
| **Participates in Flat-Tree Layout & Paint?** | **Yes** | **Yes** | **Yes** (as internal rendering of the built-in element) |

> [!IMPORTANT]
> **Where do `open` and `closed` actually differ in specs?**
> In the entire WHATWG DOM & HTML specifications, a shadow root's `mode` (`"open"` vs. `"closed"`) is only checked in **four places**, all dating back to the 2015 Shadow DOM v1 design:
> 1. `host.shadowRoot` (returns the `ShadowRoot` if `"open"`, `null` if `"closed"`).
> 2. `node.assignedSlot` (returns `<slot>` if its root is `"open"`, `null` if `"closed"`—because `slot.getRootNode()` would otherwise leak the closed `ShadowRoot`).
> 3. `slot.assignedNodes({ flatten: true })` (only flattens through child slots in `"open"` roots).
> 4. `event.composedPath()` ([WHATWG DOM `#concept-unclosed-node`](https://dom.spec.whatwg.org/#concept-unclosed-node), Blink [`TreeScopeEventContext::IsUnclosedTreeOf()`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/renderer/core/dom/events/tree_scope_event_context.cc#L39-L64)), which includes inner nodes of `"open"` shadow roots while truncating `"closed"` shadow roots at `host`.
>
> Even for `event.composedPath()`, **default `event.target` still retargets to `host` for both `open` and `closed`**, and since ~2020 browser vendors (`@annevk`, `@rniwa`, `@emilio`) have treated `composedPath()`'s auto-piercing of `open` roots as a historical exception not to be repeated—requiring newer APIs (`getComposedRanges`, `caretPositionFromPoint`, `getHTML`) to treat `open` and `closed` identically unless the caller explicitly passes `shadowRoots: [...]`.

---

## 2. Demos: Observing Shadow DOM Side Effects *Without* Explicit Opt-In

A common misconception in spec discussions is that emitting performance entries for content inside a `closed` (or `open`) shadow tree reveals hidden information that the page could not otherwise observe.

In reality, because events propagate up the composed path and rendering operates on the flat tree, **light-DOM scripts can already observe user interactions, network loads, layout shifts, box geometry, and rendering timings caused by `closed` Shadow DOM without any opt-in.**

### Demo 1: Normal DOM Events (Bubbling, `composed: true`, and Retargeting)

Standard UI events (`pointerdown`, `click`, `keydown`, `input`, `wheel`, `focusin`) have `bubbles: true` and `composed: true`. When dispatched on an inner element inside a **closed** shadow root, the event crosses the shadow boundary into the light DOM, while WHATWG DOM's [`retarget(A, B)`](https://dom.spec.whatwg.org/#retarget) algorithm rewrites `event.target` to the shadow host:

```html
<!DOCTYPE html>
<my-closed-button id="checkout-btn"></my-closed-button>

<script>
  class MyClosedButton extends HTMLElement {
    constructor() {
      super();
      // Closed shadow root: outer page has zero access to this ShadowRoot
      const closedRoot = this.attachShadow({ mode: 'closed' });
      closedRoot.innerHTML = `
        <button id="secret-inner-btn">
          <span class="label">Buy Now</span>
        </button>
      `;
    }
  }
  customElements.define('my-closed-button', MyClosedButton);

  // Light-DOM listener with NO opt-in and NO reference to closedRoot:
  window.addEventListener('click', (event) => {
    console.log('Event type:', event.type);                 // "click"
    console.log('Retargeted target:', event.target);        // <my-closed-button id="checkout-btn">
    console.log('Target ID:', event.target.id);             // "checkout-btn" (NOT "secret-inner-btn")
    console.log('Composed path:', event.composedPath());    // [my-closed-button#checkout-btn, body, html, document, Window]
    console.log('Timestamp:', event.timeStamp);             // Exact high-res timestamp of interaction
  });
</script>
```

**Exception — Shadow-Confined Events:** If a pointer moves between two sibling elements *inside* the same shadow tree (`mouseover` / `mouseout`), both `event.target` (the entered child) and `event.relatedTarget` (the exited child) retarget to the same `<my-closed-button>` host in the light DOM. Step 10 of [WHATWG DOM Event Path calculation](https://dom.spec.whatwg.org/#concept-event-path) stops propagation at the shadow root whenever `retargetedTarget === retargetedRelatedTarget`, so light-DOM listeners do not fire for internal pointer movements.

---

### Demo 2: Observing Rendering Lifecycle, Geometry, and Layout Shifts of `closed` Shadow Content

Even if a component uses `mode: "closed"`, its visual rendering affects the flat tree. A light-DOM script can observe when the closed shadow content paints, how large it is, what pixels it occupies, and how much it shifts surrounding light-DOM elements:

```html
<!DOCTYPE html>
<my-closed-hero id="hero-component" style="display: block"></my-closed-hero>
<footer id="page-footer">Footer text</footer>

<script>
  // 1. Component author creates a CLOSED shadow root that renders asynchronously
  customElements.define('my-closed-hero', class extends HTMLElement {
    connectedCallback() {
      const closedRoot = this.attachShadow({ mode: 'closed' });
      setTimeout(() => {
        // Inject a 600x400 block inside the closed shadow root after 300ms
        const innerBanner = document.createElement('div');
        innerBanner.style.cssText = 'width: 600px; height: 400px; background: royalblue;';
        innerBanner.textContent = 'Hero Content Inside Closed Shadow DOM';
        closedRoot.appendChild(innerBanner);
      }, 300);
    }
  });

  // 2. Page / RUM script in light DOM (NO access to closedRoot):
  const host = document.getElementById('hero-component');
  const footer = document.getElementById('page-footer');

  // Hit-testing and layout geometry in light DOM reflect the closed shadow content:
  setTimeout(() => {
    const hit = document.elementFromPoint(100, 100);
    console.log('elementFromPoint(100, 100):', hit); // <my-closed-hero id="hero-component">
    console.log('Host rect:', host.getBoundingClientRect()); // 600x400
    console.log('Footer shifted down to Y =', footer.getBoundingClientRect().top); // 400+
  }, 400);
</script>
```

---

### Demo 3: Observing Subresource Loads & Frame Rendering Cost Inside `closed` Shadow DOM

When an `<img>`, `<video>`, font, or stylesheet loads inside a `closed` shadow root, or when a closed shadow tree runs expensive style/layout:
1. **`PerformanceResourceTiming`** emits an entry into the global `window.performance.getEntriesByType('resource')` buffer with the full resource `name` (URL) and timing phases.
2. **`window.onload`** is delayed until the `<img>` inside the closed shadow tree finishes loading.
3. **`PerformanceLongAnimationFrameTiming`** (`long-animation-frame`) includes the style and layout duration of the closed shadow tree in `styleAndLayoutStart` / `duration`.

```html
<!DOCTYPE html>
<div id="closed-image-host"></div>

<script>
  // Observe global resource timing and long animation frames from light DOM
  new PerformanceObserver((list) => {
    for (const entry of list.getEntries()) {
      // Fires for images/stylesheets inside closed shadow roots!
      console.log('[ResourceTiming from closed shadow]:', entry.initiatorType, entry.name);
    }
  }).observe({ type: 'resource', buffered: true });

  const host = document.getElementById('closed-image-host');
  const closedRoot = host.attachShadow({ mode: 'closed' });
  const img = document.createElement('img');
  img.src = 'https://www.w3.org/Icons/w3c_home.png';
  closedRoot.appendChild(img);
</script>
```

---

### Demo 4: Observing *Paint Timing* Specifically From the Light DOM

Beyond layout geometry, resource loads, and script durations, can a light-DOM page observe **when pixels inside a `closed` shadow root actually paint to the screen** (`paintTime` / `presentationTime`) without LCP or Container Timing?

Yes—via two mechanisms already in the platform, plus one gap where Paint Timing provides unique signal:

#### Strategy 1: Global `first-paint` and `first-contentful-paint` (`PerformancePaintTiming`)
If the closed shadow root is the first contentful thing to render on the page, global `first-paint` and `first-contentful-paint` fire and directly give the light DOM the exact compositor `startTime` (`renderTime`) and `paintTime` of that closed shadow paint:

```html
<!DOCTYPE html>
<!-- Light DOM has zero text or images; only an empty custom element host -->
<my-closed-app id="app"></my-closed-app>

<script>
  // 1. Light-DOM observer (NO access to closed shadow root)
  new PerformanceObserver((list) => {
    for (const entry of list.getEntries()) {
      // Fires at ~300ms when the closed shadow root paints its inner text!
      console.log(`[PaintTiming] ${entry.name}: startTime=${entry.startTime}, paintTime=${entry.paintTime}`);
    }
  }).observe({ type: 'paint', buffered: true });

  // 2. Closed shadow root paints content asynchronously at 300ms
  customElements.define('my-closed-app', class extends HTMLElement {
    connectedCallback() {
      const closedRoot = this.attachShadow({ mode: 'closed' });
      setTimeout(() => {
        const h1 = document.createElement('h1');
        h1.textContent = 'Painted inside closed Shadow DOM';
        closedRoot.appendChild(h1);
      }, 300);
    }
  });
</script>
```

#### Strategy 2: `IntersectionObserver` v2 (`trackVisibility: true`) Painted Pixel Occlusion
Even if `first-contentful-paint` has already occurred earlier on the page **and** the closed shadow host has fixed dimensions (so no layout boxes resize or shift), `IntersectionObserver` v2 ([`intersection_observer.cc`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/renderer/core/intersection_observer/intersection_observer.cc#L340-L357)) tests actual **painted pixel occlusion** across the flat tree.

If a light-DOM target sits underneath a transparent `closed` shadow host, the moment the closed shadow root paints opaque pixels over it (e.g., changing `opacity: 0 -> 1` or painting a decoded image), `entry.isVisible` flips from `true` to `false`:

```html
<!DOCTYPE html>
<div style="position: relative; width: 400px; height: 300px;">
  <!-- Light-DOM target positioned underneath the closed shadow host -->
  <div id="light-dom-target" style="width: 100px; height: 100px;">Probe</div>

  <!-- Fixed-size closed shadow host sitting on top of #light-dom-target -->
  <div id="closed-overlay-host"
       style="position: absolute; inset: 0; width: 400px; height: 300px;"></div>
</div>

<script>
  // 1. Attach closed shadow root whose inner content starts unpainted (opacity: 0)
  const host = document.getElementById('closed-overlay-host');
  const closedRoot = host.attachShadow({ mode: 'closed' });
  const innerCard = document.createElement('div');
  innerCard.style.cssText = 'width: 100%; height: 100%; background: black; opacity: 0;';
  closedRoot.appendChild(innerCard);

  // 2. Light-DOM IntersectionObserver v2 tracks actual painted visibility of #light-dom-target
  const io = new IntersectionObserver((entries) => {
    for (const entry of entries) {
      // Initially: isVisible === true (innerCard in closed shadow root is opacity: 0)
      // After 500ms: isVisible === false (innerCard in closed shadow root painted opaque pixels!)
      console.log(`[IOv2 Paint Occlusion] time=${entry.time}, isVisible=${entry.isVisible}`);
    }
  }, { threshold: 1.0, trackVisibility: true, delay: 100 });

  io.observe(document.getElementById('light-dom-target'));

  // 3. After 500ms, closed shadow root paints opaque pixels without changing any box size
  setTimeout(() => {
    innerCard.style.opacity = '1';
  }, 500);
</script>
```

#### Where Paint Timing Adds Unique Signal (and Why Host Retargeting Is Needed)
Without positioning an occlusion target behind the component, if a closed shadow host has **fixed dimensions** (`width: 400px; height: 300px`), FCP has already fired earlier on the page, and an `<img width="400" height="300">` inside the closed shadow root finishes downloading (`ResourceTiming.responseEnd`) and then spends several frames in **asynchronous image decoding** before its pixels are committed to the screen:
- No layout boxes change size or shift, and `ResourceTiming` only exposes network download completion (`responseEnd`), not when async image decode finished and presented pixels to the user.
- That exact gap—measuring when decoded images or visual updates inside a component are actually presented to the screen without requiring hacky occlusion probes—is why Paint Timing, LCP, and Container Timing exist, and attributing that paint timestamp to `shadowHost` brings fixed-size shadow components to parity with the rest of the platform.

---

## 3. Core Platform Principles for Shadow DOM in Web Performance

From the platform mechanics demonstrated above, three foundational principles emerge for Interop and Web Performance specifications:

### Principle 1: Flat-Tree Completeness (Measure All Author Content)
Page-level performance APIs (FCP, LCP, CLS, INP / Event Timing, LoAF) measure the user's visual and interaction experience of the **rendered page (the flat tree)**. Because the visual size, screen coordinates, frame timing, and network loads of Shadow DOM content are *already* observable in the light DOM, **excluding author Shadow DOM from performance metrics does not provide meaningful privacy/encapsulation—it only blinds RUM tools and penalizes Web Component adoption** (as seen in the `connox.be` e-commerce regression in [`w3c/largest-contentful-paint#152`](https://github.com/w3c/largest-contentful-paint/issues/152#issuecomment-6018884522)).

### Principle 2: Encapsulation via Retargeting (`retarget(node, document)`), Not `null`
Shadow DOM encapsulates **inner DOM subtree structure and node identity**, treating the component boundary as an opaque black box to outer scopes.
- When an event, paint, or layout shift occurs on a node `A` inside an author shadow tree and is reported to a `Document`-scoped observer, attribution must not return `null` (which discards knowledge of *which component* caused the work) and must not expose `A` (which breaks encapsulation).
- Instead, attribution should apply the standard WHATWG DOM **[`retarget(A, document)`](https://dom.spec.whatwg.org/#retarget)** algorithm (implemented in Blink as [`TreeScope::Retarget()`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/renderer/core/dom/tree_scope.cc#L654-L687)), resolving attribution to the **outermost light-DOM shadow host**:
  - **Node reference (`element`, `target`, `node`, `lastPaintedElement`):** Outermost light-DOM `shadowHost`.
  - **String identifiers (`id`, `targetSelector`, `invoker`):** Taken from the retargeted `shadowHost` (`shadowHost.id`), never from the inner shadow node.
  - **Resource URL on node-attributed entries (`LCP.url`):** When an inner shadow node is retargeted to its shadow host, internal attributes of the inner node (such as an internal `<img>`'s `url`) should not be copied onto the host's attribution (use `""` unless the host itself owns the URL).
  - **Visual geometry (`size`, `intersectionRect`, `previousRect`, `currentRect`):** Visual rectangles and painted pixel sizes exist in the flat-tree layout realm and are reported accurately.

### Principle 3: `open` == `closed` by Default; `UA` Shadow Roots Are Built-In Internals
1. **`open` and `closed` author shadow roots must behave identically by default.** No performance API should automatically pierce `open` shadow roots without an explicit capability opt-in.
2. **User-Agent (`UA`) shadow roots are browser implementation details of native HTML elements, not author components.**
   - Internal UA shadow widgets (e.g., `<video controls>` play/mute buttons, `<input type="range">` slider thumbs, `<input type="file">` browse buttons, `<input type="date">` pickers) should **not** generate independent paint/layout/element timing candidates.
   - Instead, built-in HTML elements participate in timing metrics strictly through **host-level rules** defined in the spec (e.g., `<img src>`, `<video poster>` / video frame replacement, and `<input>` with a non-empty `value` or `placeholder` per [`w3c/paint-timing#73`](https://github.com/w3c/paint-timing/pull/73)).

---

## 4. Historical Context & Specification Audit Against the Principles

### 4.1 Historical Context: How the Specs & Browsers Diverged

1. **2018–2020 — Early Paint Timing, Element Timing, Event Timing & CLS:**
   - **Event Timing** ([`w3c/event-timing#9`](https://github.com/w3c/event-timing/issues/9), [`#60`](https://github.com/w3c/event-timing/issues/60)) explicitly chose to hook into the end of the DOM event dispatch algorithm, exposing the final **retargeted** `event.target` (the outermost light-DOM shadow host) rather than `composedPath()` targets (`@tdresser`, `@npm1`). Tested in WPTs [`event-retarget.html`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/web_tests/external/wpt/event-timing/event-retarget.html), [`first-input-shadow-dom.html`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/web_tests/external/wpt/event-timing/first-input-shadow-dom.html), and [`shadow-dom-null-target.html`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/web_tests/external/wpt/event-timing/shadow-dom-null-target.html).
   - **Layout Instability (CLS)** ([`WICG/layout-instability#11`](https://github.com/WICG/layout-instability/issues/11), [PR `#32`](https://github.com/WICG/layout-instability/pull/32), [PR `#36`](https://github.com/WICG/layout-instability/pull/36)) defined [`unstable node set`](https://wicg.github.io/layout-instability/#sec-unstable-nodes) over `shadow-including descendants` so shadow shifts count toward CLS. In `#11`, `@skobes-chromium` and `@npm1` discussed retargeting `LayoutShiftAttribution.node` to the top-level shadow host, but PR `#36` shipped the conservative fallback of returning `node: null` (while still exposing `previousRect` and `currentRect`).
   - **Element Timing** ([`w3c/element-timing#3`](https://github.com/w3c/element-timing/issues/3), opened May 2019; [`#61`](https://github.com/w3c/element-timing/issues/61)) excluded shadow trees entirely until an opt-in model could be agreed upon, backed by WPTs [`observe-shadow-image.html`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/web_tests/external/wpt/element-timing/observe-shadow-image.html) and [`observe-shadow-text.html`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/web_tests/external/wpt/element-timing/observe-shadow-text.html).
   - **LCP in Chromium & Gecko:** Measured shadow-tree text and images as LCP candidates, using `exposed for paint timing` only on the [`LargestContentfulPaint.element` getter](https://w3c.github.io/largest-contentful-paint/#dom-largestcontentfulpaint-element) to return `null` for shadow nodes.
2. **2021–2022 — Form Controls & The Paint Timing Refactoring Accident:**
   - In [`w3c/paint-timing#52`](https://github.com/w3c/paint-timing/issues/52) and [PR `#73`](https://github.com/w3c/paint-timing/pull/73), the WG addressed `<input type="text" value="...">` (whose text lives in a UA shadow root) by adding explicit host-level contentfulness rules rather than piercing UA shadow trees.
   - In [PR `w3c/paint-timing#100`](https://github.com/w3c/paint-timing/pull/100), common paint primitives ([`mark paint timing`](https://w3c.github.io/paint-timing/#mark-paint-timing), [`process an image that finished loading`](https://w3c.github.io/paint-timing/#process-image-that-finished-loading), [`exposed for paint timing`](https://w3c.github.io/paint-timing/#exposed-for-paint-timing)) were refactored out of LCP and Element Timing into Paint Timing. **Unintentionally**, Element Timing's shadow-excluding checks (`doc's descendants` and `If root is not a Document, return`) were baked into the shared base algorithm, making the normative spec text exclude Shadow DOM from LCP—contradicting the [`LargestContentfulPaint.element` getter note](https://w3c.github.io/largest-contentful-paint/#dom-largestcontentfulpaint-element).
3. **2025–2026 — Container Timing & The LCP `#152` Reckoning:**
   - While designing Shadow DOM support for Container Timing ([`w3c/container-timing#14`](https://github.com/w3c/container-timing/issues/14), [`#20`](https://github.com/w3c/container-timing/issues/20), [`#22`](https://github.com/w3c/container-timing/issues/22)), `@jasonwilliams` filed [`w3c/largest-contentful-paint#152`](https://github.com/w3c/largest-contentful-paint/issues/152) after finding that inline `<span>` in Shadow DOM attributed LCP `element` to `<body>` or `div#host` while block `<p>` emitted `element: null`.
   - `@mmocny` ([comment #1](https://github.com/w3c/largest-contentful-paint/issues/152#issuecomment-3508891772), [#10](https://github.com/w3c/largest-contentful-paint/issues/152#issuecomment-4215947060)) traced this to Paint Timing's [text aggregation](https://w3c.github.io/paint-timing/#sec-modifications-dom) (which walks to the CSS containing block across inline shadow hosts) and identified the spec contradiction introduced by `paint-timing#100`.
   - `@smfr` ([comment #2](https://github.com/w3c/largest-contentful-paint/issues/152#issuecomment-4194246880)) noted that WebKit landed ([`310575@main`](https://commits.webkit.org/310575@main)) and then reverted ([`310655@main`](https://commits.webkit.org/310655@main)) Shadow DOM LCP support because the current spec text excluded shadow trees.
   - Debate between strict DOM encapsulation (`@annevk` [comment #4](https://github.com/w3c/largest-contentful-paint/issues/152#issuecomment-4197726138); `@rniwa` [comments #6](https://github.com/w3c/largest-contentful-paint/issues/152#issuecomment-4201020064), [#8](https://github.com/w3c/largest-contentful-paint/issues/152#issuecomment-4210172474), [#14](https://github.com/w3c/largest-contentful-paint/issues/152#issuecomment-4217383641)) and flat-tree rendering observability (`@noamr` [comments #9](https://github.com/w3c/largest-contentful-paint/issues/152#issuecomment-4215779626), [#12](https://github.com/w3c/largest-contentful-paint/issues/152#issuecomment-4216217643); `@mmocny` [comments #5](https://github.com/w3c/largest-contentful-paint/issues/152#issuecomment-4200972800), [#7](https://github.com/w3c/largest-contentful-paint/issues/152#issuecomment-4201681005); `@fbuecklers` [comment #16](https://github.com/w3c/largest-contentful-paint/issues/152#issuecomment-6018884522) showing a production `connox.be` e-commerce LCP blind spot) led to the **Apr 9, 2026 WebPerf WG meeting** ([comment #13](https://github.com/w3c/largest-contentful-paint/issues/152#issuecomment-4217337089)) and **May 19, 2026 Blink/Gecko F2F** with `@Bas-moz` and `@emilio` ([comment #15](https://github.com/w3c/largest-contentful-paint/issues/152#issuecomment-4490106861)), converging on:
     1. Identical default behavior for `open` and `closed` author shadow roots.
     2. Exclusion of browser-internal (`UA`) shadow roots.
     3. **Retargeting attribution to the Shadow Host** by default, with `PerformanceObserver` capability opt-in for deeper inspection.

---

### 4.2 Specification & Chromium Overview Table

| Specification & Key Issues | Adheres to Principles Today? | What the Spec Currently Says | Chromium Implementation Status & Mismatches |
| :--- | :--- | :--- | :--- |
| **Paint Timing** (FP / FCP)<br>[Spec](https://w3c.github.io/paint-timing/) · [`#52`](https://github.com/w3c/paint-timing/issues/52) · [PR `#73`](https://github.com/w3c/paint-timing/pull/73) · [PR `#100`](https://github.com/w3c/paint-timing/pull/100) | **No** (Spec accidentally excludes shadow trees; Blink includes UA shadow trees) | [`mark paint timing`](https://w3c.github.io/paint-timing/#mark-paint-timing) iterates `doc's descendants` (light DOM only); [`process image`](https://w3c.github.io/paint-timing/#process-image-that-finished-loading) returns if `root != Document`. [Text aggregation](https://w3c.github.io/paint-timing/#sec-modifications-dom) walks to CSS containing block across inline shadow hosts. | **Mismatches:**<br>1. Blink includes all shadow trees (good for author shadow DOM, **bug for UA shadow DOM** like `<video controls>`).<br>2. Inline shadow host text aggregates past the host up to a light-DOM block ancestor (`<body>`). |
| **Largest Contentful Paint** (LCP)<br>[Spec](https://w3c.github.io/largest-contentful-paint/) · [`#152`](https://github.com/w3c/largest-contentful-paint/issues/152) | **No** (Spec contradicts itself; Safari excludes; Blink nulls `element` & leaks `url`) | [Candidate loop](https://w3c.github.io/largest-contentful-paint/#sec-report-largest-contentful-paint) checks `exposed for paint timing` (excludes shadow trees), contradicting [`.element` getter note](https://w3c.github.io/largest-contentful-paint/#dom-largestcontentfulpaint-element) that says shadow elements return `null`. | **Mismatches:**<br>1. Nulls `element` and `id` instead of retargeting to shadow host.<br>2. **Leaks inner `url`** of images inside closed shadow roots.<br>3. Counts UA shadow roots (`<input type="file">`, `<video controls>`). |
| **Event Timing** (INP / FID)<br>[Spec](https://w3c.github.io/event-timing/) · [`#9`](https://github.com/w3c/event-timing/issues/9) · [`#60`](https://github.com/w3c/event-timing/issues/60) · [`#126`](https://github.com/w3c/event-timing/issues/126) | **Mostly Yes** (Target retargets; gaps in `targetSelector` & UA shadow events) | [`Finalize event timing`](https://w3c.github.io/event-timing/#sec-fin-event-timing) records final `event.target` at end of dispatch (shadow host for composed events; `null` via [`.target` getter](https://w3c.github.io/event-timing/#dom-performanceeventtiming-target) if confined to shadow tree). | **Mismatches:**<br>1. **Bug in `EventPathToSelector()`**: builds `targetSelector` from index 0 (inner shadow node) up to `ShadowRoot`, leaking shadow internals!<br>2. Spec's `finalize event timing` computes `targetSelector` before checking if `target` is in a shadow root. |
| **Layout Instability** (CLS)<br>[Spec](https://wicg.github.io/layout-instability/) · [`#11`](https://github.com/WICG/layout-instability/issues/11) · [PR `#32`](https://github.com/WICG/layout-instability/pull/32) · [PR `#36`](https://github.com/WICG/layout-instability/pull/36) | **Partial** (Includes flat-tree shifts & rects, but returns `node: null`) | [`unstable node set`](https://wicg.github.io/layout-instability/#sec-unstable-nodes) includes `shadow-including descendants`. [`LayoutShiftAttribution.node`](https://wicg.github.io/layout-instability/#sec-layout-shift-attribution) returns `null` via `get an element`. | **Mismatches:**<br>1. `sources[i].node` returns `null` instead of retargeting to the shadow host.<br>2. Only excludes `<input type="range">` thumb (`IsSliderThumbElement()`); other UA shadow shifts are still tracked. |
| **Long Animation Frames** (LoAF)<br>[Spec](https://w3c.github.io/long-animation-frames/) | **Partial** (Frame style/layout includes flat tree, but `invoker` leaks shadow IDs/URLs) | [`record timing info for event listener`](https://w3c.github.io/long-animation-frames/#report-event-listener) reads `event.currentTarget`'s `nodeName`, `id`, and `src` without checking if `currentTarget` is in a shadow tree. | **Mismatch:**<br>Blink's `EventTargetToString(currentTarget)` serializes `TAG#inner-id.onclick` and `TAG[src="..."].onclick` from inside closed and UA shadow roots. |
| **Element Timing**<br>[Spec](https://w3c.github.io/element-timing/) · [`#3`](https://github.com/w3c/element-timing/issues/3) · [`#61`](https://github.com/w3c/element-timing/issues/61) | **By Design Excluded** (Awaiting opt-in / retargeting model) | Driven by [`mark paint timing`](https://w3c.github.io/element-timing/#sec-report-element-timing) (excludes shadow trees). | Drops all nodes where `element->IsInShadowTree()` is true. |
| **Container Timing**<br>[Spec](https://w3c.github.io/container-timing/) · [`#14`](https://github.com/w3c/container-timing/issues/14) · [`#20`](https://github.com/w3c/container-timing/issues/20) · [`#22`](https://github.com/w3c/container-timing/issues/22) | **By Design Excluded in v1** (Slotted light-DOM works) | Explicitly returns `false` in [`contributes to container timing`](https://w3c.github.io/container-timing/#contributes-to-container-timing) if `element` is in a shadow tree. | Skips shadow nodes in pre-paint (`kPropagateAncestorRoot`), while allowing slotted light-DOM children to contribute to light-DOM containers. |
| **Soft Navigations / ICP** | **Partial** (Has UA shadow retargeting for DOM modifications, but inherits LCP calculator) | Tracks DOM modifications and attributes subsequent paints/LCP to interactions. | [`SoftNavigationPaintAttributionTracker`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/renderer/core/timing/soft_navigation_paint_attribution_tracker.cc#L36-L44) explicitly retargets UA shadow roots (`root->IsUserAgent()`) to `&root->host()`, but leaves author shadow roots un-retargeted in `LargestContentfulPaintCalculator`. |

---

### Detailed Spec Proposal, Chromium Action Plan & Stretch Goal Design

---

## 5. Fundamental Spec Proposal: Replacing `exposed for paint timing` with `retarget for performance timing`

Currently, Paint Timing defines a boolean check [`exposed for paint timing(element, document)`](https://w3c.github.io/paint-timing/#exposed-for-paint-timing) that returns `false` if `element`'s root is not `document`. Every downstream spec (LCP, Event Timing, Element Timing, Container Timing) calls this boolean check and either drops the candidate or returns `null`.

Instead, we can replace `exposed for paint timing` with a single, shared node-returning primitive built on WHATWG DOM's [`retarget(A, B)`](https://dom.spec.whatwg.org/#retarget) (in Blink, [`TreeScope::Retarget()`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/renderer/core/dom/tree_scope.cc#L654-L687) and [`DomSelection::Rescope()`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/renderer/core/editing/dom_selection.cc#L649-L670)):

### Proposed Spec Algorithm: `retarget for performance timing`

> To **retarget for performance timing** given a `Node` *node*, a `Document` *document*, and an optional set of `Node`s *allowedScopes* (default empty):
> 1. If *node* is null, or *node* is not [connected](https://dom.spec.whatwg.org/#connected), or *node*'s [shadow-including root](https://dom.spec.whatwg.org/#concept-shadow-including-root) is not *document*, return `null`.
> 2. If *node* is a pseudo-element, set *node* to *node*'s ultimate originating element.
> 3. If *node*'s [root](https://dom.spec.whatwg.org/#concept-tree-root) is a **user-agent shadow root**, return `null` (unless *node*'s host participates via an explicit host-level rule).
> 4. While *node*'s [root](https://dom.spec.whatwg.org/#concept-tree-root) is a `ShadowRoot` *root*:
>    a. If *root* is the [root](https://dom.spec.whatwg.org/#concept-tree-root) of any node in *allowedScopes*, break.
>    b. Set *node* to *root*'s [host](https://dom.spec.whatwg.org/#concept-documentfragment-host).
> 5. Return *node*.

**Why this single primitive solves all cases:**
- **Default document-level observers (`allowedScopes` = empty):** Loops until *node* is in `document`, returning the **outermost light-DOM shadow host** ( matching [`TreeScope::Retarget()`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/renderer/core/dom/tree_scope.cc#L654-L687)).
- **Checking if retargeting crossed a shadow boundary:** If `retargetedNode !== originalNode`, the caller knows it crossed an un-opted-in shadow boundary and must use `retargetedNode.id` and clear internal resource attributes (`LCP.url = ""`).
- **Forward-compatible with `PerformanceObserver` opt-in:** When an observer provides `shadowRoots: [sr1, sr2]` or `node: containerNode`, passing those into *allowedScopes* automatically stops retargeting at the deepest authorized shadow root!

### 5.2 Does Emitting Multiple LCP Candidates for the Same Retargeted `shadowHost` Break Anything?

If a shadow tree paints a small `<h1>` at `t=100ms`, a larger `<p>` at `t=300ms`, and a hero `<img>` at `t=800ms`, retargeting means LCP will emit **three consecutive `LargestContentfulPaint` entries that all point to the same `entry.element === shadowHost`** (with different `startTime`, `size`, and `loadTime`).

In practice, **this scheme works cleanly and matches existing platform behavior** for three reasons:

1. **Chromium & Firefox *already* emit all three entries today—just with `element: null`:**
   - Retargeting does not change the number of LCP entries, their timestamps, or their sizes. It only replaces `element: null` with `element: shadowHost` on those exact same entries.
2. **Light DOM *already* emits multiple LCP entries for the exact same `Element` today:**
   - **Text + CSS `background-image` on the same element:** In Blink, [`TextPaintTimingDetector`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/renderer/core/paint/timing/text_paint_timing_detector.cc#L83-L86) deduplicates by `&aggregator` (`LayoutBoxModelObject`), while [`ImagePaintTimingDetector`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/renderer/core/paint/timing/image_paint_timing_detector.cc#L175-L223) deduplicates by `RecordId = (&image_layout_object, &media_timing)`. A light-DOM `<div id="hero" style="background-image: url(slow.jpg)">Welcome</div>` already emits two LCP entries with `entry.element === div#hero`: first when the text paints (`loadTime: 0`), and a second time when `slow.jpg` loads (`loadTime > 0`)!
   - **Multiple `background-image` layers or `<img>` `src`/`srcset` upgrades:** Whenever `MediaTiming` changes on an `<img>` (e.g., low-res placeholder swapped to high-res image, or `srcset` switching on resize) or a second background image loads on a `<div>`, Blink creates a new `ImageRecord` and emits another LCP entry for the same `Element`.
3. **How RUM libraries (`web-vitals.js`, etc.) consume LCP:**
   - RUM collectors always report the **last** `LargestContentfulPaint` entry emitted before first input or `visibilitychange`. Having intermediate candidates also point to `shadowHost` (or overwrite `map.set(entry.element, entry)`) causes zero issues and makes the final winning entry attributable.
   - **Crucial implementation requirement:** Paint Timing's "once per element" deduplication (`recorded_set_` and `recorded_images_`) **must remain keyed on the inner node's `LayoutObject` / `MediaTiming`**, *not* the retargeted `shadowHost`. If deduplication were keyed on `shadowHost`, an early loading spinner or title inside `<my-hero>` at `t=100ms` would mark `<my-hero>` as "already reported" and suppress the real hero image inside `<my-hero>` at `t=800ms`!

---

## 6. Chromium & W3C Spec Action Plan (Next Steps)

Below is the concrete roadmap for Chromium, separated by whether changes can land **immediately as Chromium bug fixes** or **require W3C spec changes first**.

### Summary Matrix of Next Steps

| # | Work Item | Chromium Files Affected | Needs Spec Change? | Spec Repo / Issue |
| :--- | :--- | :--- | :--- | :--- |
| **1.1** | Fix `EventTiming` `targetSelector` leaking inner shadow tree selectors | [`timing_utils.cc`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/renderer/core/timing/timing_utils.cc#L59-L98) | **No** (Blink bug in experimental feature; minor spec guard for non-composed events) | `w3c/event-timing#126` |
| **1.2** | Stop LCP from leaking internal `url` when image/video is in a shadow tree | [`largest_contentful_paint_calculator.cc`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/renderer/core/paint/timing/largest_contentful_paint_calculator.cc#L218-L235) | **No** (Spec already doesn't allow shadow candidates; fixing leak aligns with privacy/encapsulation) | `w3c/largest-contentful-paint#152` |
| **1.3** | Stop Paint Timing text aggregation at `ShadowRoot` / Shadow Host boundary for inline hosts | [`text_paint_timing_detector.cc`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/renderer/core/paint/timing/text_paint_timing_detector.cc#L49-L96) | **Clarification** in `w3c/paint-timing` text aggregation | `w3c/largest-contentful-paint#152`, `w3c/paint-timing` |
| **2.1** | Retarget `event.currentTarget` in LoAF `invoker` (`EventTargetToString`) so shadow `#id`/`[src]` don't leak | [`animation_frame_timing_monitor.cc`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/renderer/core/frame/animation_frame_timing_monitor.cc#L1153-L1154), [`timing_utils.cc`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/renderer/core/timing/timing_utils.cc#L41-L54) | **Yes** (Small spec fix in `report-event-listener`) | `w3c/long-animation-frames` |
| **2.2** | Exclude User-Agent (`kUserAgent`) shadow roots from FCP, LCP, and CLS (except explicit host rules like `<input value>`, `<video poster>`) | [`text_paint_timing_detector.cc`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/renderer/core/paint/timing/text_paint_timing_detector.cc), [`image_paint_timing_detector.cc`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/renderer/core/paint/timing/image_paint_timing_detector.cc), [`layout_shift_tracker.cc`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/renderer/core/layout/layout_shift_tracker.cc#L222-L225) | **Yes** (Update `mark paint timing` & `unstable node set` to include author shadow trees and exclude UA shadow trees) | `w3c/paint-timing`, `w3c/largest-contentful-paint#152`, `WICG/layout-instability` |
| **3.1** | Retarget `LargestContentfulPaint.element` and `id` to outermost light-DOM shadow host instead of `null` / `""` | [`largest_contentful_paint_calculator.cc`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/renderer/core/paint/timing/largest_contentful_paint_calculator.cc#L225-L265), [`largest_contentful_paint.cc`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/renderer/core/timing/largest_contentful_paint.cc#L46-L49) | **Yes** (Normative change + WPTs) | `w3c/largest-contentful-paint#152`, `w3c/paint-timing` |
| **3.2** | Retarget `LayoutShiftAttribution.node` to outermost light-DOM shadow host instead of `null` | [`layout_shift_attribution.cc`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/renderer/core/timing/layout_shift_attribution.cc#L37-L39) | **Yes** (Normative change + WPTs) | `WICG/layout-instability#11` |
| **3.3** | Include author shadow trees in Container Timing and retarget `lastPaintedElement` to shadow host | [`container_timing_paint_attribution_tracker.cc`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/renderer/core/paint/timing/container_timing_paint_attribution_tracker.cc#L21-L24), [`container_timing.cc`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/renderer/core/paint/timing/container_timing.cc#L51) | **Yes** (Normative change in `contributes to container timing`) | `w3c/container-timing#14`, `#22` |
| **4.1** | **Stretch Goal:** Add `shadowRoots` (for global APIs) and `node` / `roots` (for Container/Element Timing) to `PerformanceObserverInit` | [`performance_observer_init.idl`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/renderer/core/timing/performance_observer_init.idl), [`performance_observer.cc`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/renderer/core/timing/performance_observer.cc), [`performance.cc`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/renderer/core/timing/performance.cc) | **Yes** (New feature across `performance-timeline` + consumer specs) | `w3c/performance-timeline`, `w3c/container-timing#20`, `w3c/element-timing#3` |

---

### Details on Each Tier of Chromium & Spec Work

#### Tier 1: Immediate Chromium Bug Fixes (No Spec Blocker)
1. **Fix `EventPathToSelector()` in [`timing_utils.cc`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/renderer/core/timing/timing_utils.cc#L59-L98):**
   - *Bug:* `EventPath::NodeEventContexts()` starts at index `0` (the raw inner target inside the shadow root) and walks up to `Document`. The loop at `lines 71-86` collects elements starting at index `0` and `break`s at `IsA<ShadowRoot>(node)`, which means it builds a selector of the **shadow tree internals**!
   - *Fix:* Start selector collection at the retargeted light-DOM target (`event.GetEventPath().GetWindowEventContext().Target()`, or skip any `NodeEventContext` where `node->IsInShadowTree()`). If the event was confined inside a shadow tree (`WindowEventContext().Target() == nullptr`), return `g_empty_atom`.
2. **Clear `image_url` for shadow-tree images in [`LargestContentfulPaintCalculator::UpdateWebExposedLargestContentfulImage()`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/renderer/core/paint/timing/largest_contentful_paint_calculator.cc#L218-L235):**
   - *Bug:* When `image_node->IsInShadowTree()` is true, `image_element` and `image_id` are cleared (`lines 225-228`), but `image_url` (`lines 218-223`) is still populated and exposed on `LargestContentfulPaint.url`.
   - *Fix:* Gate `image_url` on `!image_node->IsInShadowTree()` (or in the future, on whether the image's `TreeScope` was opted into by the observer).
3. **Cap Paint Timing text aggregation at Shadow Boundaries in [`TextPaintTimingDetector`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/renderer/core/paint/timing/text_paint_timing_detector.cc#L49-L96):**
   - *Bug:* As discovered in `w3c/largest-contentful-paint#152`, a `<span>` inside an inline `<my-element>` shadow host aggregates its text to the nearest CSS containing block (`<body>` or an outer `<div>`), bypassing the shadow host.
   - *Fix:* When finding the aggregator `LayoutBoxModelObject` for a text node in a shadow tree, if the containing block is in an ancestor `TreeScope` (outside the text node's `ShadowRoot`), attribute/aggregate to the shadow host instead of walking past it into the light DOM.

#### Tier 2: Spec Clarifications & Security/Encapsulation Fixes
1. **Retarget `event.currentTarget` in LoAF (`w3c/long-animation-frames` & Blink):**
   - *Spec change:* In [`record timing info for event listener`](https://w3c.github.io/long-animation-frames/#report-event-listener), replace `let target be event's currentTarget` with `let target be the result of retargeting event's currentTarget against document`.
   - *Chromium change:* In [`EventTargetToString(EventTarget* target)`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/renderer/core/timing/timing_utils.cc#L41-L54), if `Element* element = DynamicTo<Element>(node)` has `element->IsInShadowTree()`, replace `element` with `&element->GetDocument().Retarget(*element)` before formatting `nodeName#id` or `[src="..."]` (and return `g_empty_atom` if in a UA shadow root or disconnected).
2. **Exclude User-Agent (`kUserAgent`) Shadow Roots from FCP / LCP / CLS:**
   - *Spec change:* In `w3c/paint-timing` (`mark paint timing` and `process an image that finished loading`) and `WICG/layout-instability` (`unstable node set`), explicitly include `shadow-including descendants` in **author** shadow trees while excluding nodes whose root is a **user-agent shadow root** (relying on explicit host rules for `<input value="...">`, `<input placeholder="...">`, and `<video poster="...">`).
   - *Chromium change:* Check `node->ContainingShadowRoot()->IsUserAgent()` in `TextPaintTimingDetector`, `ImagePaintTimingDetector`, and `LayoutShiftTracker::NeedsToTrack()`, and update unit test [`TextPaintTimingDetectorTest.CaptureFileUploadController`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/renderer/core/paint/timing/text_paint_timing_detector_test.cc#L481-L491).

#### Tier 3: Normative Spec & Chromium Updates for Default Host Retargeting
1. **LCP Host Retargeting (`w3c/largest-contentful-paint#152`):**
   - *Spec change:* Use `retarget for performance timing(candidateElement, document)` to populate `LargestContentfulPaint.element` and `id`, while keeping `has been reported` candidate deduplication on the inner node/record.
   - *Chromium change:* In [`LargestContentfulPaintCalculator`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/renderer/core/paint/timing/largest_contentful_paint_calculator.cc#L225-L265), replace `image_node->IsInShadowTree() ? nullptr : To<Element>(image_node)` with `&image_node->GetDocument().Retarget(*To<Element>(image_node))`.
2. **CLS Host Retargeting (`WICG/layout-instability#11`):**
   - *Spec & Chromium change:* Update `LayoutShiftAttribution.node` ([`layout_shift_attribution.cc`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/renderer/core/timing/layout_shift_attribution.cc#L37-L39)) to return `Performance::RetargetNode(node_, source_)` instead of `nullptr` when `node_` is in an author shadow tree.
3. **Container Timing Host Retargeting (`w3c/container-timing#14`, `#22`):**
   - *Spec & Chromium change:* In [`ContainerTimingPaintAttributionTracker::UpdateOnPrePaint()`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/renderer/core/paint/timing/container_timing_paint_attribution_tracker.cc#L21-L24), allow author shadow nodes to inherit `context_container_root` so their paints are counted toward an ancestor container's `paintedArea`, while retargeting `lastPaintedElement` to the shadow host.

---

## 7. Stretch Goal: Dual Capability Opt-In for `PerformanceObserver`

### 7.1 History of Shadow DOM Opt-In Across W3C & WHATWG Specifications

1. **2012 — `MutationObserver` ([WHATWG DOM `#interface-mutationobserver`](https://dom.spec.whatwg.org/#interface-mutationobserver)):**
   - Designed alongside early Shadow DOM. Instead of observing the global `window`, `mo.observe(target, options)` requires passing a `Node` reference (`target`).
   - Passing a `ShadowRoot` (or an `Element` inside a shadow root) acts as an object-capability proof: you can only observe mutations in a tree if you hold a reference to a node in that tree.
   - **Non-piercing subtree boundary:** Step 2 of [`queue a mutation record`](https://dom.spec.whatwg.org/#queueing-a-mutation-record) only walks `node.parentNode` ([`Node::GetRegisteredMutationObserversOfType()`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/renderer/core/dom/node.cc#L3220)), so `subtree: true` stops at nested `ShadowRoot` boundaries (`parentNode === null`) unless the caller also calls `mo.observe(nestedShadowRoot, ...)`.
2. **2016–2018 — `IntersectionObserver` & `ResizeObserver`:**
   - **Intersection Observer** ([Spec `#intersection-observer-api`](https://w3c.github.io/IntersectionObserver/#intersection-observer-api), [`w3c/IntersectionObserver#84`](https://github.com/w3c/IntersectionObserver/issues/84), [PR `#218`](https://github.com/w3c/IntersectionObserver/pull/218), [PR `#233`](https://github.com/w3c/IntersectionObserver/pull/233)): `io.observe(element)` requires a direct `Element` reference (open or closed shadow root), while [`compute the intersection`](https://w3c.github.io/IntersectionObserver/#calculate-intersection-rect-algo) walks the CSS [containing block chain](https://drafts.csswg.org/css-display/#containing-block-chain) across the flat tree.
   - **Resize Observer** ([Spec `#api`](https://drafts.csswg.org/resize-observer-1/#api), [`#calculate-depth-for-node-h`](https://drafts.csswg.org/resize-observer-1/#calculate-depth-for-node-h)): `ro.observe(element)` takes an `Element` capability token and orders notifications by flat-tree depth.
3. **2019–2022 — `w3c/element-timing#3` & TPAC 2022 Options:**
   - In [`w3c/element-timing#3`](https://github.com/w3c/element-timing/issues/3) ("How should Element Timing interact with Shadow DOM?"), `@annevk` emphasized that `open` and `closed` shadow roots must behave identically by default, while `@hayatoito` showed why `<slot>`-only tracking fails when components nest custom elements inside shadow trees.
   - At TPAC 2022, `@nolanlawson` (Salesforce) presented 4 options:
     1. **Top-down observer opt-in:** `observer.observe({ entryTypes: ["element"], shadowRoots: [...] })`.
     2. **Bottom-up host delegation:** `attachShadow({ mode: "open", delegatesElementTiming: true })` (preferred by `@rniwa` for component encapsulation, though painful across 20+ nested shadow levels).
     3. **Global unconditional exposure:** Rejected as breaking encapsulation.
     4. **Namespaced exposure on `attachShadow`:** Liked by `@smaug----` (Mozilla) as a "C++ friend" pattern.
     5. **Host retargeting + opt-in (`@yoavweiss`):** Emit Element Timing globally with `entry.element` retargeted to the top-most shadow host unless opted in.
4. **2023–2025 — Platform Convergence on `{ shadowRoots: [...] }`:**
   - WHATWG and CSSWG standardized a unified capability dictionary pattern across three major APIs that treats `open` and `closed` identically and retargets unlisted shadow roots:
     - **`Selection.prototype.getComposedRanges({ shadowRoots: [...] })`** ([Selection API `#dom-selection-getcomposedranges`](https://w3c.github.io/selection-api/#dom-selection-getcomposedranges), [`w3c/selection-api#163`](https://github.com/w3c/selection-api/issues/163), [`whatwg/html#5484`](https://github.com/whatwg/html/issues/5484), Blink [`DomSelection::getComposedRanges()`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/renderer/core/editing/dom_selection.cc#L611-L670)).
     - **`document.caretPositionFromPoint(x, y, { shadowRoots: [...] })`** ([CSSOM View `#dom-document-caretpositionfrompoint`](https://drafts.csswg.org/cssom-view-1/#dom-document-caretpositionfrompoint), Blink [`Document::caretPositionFromPoint()`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/renderer/core/dom/document.cc#L2120)).
     - **`Element.prototype.getHTML({ serializableShadowRoots, shadowRoots: [...] })`** ([WHATWG HTML `#dom-element-gethtml`](https://html.spec.whatwg.org/multipage/dynamic-markup-insertion.html#dom-element-gethtml), [PR `whatwg/html#10139`](https://github.com/whatwg/html/pull/10139)), combining bottom-up `attachShadow({ serializable: true })` with top-down `shadowRoots: [...]`.
5. **2025–2026 — Container Timing & LCP `#152` Convergence:**
   - In [`w3c/container-timing#20`](https://github.com/w3c/container-timing/issues/20) ("How should Container Timing attributes work? Could multiple observers have different views?") and [`w3c/largest-contentful-paint#152` comment #15](https://github.com/w3c/largest-contentful-paint/issues/152#issuecomment-4490106861) & [#17](https://github.com/w3c/largest-contentful-paint/issues/152#issuecomment-6019646476), `@mmocny`, `@jasonwilliams`, `@emilio`, and `@Bas-moz` aligned on default host retargeting paired with observer-level capability options in `PerformanceObserver.prototype.observe()`.

---

### 7.2 Why We Need Both `shadowRoots` and `root` (`node`) in `PerformanceObserverInit`

Global page-level APIs and scoped container/element APIs have fundamentally different observation scopes—so **we need both capability options** in `PerformanceObserverInit` ([`w3c/performance-timeline#observe-method`](https://w3c.github.io/performance-timeline/#observe-method)):

1. **`shadowRoots: sequence<ShadowRoot>` — For Global / Page-Level APIs (LCP, Event Timing, CLS, LoAF):**
   - The observer still observes the **entire page (`Document`)**, but lists all `ShadowRoot` references (open or closed) that it has capability to inspect.
   - Any event, paint, or shift inside a listed `ShadowRoot` exposes the inner attribution (or retargets to a nested child host if a deeper shadow root is *not* in `shadowRoots`).
2. **`node: Node` (or `root: Node` / `roots: sequence<Node>`) — For Subtree / Scoped APIs (Container Timing, Element Timing):**
   - The observer passes the specific **container root `Element`** or **`ShadowRoot`** it wants to measure (which may itself be in the light DOM or inside an open/closed shadow tree!).
   - Holding a reference to `node` automatically grants capability for `node.getRootNode()` (`node->GetTreeScope()`) and scopes the observer to only emit entries for that container/subtree.
   - Furthermore, if that container `node` *also* contains nested shadow roots that the caller has access to, `node` and `shadowRoots` compose naturally!

### Proposed WebIDL Extension (`w3c/performance-timeline`)

```webidl
dictionary PerformanceObserverInit {
  sequence<DOMString> entryTypes;
  DOMString type;
  boolean buffered;

  // 1. For global APIs (LCP, Event Timing, CLS, LoAF) or piercing nested roots:
  //    List of ShadowRoots the observer is authorized to see inside.
  sequence<ShadowRoot> shadowRoots = [];

  // 2. For scoped APIs (Container Timing, Element Timing):
  //    Specific container Element(s) or ShadowRoot(s) to observe.
  Node root; // or sequence<Node> roots;
};
```

### Usage Examples Across Both Patterns

```js
// Pattern 1: Page-level RUM observer with access to known ShadowRoots (LCP / INP / CLS)
const pageObserver = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    // If LCP candidate is inside appShadowRoot or heroShadowRoot,
    // entry.element is the inner Element (and entry.id / entry.url are populated).
    // If LCP candidate is inside a 3rd-party widget's unlisted shadow root,
    // entry.element is retargeted to that 3rd-party widget's host!
    console.log(entry.entryType, entry.element, entry.id, entry.url);
  }
});
pageObserver.observe({
  type: 'largest-contentful-paint',
  buffered: true,
  shadowRoots: [appShadowRoot, heroShadowRoot]
});

// Pattern 2: Component-scoped Container Timing observer passing its container Node
// Works inside a closed ShadowRoot WITHOUT needing HTML attributes!
class ProductGrid extends HTMLElement {
  #shadow = this.attachShadow({ mode: 'closed' });

  connectedCallback() {
    this.#shadow.innerHTML = `<section class="grid">...</section>`;
    const gridSection = this.#shadow.querySelector('.grid');

    const containerObserver = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        // entry.rootElement === gridSection
        // entry.lastPaintedElement === inner element inside this.#shadow
        // (or retargeted to a child custom element host if nested!)
        console.log('Grid container painted:', entry.size, entry.lastPaintedElement);
      }
    });

    containerObserver.observe({
      type: 'container',
      root: gridSection // Scopes to gridSection AND authorizes this.#shadow's TreeScope!
    });
  }
}
```

### How Both Options Unify Under the Hood in Blink

In Blink, both `shadowRoots` and `root` map directly onto `TreeScope` capability sets:
1. **Authorized `TreeScope` set for an observer (`observer_scopes_`):**
   - Starts with `{ document.GetTreeScope() }`.
   - For each `sr` in `options.shadowRoots()`, add `sr->GetTreeScope()`.
   - If `options.root()` is provided, add `options.root()->GetTreeScope()` (and filter emitted entries to descendants of `options.root()`).
2. **Resolving any `raw_node` for that observer:**
   - Walk `Node* curr = raw_node` upwards via `curr = curr->OwnerShadowHost()` while `curr->IsInShadowTree()` and `!observer_scopes_.Contains(&curr->GetTreeScope())`.
   - The resulting `curr` is the exact retargeted `Element*` for that observer!
   - If `curr != raw_node` (retargeting occurred across an unauthorized boundary), use `curr->GetIdAttribute()` and suppress `raw_node`'s internal `url`.
3. **Zero-overhead gating for Container / Element Timing inside Shadow DOM:**
   - When an observer registers with `root` or `shadowRoots`, Blink marks the corresponding `TreeScope`s (and/or `root` `Element` for Container Timing) as having active scoped observers, enabling pre-paint and paint timing collection inside those specific shadow trees without paying overhead on un-observed shadow trees.

---

## 8. Complete Reference Index: Specifications, Issues, PRs & Tests

### Web Performance Specifications & Issues
- **Largest Contentful Paint (`w3c/largest-contentful-paint`):**
  - Spec: [`LargestContentfulPaint.element` getter](https://w3c.github.io/largest-contentful-paint/#dom-largestcontentfulpaint-element), [`Compute a new LCP candidate`](https://w3c.github.io/largest-contentful-paint/#sec-report-largest-contentful-paint)
  - [`w3c/largest-contentful-paint#152`](https://github.com/w3c/largest-contentful-paint/issues/152) — *"What is the behavior when paints emit from inside the Shadow DOM?"*
  - WebKit commits referenced in `#152`: [`310575@main`](https://commits.webkit.org/310575@main) (initial implementation) and [`310655@main`](https://commits.webkit.org/310655@main) (revert due to Paint Timing spec text)
- **Paint Timing (`w3c/paint-timing`):**
  - Spec: [`Process image that finished loading`](https://w3c.github.io/paint-timing/#process-image-that-finished-loading), [`Text aggregation`](https://w3c.github.io/paint-timing/#sec-modifications-dom), [`Mark paint timing`](https://w3c.github.io/paint-timing/#mark-paint-timing), [`Exposed for paint timing`](https://w3c.github.io/paint-timing/#exposed-for-paint-timing)
  - [`w3c/paint-timing#52`](https://github.com/w3c/paint-timing/issues/52) & [PR `#73`](https://github.com/w3c/paint-timing/pull/73) — Contentfulness of form controls (`<input value>`, placeholders) rendered via UA shadow roots
  - [PR `w3c/paint-timing#100`](https://github.com/w3c/paint-timing/pull/100) — Refactoring of shared paint primitives from LCP & Element Timing into Paint Timing
- **Event Timing (`w3c/event-timing`):**
  - Spec: [`Modifications to DOM event dispatch`](https://w3c.github.io/event-timing/#sec-modifications-DOM), [`Finalize event timing`](https://w3c.github.io/event-timing/#sec-fin-event-timing), [`PerformanceEventTiming.target` getter](https://w3c.github.io/event-timing/#dom-performanceeventtiming-target), [`Target Selectors`](https://w3c.github.io/event-timing/#sec-target-selectors)
  - [`w3c/event-timing#9`](https://github.com/w3c/event-timing/issues/9) — *"Original Event as attribute"* (decision to use retargeted `event.target`)
  - [`w3c/event-timing#60`](https://github.com/w3c/event-timing/issues/60) — *"Add element and/or id fields to PerformanceEventTiming"*
  - [`w3c/event-timing#126`](https://github.com/w3c/event-timing/issues/126) — *"Add `targetSelector` attribute"*
  - WPTs: [`event-retarget.html`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/web_tests/external/wpt/event-timing/event-retarget.html), [`first-input-shadow-dom.html`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/web_tests/external/wpt/event-timing/first-input-shadow-dom.html), [`shadow-dom-null-target.html`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/web_tests/external/wpt/event-timing/shadow-dom-null-target.html)
- **Element Timing (`w3c/element-timing`):**
  - Spec: [`Report Element Timing`](https://w3c.github.io/element-timing/#sec-report-element-timing), [`PerformanceElementTiming.element` getter](https://w3c.github.io/element-timing/#dom-performanceelementtiming-element)
  - [`w3c/element-timing#3`](https://github.com/w3c/element-timing/issues/3) — *"How should Element Timing interact with Shadow DOM?"* (2019–present, TPAC 2022 options)
  - [`w3c/element-timing#61`](https://github.com/w3c/element-timing/issues/61) — *"ElementTiming in CustomElements"*
  - WPTs: [`observe-shadow-image.html`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/web_tests/external/wpt/element-timing/observe-shadow-image.html), [`observe-shadow-text.html`](file:///usr/local/btrfs_mount/InvestigateShadowDOM/src/third_party/blink/web_tests/external/wpt/element-timing/observe-shadow-text.html)
- **Container Timing (`w3c/container-timing`):**
  - Spec: [`Contributes to Container Timing`](https://w3c.github.io/container-timing/#contributes-to-container-timing), [`Getting the container root Element`](https://w3c.github.io/container-timing/#getting-container-root-element)
  - [`w3c/container-timing#14`](https://github.com/w3c/container-timing/issues/14) — *"Shadow DOM Handling"*
  - [`w3c/container-timing#20`](https://github.com/w3c/container-timing/issues/20) — *"How should Container Timing attributes work? Could multiple observers have different views?"*
  - [`w3c/container-timing#22`](https://github.com/w3c/container-timing/issues/22) — *"Convert 'ignore' into a nesting mode, and add variants (`transparent` / `private` / `shadowed`)"*
- **Layout Instability (`WICG/layout-instability`):**
  - Spec: [`Unstable nodes`](https://wicg.github.io/layout-instability/#sec-unstable-nodes), [`LayoutShiftAttribution` interface](https://wicg.github.io/layout-instability/#sec-layout-shift-attribution)
  - [`WICG/layout-instability#11`](https://github.com/WICG/layout-instability/issues/11), [PR `#32`](https://github.com/WICG/layout-instability/pull/32), [PR `#36`](https://github.com/WICG/layout-instability/pull/36) — Shadow DOM node attribution and rect preservation
- **Performance Timeline (`w3c/performance-timeline`) & Long Animation Frames (`w3c/long-animation-frames`):**
  - Performance Timeline Spec: [`Performance entry buffer map`](https://w3c.github.io/performance-timeline/#extensions-to-the-windoworworkerglobalscope), [`PerformanceObserver.observe()`](https://w3c.github.io/performance-timeline/#observe-method), [`Queue a PerformanceEntry`](https://w3c.github.io/performance-timeline/#queue-a-performanceentry)
  - LoAF Spec: [`LoAF processing model`](https://w3c.github.io/long-animation-frames/#loaf-processing-model), [`Record timing info for event listener`](https://w3c.github.io/long-animation-frames/#report-event-listener), [`PerformanceScriptTiming.invoker`](https://w3c.github.io/long-animation-frames/#sec-PerformanceScriptTiming)

### Related DOM, CSS & HTML Platform Precedents
- **WHATWG DOM:** [`Retarget` algorithm (`#retarget`)](https://dom.spec.whatwg.org/#retarget), [`Event path` (`#concept-event-path`)](https://dom.spec.whatwg.org/#concept-event-path), [`Event.composedPath()` (`#dom-event-composedpath`)](https://dom.spec.whatwg.org/#dom-event-composedpath), [`MutationObserver` (`#interface-mutationobserver`)](https://dom.spec.whatwg.org/#interface-mutationobserver), [`Queueing a mutation record` (`#queueing-a-mutation-record`)](https://dom.spec.whatwg.org/#queueing-a-mutation-record)
- **CSS Scoping Module Level 1:** [`Flat tree` (`#flat-tree`)](https://drafts.csswg.org/css-scoping/#flat-tree)
- **Intersection Observer & Resize Observer:** [`IntersectionObserver` spec](https://w3c.github.io/IntersectionObserver/#calculate-intersection-rect-algo), [`w3c/IntersectionObserver#84`](https://github.com/w3c/IntersectionObserver/issues/84), [PR `#218`](https://github.com/w3c/IntersectionObserver/pull/218), [PR `#233`](https://github.com/w3c/IntersectionObserver/pull/233); [`ResizeObserver` flattened depth](https://drafts.csswg.org/resize-observer-1/#calculate-depth-for-node-h)
- **Selection API, CSSOM View & HTML Serialization (`{ shadowRoots: [...] }` pattern):**
  - [`Selection.prototype.getComposedRanges({ shadowRoots })`](https://w3c.github.io/selection-api/#dom-selection-getcomposedranges) ([`w3c/selection-api#163`](https://github.com/w3c/selection-api/issues/163), [`whatwg/html#5484`](https://github.com/whatwg/html/issues/5484))
  - [`Document.prototype.caretPositionFromPoint(x, y, { shadowRoots })`](https://drafts.csswg.org/cssom-view-1/#dom-document-caretpositionfrompoint)
  - [`Element.prototype.getHTML({ serializableShadowRoots, shadowRoots })`](https://html.spec.whatwg.org/multipage/dynamic-markup-insertion.html#dom-element-gethtml) ([PR `whatwg/html#10139`](https://github.com/whatwg/html/pull/10139))

---

## 9. Summary of Proposed Spec Changes & WPT Test Suite Plan

### 9.1 Summary of Proposed Normative Spec Changes

1. **`w3c/paint-timing`:**
   - Replace boolean [`exposed for paint timing`](https://w3c.github.io/paint-timing/#exposed-for-paint-timing) with the shared node-returning algorithm **`retarget for performance timing(node, document, allowedScopes)`** built on WHATWG DOM's [`retarget(A, B)`](https://dom.spec.whatwg.org/#retarget).
   - Update [`mark paint timing`](https://w3c.github.io/paint-timing/#mark-paint-timing) and [`process an image that finished loading`](https://w3c.github.io/paint-timing/#process-image-that-finished-loading) to iterate over `shadow-including descendants` in **author** shadow trees while excluding **user-agent (`UA`) shadow trees** (relying on explicit host rules for `<input value>`, `<input placeholder>`, and `<video poster>`).
   - Cap [`text aggregation`](https://w3c.github.io/paint-timing/#sec-modifications-dom) at `ShadowRoot` / Shadow Host boundaries so inline text inside an inline shadow host attributes to the shadow host rather than `<body>`.
2. **`w3c/largest-contentful-paint`:**
   - Populate `LargestContentfulPaint.element` and `id` via `retarget for performance timing(candidateElement, document)` (exposing the outermost light-DOM shadow host by default), while keeping candidate deduplication (`has been reported`) keyed on the inner record.
   - Set `LargestContentfulPaint.url` to `""` whenever `retargetedNode !== candidateElement` so internal shadow image URLs do not leak.
3. **`w3c/event-timing`:**
   - Guard `targetSelector` in [`Finalize event timing` Step 6](https://w3c.github.io/event-timing/#sec-fin-event-timing) with `retarget for performance timing` so shadow-confined events report `""` and composed events build selectors starting from the retargeted light-DOM shadow host.
4. **`WICG/layout-instability`:**
   - Update [`LayoutShiftAttribution.node`](https://wicg.github.io/layout-instability/#sec-layout-shift-attribution) to return `retarget for performance timing(node, document)` (the shadow host) instead of `null`, and exclude all UA shadow roots from [`unstable node set`](https://wicg.github.io/layout-instability/#sec-unstable-nodes).
5. **`w3c/long-animation-frames`:**
   - In [`record timing info for event listener`](https://w3c.github.io/long-animation-frames/#report-event-listener), retarget `event.currentTarget` via `retarget for performance timing(currentTarget, document)` before reading `nodeName`, `id`, and `src` to construct `PerformanceScriptTiming.invoker`.
6. **`w3c/container-timing`:**
   - Allow painted nodes inside author shadow roots to contribute to an ancestor container root's painted area/timestamps while retargeting `lastPaintedElement` to the shadow host.
7. **`w3c/performance-timeline` (Stretch Goal):**
   - Extend `PerformanceObserverInit` with `sequence<ShadowRoot> shadowRoots = []` (for global APIs like LCP, Event Timing, CLS, and LoAF) and `Node root` / `sequence<Node> roots` (for scoped APIs like Container Timing and Element Timing).

---

### 9.2 List of Web Platform Tests (WPTs) Worth Writing

| WPT Suite Directory | Proposed Test File | What It Verifies |
| :--- | :--- | :--- |
| **`paint-timing/`** | `fcp-shadow-dom-open-closed.html` | `first-paint` and `first-contentful-paint` fire identically when the only contentful element on the page is inside an `open` or `closed` shadow root. |
| **`paint-timing/`** | `fcp-ua-shadow-excluded.html` | Native browser UI inside UA shadow roots (e.g., empty `<video controls>`, `<input type="range">`, `<input type="file">` with no author text) does **not** trigger FCP. |
| **`largest-contentful-paint/`** | `shadow-dom-retarget-host.html` | Block `<p>` and `<img>` inside `open` and `closed` shadow roots emit LCP entries with `entry.element === shadowHost`, `entry.id === shadowHost.id`, and `entry.url === ""` (verifying no inner `url` leak). |
| **`largest-contentful-paint/`** | `shadow-dom-inline-host-text-aggregation.html` | Inline `<span>` inside an inline custom element (`display: inline`) attributes `entry.element` to the custom element shadow host, not `<body>` or an outer light-DOM `<div>` (regression test for `lcp#152`). |
| **`largest-contentful-paint/`** | `shadow-dom-multiple-candidates-same-host.html` | Progressively larger paints inside a single shadow root (e.g., `<h1>` at `t1`, larger `<p>` at `t2`, largest `<img>` at `t3`) emit three LCP entries that all report `entry.element === shadowHost` with increasing `size`. |
| **`largest-contentful-paint/`** | `shadow-dom-nested-retarget-outermost-host.html` | Content painted inside a nested shadow tree (`host1 -> sr1 -> host2 -> sr2 -> img`) retargets `entry.element` to the outermost light-DOM host (`host1`). |
| **`largest-contentful-paint/`** | `shadow-dom-ua-excluded.html` | `<video controls>` overlay buttons and `<input type="file">` browse button text do not emit LCP candidates, whereas `<input value=" text ">` and `<video poster="...">` do. |
| **`event-timing/`** | `target-selector-shadow-dom.html` | Clicking an element inside an `open` or `closed` shadow root sets `entry.targetSelector` to the light-DOM shadow host selector (never leaking inner `#id` or tag names); shadow-confined `mouseover` sets `entry.target === null` and `entry.targetSelector === ""`. |
| **`layout-instability/`** | `shadow-dom-attribution-retarget.html` | Shifting an element inside an `open` or `closed` shadow root contributes to `LayoutShift.value`, populates `previousRect` and `currentRect`, and sets `sources[0].node === shadowHost`. |
| **`long-animation-frames/`** | `loaf-event-listener-shadow-dom-invoker.html` | A slow (>50ms) event listener attached directly to an element `<button id="secret">` inside an `open` or `closed` shadow root reports the retargeted `shadowHost` in `script.invoker`, never `#secret`. |
| **`container-timing/`** | `container-timing-shadow-host-retarget.html` | Painting text/images inside an `open` or `closed` shadow root inside a light-DOM `containertiming` element contributes to `size` and sets `lastPaintedElement === shadowHost`. |
| **`performance-timeline/`** *(Stretch Goal)* | `po-observe-shadow-roots-and-root.tentative.html` | Two simultaneous observers—Observer A (default) and Observer B (`shadowRoots: [sr]` or `root: innerContainer`)—receive entries from the same shadow paint where Observer A sees `shadowHost`, Observer B sees `innerElement`, and `performance.getEntriesByType()` only contains `shadowHost`. |

---

### 9.3 Interactive W3C Presentation Deck & Standalone Test Pages

- **Presentation Deck (Slide Mode + Document View):** [presentation/index.html](file:///usr/local/google/home/mmocny/.gemini/chrome-ide-dev/brain/b0702da0-b2d6-4bdc-8679-51c4bfc08eef/presentation/index.html)
- **Standalone Live Browser Test Pages:**
  - [demo-1-events.html](file:///usr/local/google/home/mmocny/.gemini/chrome-ide-dev/brain/b0702da0-b2d6-4bdc-8679-51c4bfc08eef/presentation/demos/demo-1-events.html) — Normal DOM events & `PerformanceEventTiming` retargeting (`open` vs. `closed` vs. shadow-confined `mouseover`)
  - [demo-2-geometry-cls.html](file:///usr/local/google/home/mmocny/.gemini/chrome-ide-dev/brain/b0702da0-b2d6-4bdc-8679-51c4bfc08eef/presentation/demos/demo-2-geometry-cls.html) — Closed Shadow DOM layout geometry, `elementFromPoint`, and `LayoutShift` (`CLS`) attribution
  - [demo-3-lcp-resource.html](file:///usr/local/google/home/mmocny/.gemini/chrome-ide-dev/brain/b0702da0-b2d6-4bdc-8679-51c4bfc08eef/presentation/demos/demo-3-lcp-resource.html) — Reproducing `w3c/largest-contentful-paint#152` (inline `<span>` containing-block leak vs. block `<p>` `null` vs. `<img>` `url` leak)
  - [demo-4-paint-visibility.html](file:///usr/local/google/home/mmocny/.gemini/chrome-ide-dev/brain/b0702da0-b2d6-4bdc-8679-51c4bfc08eef/presentation/demos/demo-4-paint-visibility.html) — Observing `closed` Shadow DOM paint timing via Strategy 1 (global FCP) and Strategy 2 (`IntersectionObserver` v2 `trackVisibility: true` painted pixel occlusion)



