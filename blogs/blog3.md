# The Architecture of Google Chrome

## 1. Architectural Overview: Processes, Threads, Sandboxing

Modern Chrome is not “one big process.” It’s a system of cooperating processes connected by IPC, arranged to optimize **fault isolation**, **security**, and **responsiveness**:

* **Browser process** (a.k.a. the “chrome” of Chrome): owns privileged capabilities and global coordination—UI (tabs, omnibox), networking stack, disk/storage, permissions. It brokers access to OS resources and orchestrates navigation and lifecycle.
* **Renderer processes**: untrusted, sandboxed workers that host a single site (or site-tree) worth of web content—HTML/CSS/JS parsing & execution, layout, painting, compositing. Chrome prefers *site-per-process* (including cross-site iframes via **Site Isolation**) rather than tab-per-process.
* **GPU process**: aggregates GPU access from multiple clients and executes compositing and raster work on the device safely, isolated from renderers.
* **Others**: extensions, utility, plugin (historical Flash) processes show up depending on features/extensions in play.

**Why multi-process?**

1. A crashed/hung renderer only kills a tab (or frame), not the entire browser.
2. The sandbox limits the blast radius of hostile input.
3. Trade-off: each process carries its own heap and runtime (e.g., V8), which increases memory. Chrome therefore caps process counts and can consolidate when resource constrained. **Servicification** lets the browser process split/merge internal services across processes to fit device budgets.

**Security hardening: Site Isolation**
Cross-site iframes get their **own renderer process**. This aligns with the web’s core security boundary (Same-Origin Policy) and became more urgent post-Spectre/Meltdown. Consequence: even “simple” features like Find in Page or DevTools must fan out across multiple renderers. 

> Takeaway
> Treat “a page” as a *graph of processes*, not a single JS world. When you design features that depend on frame messaging (e.g., analytics pings, embedded editors), architect assuming **per-frame process boundaries**.

---

## 2. Navigation Lifecycle: From Omnibox to Commit

A user hits Enter. What actually happens?

1. **Input parsing** (Browser/UI thread): decide search vs URL, then initiate navigation.
2. **Network start** (Browser/Network thread): DNS, TLS, request dispatch. Handles 30x redirects before content streaming begins.
3. **Response classification** (Browser/Network): determine resource type primarily via `Content-Type`, but fall back to **MIME sniffing** if needed; gate dangerous responses with **Safe Browsing**; enforce **CORB** to keep sensitive cross-origin resources from reaching renderers. If the response is a download, hand off to the download manager instead of the renderer.
4. **Renderer selection** (Browser/UI): pre-warm or pick a renderer (often already queued in parallel with the request) appropriate for the destination site (and frame). 
5. **Commit** (Browser → Renderer IPC): hand over the data stream; upon renderer’s ACK, the address bar/security state/history update; renderer begins document loading. Only after all frames’ `load` events complete does the browser stop the spinner (client JS may still stream more work).

**Special cases**

* **Renderer-initiated navigations** (`<a>` clicks, `window.location=`) follow the same path but start in the renderer, including `beforeunload` coordination and old-renderer `unload` handling.
* **Service Workers**: on navigation, the Browser/Network thread checks worker scope; if matched, it spins up a renderer to run the service worker, which may satisfy from cache or proxy to network. **Navigation Preload** lets the browser begin a network request in parallel with SW startup to shave TTFB.

> Takeaways
>
> * Late `beforeunload` handlers add **latent round-trips** to every nav; use only when absolutely necessary.
> * If SW sometimes passes through to network, enable **Navigation Preload** so you’re not paying SW startup + network sequentially.

---

## 3. Inside a Renderer: Parsing → Style → Layout → Paint → Composite

Each renderer hosts several key threads:

* **Main thread**: HTML/CSS parse, DOM construction, style calc, layout, script execution, paint list, and layer assignment.
* **Compositor thread**: layerization decisions from main are consumed here; it orchestrates tile raster and submits compositor frames.
* **Raster threads**: parallelize bitmap generation of tiles, backed by GPU memory.
* **Worker threads**: Web Workers/Service Workers for off-main-thread JS.

### Parsing and subresource scheduling

* HTML → **DOM** via the WHATWG HTML parser; error-tolerant by design.
* **Preload scanner** runs concurrently to kick off fetches for `<img>`, `<link>`, `<script>`, etc., without waiting for full parse.
* **Scripts block** the parser unless `async`/`defer`/modules are used because `document.write()` and synchronous script execution can mutate DOM mid-parse. 

### Style calculation

* CSSOM + DOM → **computed styles** per element. Even with no author CSS, UA styles apply (visible in DevTools’ “Computed” panel).

### Layout (aka reflow)

* Compute the **layout tree** (geometry, positions, sizes) for visible nodes; `display:none` nodes are excluded, generated content/pseudo-elements included. Complexities include line breaking, writing modes, floats, overflow, etc.

### Paint

* Produce an ordered list of **paint records** (draw commands) that respect stacking contexts and `z-index`, not just DOM order. Any layout change can force paint order recomputation for affected regions.

### Compositing, tiles, and frames

* The main thread decides **layerization** (or you hint with `will-change`). Too many layers can hurt—measure! 
* Compositor thread chops layers into **tiles**, prioritizes in-viewport tiles for raster, generates **draw quads**, assembles a **compositor frame**, hands it to the Browser process, which submits to **GPU** for display. This pipeline enables smooth scroll/transform animations **without** blocking on the main thread.

> Takeaways
>
> * Prefer **compositor-only animations** (`transform`, `opacity`) and keep layer count within budget; use `will-change` narrowly and remove it when idle.
> * Break long JS tasks and schedule visual updates with `requestAnimationFrame`; move heavy logic to workers.

