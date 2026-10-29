# Shadow DOM in Web Performance APIs

**[▶ Open the Live Presentation & Demos](https://mmocny.com/shadow-dom-webperf/)**

Interactive presentation and live test pages for the W3C Web Performance Working Group exploring how Shadow DOM works across the Web Platform, what breaks today when measuring Web Components with Performance APIs, and how to align WebPerf specifications.

---

## Standalone Interactive Demos

Each demo is also available as a standalone test page:

1. **[`demos/demo-0-queryselector.html`](https://mmocny.com/shadow-dom-webperf/demos/demo-0-queryselector.html)** — How `document.querySelectorAll()` selects Light DOM hosts (`<light-demo-card>`, `<open-demo-card>`, `<closed-demo-card>`) while stopping at the Shadow Boundary, and how `host.shadowRoot` reaches into `open` roots only.
2. **[`demos/demo-1-events.html`](https://mmocny.com/shadow-dom-webperf/demos/demo-1-events.html)** — How composed DOM events bubble out of both `open` and `closed` Shadow Roots with `event.target` retargeted to the Shadow Host (`?mode=webperf` enables `PerformanceEventTiming` observation).
3. **[`demos/demo-4-paint-visibility.html`](https://mmocny.com/shadow-dom-webperf/demos/demo-4-paint-visibility.html)** — How slow subresources (`<img>`) inside both `open` and `closed` Shadow Roots delay the outer document's `window.onload`, and how `document.elementFromPoint(x, y)` retargets to the Shadow Host.
4. **[`demos/demo-2-geometry-cls.html`](https://mmocny.com/shadow-dom-webperf/demos/demo-2-geometry-cls.html)** — Hit-testing (`elementFromPoint`) and `LayoutShift` (`CLS`) attribution on shifting elements inside Shadow DOM (`?mode=webperf`).
5. **[`demos/demo-3-lcp-resource.html`](https://mmocny.com/shadow-dom-webperf/demos/demo-3-lcp-resource.html)** — 3-step live reproduction of [`w3c/largest-contentful-paint#152`](https://github.com/w3c/largest-contentful-paint/issues/152) inside a `closed` Shadow Root (inline `<span>` text aggregation walk-past, block `<p>` with `element: null`, and `<img>` with `url` reported but `element: null`).

---

## Summary of Proposed WebPerf WG Resolutions

1. **Include Author Shadow DOM (`open` & `closed`); Exclude `UA` Shadow Roots:**
   Update Paint Timing and Largest Contentful Paint to traverse shadow-including descendants for author `open` and `closed` roots equally, while excluding User-Agent (`UA`) shadow trees.
2. **Default View (Component Users) — Retarget `element`, `id` & `node` to Host; Keep `url`:**
   Introduce a shared `retarget for performance timing(node, document, allowedScopes)` algorithm that retargets `LCP.element` (and `LCP.id` → Host's `id`), `CLS.sources[i].node`, `LoAF.invoker`, and `ContainerTiming` to the Shadow Host, preserves slotted Light DOM nodes, and keeps `LCP.url` reported (matching `ResourceTiming`).
3. **Cap Inline Text & Decide `targetSelector` (`event-timing#126`):**
   Cap Paint Timing inline text aggregation at Shadow Host boundaries, and decide whether Event Timing's `targetSelector` pierces `open` shadow roots (like `event.composedPath()`) or stops at the Shadow Host.
4. **Opt-In View (Component Authors) — `{ shadowRoots, root }` on `observe()`:**
   Allow Component Authors holding a `ShadowRoot` reference to pass `shadowRoots` / `root` in `PerformanceObserverInit` to unlock un-retargeted inner attribution *and* receive entries filtered from the main document timeline (`elementtiming`, `containertiming`, and shadow-confined `event`s).

---

## Related Spec Issues

- [`w3c/largest-contentful-paint#152`](https://github.com/w3c/largest-contentful-paint/issues/152) — LCP and Shadow DOM
- [`w3c/element-timing#3`](https://github.com/w3c/element-timing/issues/3) — Element Timing for Shadow DOM (Component Users vs. Component Authors)
- [`w3c/container-timing#14`](https://github.com/w3c/container-timing/issues/14) — Container Timing and Shadow DOM
- [`w3c/event-timing#126`](https://github.com/w3c/event-timing/issues/126) — Event Timing `targetSelector` and Shadow DOM