---

## 4. Input Handling: Getting Smooth While Staying Correct

Input begins in the **Browser process** (pointer/touch/mouse/keyboard). The browser forwards event type + coordinates to the owning renderer; the renderer performs **hit testing** (based on paint records) to find the target and run listeners.

### Fast paths vs main-thread dependency

The compositor can maintain buttery scroll/zoom **as long as** it doesn’t need the main thread. Any region with an input listener becomes a **Non-Fast Scrollable Region (NFSR)**; events occurring there must be routed to the main thread before compositing proceeds. A single `document.body` catch-all listener can mark the **entire page** NFSR.

**Mitigations**

* Mark passive listeners:

  ```js
  el.addEventListener('touchstart', onTouch, { passive: true });
  ```

  This tells Chrome not to wait for your handler to decide whether to block native scrolling. Use `event.cancelable` if you truly need to `preventDefault()` selectively. Or declare intent in CSS with `touch-action: pan-x;`. ([Chrome for Developers][4])
* **Coalescing**: Continuous events (`touchmove`, `pointermove`, `wheel`) are batched and dispatched just before the next `rAF` to avoid flooding the main thread beyond the display’s refresh rate. For high-fidelity drawing apps, use `event.getCoalescedEvents()` to recover intra-frame points. ([Chrome for Developers][4])

> Takeaways
>
> * Avoid “global” gesture handlers unless necessary; scope listeners to interactive regions.
> * Use `passive: true` by default; opt out only when you must block native scroll. ([Chrome for Developers][4])

---

## 5. A Practical Mental Model You Can Code Against

Think in **three planes** and **two directions**:

**Planes**

1. **Control plane (Browser process)**: navigation, security, resource fetch, storage, permissions.
2. **Compute plane (Renderer processes)**: DOM/CSSOM, layout/paint, scripting.
3. **Presentation plane (GPU/Compositor)**: layerization, tiling, raster, frame submission.

**Directions**

* **Downstream** (URL → pixels): omnibox → network → commit → parse/style/layout/paint → composite → GPU.
* **Upstream** (input → handler): device → browser → renderer hit test → handler → style/layout/paint (if needed) → compositor.

Optimizations that **change planes** pay off most: e.g., moving work off the main thread (workers), staying in the compositor (transform/opacity), avoiding main-thread dependencies for scroll (passive listeners/touch-action).

---

## 6. Code & Config Patterns That Map Cleanly to the Pipeline

**Navigation/Service Worker**

```js
// service-worker.js
self.addEventListener('fetch', e => {
  e.respondWith((async () => {
    const r = await caches.match(e.request);
    if (r) return r;
    const net = await fetch(e.request);
    const cache = await caches.open('v1');
    cache.put(e.request, net.clone());
    return net;
  })());
});

// Enable Navigation Preload (in SW 'activate')
self.registration.navigationPreload.enable();
```

This avoids sequential SW boot → network cost and overlaps work when the SW ultimately goes to network.

**Script loading**

```html
<script type="module" src="/app.js"></script>
<!-- or -->
<script defer src="/legacy.js"></script>
```

Use `type="module"`/`defer` to avoid parser stalls unless you *must* run inline and synchronously (rare).

**Compositor-friendly animations**

```css
.menu {
  will-change: transform; /* apply sparingly, remove when idle */
  transform: translateX(0);
  transition: transform 250ms ease-out;
}
.menu--open { transform: translateX(0); }
.menu--closed { transform: translateX(-100%); }
```

Keep animations on `transform`/`opacity` and avoid layout/paint triggers during motion.

**Input listeners**

```js
const panel = document.querySelector('#panel');
panel.addEventListener('pointermove', e => {
  if (e.cancelable && shouldLockY(e)) e.preventDefault();
  updateUI(e);
}, { passive: true }); // compositor can keep scrolling
```

Scope listeners and default to passive; rely on `touch-action` in CSS where possible.

---

## 7. What to Measure (and Why It Matters to the Architecture)

* **Long tasks on main** → block parse/style/layout/paint. Break them up and push CPU to workers.
* **Layout thrash** → repeated sync reads/writes force relayout and invalidate paint order. Batch reads, then writes.
* **Layer count** → too high: memory pressure & compositor overhead; too low: missed compositor-only animation opportunities.
* **Input delay** (FID/INP) → often caused by global listeners or main-thread queue congestion. Adopt passive listeners and reduce NFSR footprint.

Use **Lighthouse** + **DevTools Performance** to see where you’re crossing process/thread boundaries and which stage is hot.

---

## 8. Closing Perspective

Chrome’s architecture is explicitly designed to **decouple**: navigation/security (browser), computation of layout/paint/script (renderer), and presentation (GPU). The performance you see is the emergent property of *how your code engages each plane*. If you’re intentional about where work runs and which stages you invalidate, you’ll consistently ship experiences that feel native—because you’re cooperating with the browser’s design, not fighting it.

---

### Sources

- https://developer.chrome.com/blog/inside-browser-part1 "Inside look at modern web browser (part 1)  |  Blog  |  Chrome for Developers"
- https://developer.chrome.com/blog/inside-browser-part2 "Inside look at modern web browser (part 2)  |  Blog  |  Chrome for Developers"
- https://developer.chrome.com/blog/inside-browser-part3 "Inside look at modern web browser (part 3)  |  Blog  |  Chrome for Developers"
- https://developer.chrome.com/blog/inside-browser-part4 "Inside look at modern web browser (part 4)  |  Blog  |  Chrome for Developers"

---

Follow me for more write-ups:
[X/Twitter](https://x.com/KAshSecurity) | [YouTube](https://www.youtube.com/@SecurityWithKAsh) | [GitHub](https://github.com/KAshSecurity)