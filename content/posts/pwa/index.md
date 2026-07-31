+++
title = 'Ship It Offline: Building a Production-Grade Progressive Web App'
date = '2026-07-28T00:55:00+05:30'
draft = false
description = 'What separates a demo PWA from a production one - web app manifest, service workers, caching strategies, updates, security, and a concrete checklist with code.'
tags = ['PWA', 'Service Worker', 'Frontend', 'Web Performance', 'Offline', 'Engineering']
categories = ['Engineering', 'Frontend']
summary = 'A production PWA needs HTTPS, a real manifest, disciplined caching, and a safe update path - not just a Lighthouse badge. Here is how to build and ship one.'
+++

<img src="/images/posts/pwa/cover.svg" alt="PWA between website and native app" width="960" height="400" />

*A PWA is not "add a service worker and hope." Production means installability, offline behavior you designed, and updates that do not trap users on a broken build.*

Progressive Web Apps use web tech to feel closer to native: install to the home screen, work offline or on flaky networks, load fast on repeat visits. The bar for a **demo** is low. The bar for **production** is operational.

This post covers what "production-grade" actually requires, with concrete files and a ship checklist.

---

## What a PWA is (and is not)

A PWA is a website that meets a set of capabilities:

| Capability | Mechanism |
|------------|-----------|
| Installable | Web App Manifest + icons + (usually) a registered service worker |
| Works offline / flaky net | Service Worker + Cache Storage (and/or IndexedDB) |
| Secure context | **HTTPS** (localhost allowed for development) |
| App-like shell | `display: standalone` / `minimal-ui`, themed UI |

It is **not**:

- a guarantee of App Store distribution (though some stores accept PWAs / TWA wrappers)
- automatic push notifications (that is a separate permissioned API)
- a reason to ignore accessibility, auth, or backend reliability

---

## The production baseline

<img src="/images/posts/pwa/baseline.svg" alt="HTTPS, manifest, service worker, ops layer" width="960" height="420" />

### 1. HTTPS everywhere

Service workers will not register on insecure origins (except `localhost` / `127.0.0.1`). In production you need TLS end-to-end, including any CDN.

Also set sane security headers (at least start here):

```http
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
Content-Security-Policy: default-src 'self'; ...
```

CSP must allow your SW script origin and any CDN you really need. Over-strict CSP + SW is a common "works in Chrome flags, fails in prod" bug.

### 2. Web App Manifest

`public/manifest.webmanifest` (or `manifest.json`):

```json
{
  "name": "Northwind Inventory",
  "short_name": "Northwind",
  "description": "Warehouse inventory for field staff",
  "start_url": "/?source=pwa",
  "scope": "/",
  "display": "standalone",
  "orientation": "any",
  "background_color": "#0f172a",
  "theme_color": "#0f172a",
  "icons": [
    {
      "src": "/icons/icon-192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "/icons/icon-512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "/icons/icon-512-maskable.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "maskable"
    }
  ]
}
```

Link it from HTML:

```html
<link rel="manifest" href="/manifest.webmanifest" />
<meta name="theme-color" content="#0f172a" />
<meta name="apple-mobile-web-app-capable" content="yes" />
<link rel="apple-touch-icon" href="/icons/icon-192.png" />
```

Production details people skip:

- Provide **maskable** icons (safe zone) or Android crops your logo  
- `start_url` should be a real route that works offline (often the app shell)  
- `short_name` appears under the icon - keep it short  
- Prefer `.webmanifest` with `application/manifest+json` content type  

### 3. Service Worker with a versioned cache

Register only on HTTPS / localhost, after the page is interactive:

```js
// register-sw.js
if ("serviceWorker" in navigator) {
  window.addEventListener("load", () => {
    navigator.serviceWorker
      .register("/sw.js", { scope: "/" })
      .then((reg) => {
        // Optional: listen for updates
        reg.addEventListener("updatefound", () => {
          const worker = reg.installing;
          worker?.addEventListener("statechange", () => {
            if (worker.state === "installed" && navigator.serviceWorker.controller) {
              // New SW waiting - show "Refresh to update" UI
              console.log("Update available");
            }
          });
        });
      })
      .catch(console.error);
  });
}
```

Minimal production-shaped worker (vanilla; Workbox is fine too):

```js
// sw.js
const VERSION = "v2026.07.28.1"; // bump on every release
const PRECACHE = `precache-${VERSION}`;
const RUNTIME = `runtime-${VERSION}`;

const PRECACHE_URLS = [
  "/",
  "/index.html",
  "/offline.html",
  "/manifest.webmanifest",
  "/icons/icon-192.png",
  "/assets/app.css",
  "/assets/app.js",
];

self.addEventListener("install", (event) => {
  event.waitUntil(
    caches.open(PRECACHE).then((cache) => cache.addAll(PRECACHE_URLS))
  );
  // Optional: self.skipWaiting() if you intentionally take over immediately
});

self.addEventListener("activate", (event) => {
  event.waitUntil(
    (async () => {
      const keys = await caches.keys();
      await Promise.all(
        keys
          .filter((key) => key !== PRECACHE && key !== RUNTIME)
          .map((key) => caches.delete(key))
      );
      await self.clients.claim();
    })()
  );
});

self.addEventListener("fetch", (event) => {
  const { request } = event;
  if (request.method !== "GET") return;

  const url = new URL(request.url);
  if (url.origin !== self.location.origin) return; // don't cache arbitrary third parties by default

  // HTML navigations: network first, offline fallback
  if (request.mode === "navigate") {
    event.respondWith(networkFirst(request));
    return;
  }

  // Fingerprinted static assets: cache first
  if (url.pathname.startsWith("/assets/")) {
    event.respondWith(cacheFirst(request));
    return;
  }

  // Default: stale-while-revalidate
  event.respondWith(staleWhileRevalidate(request));
});

async function cacheFirst(request) {
  const cached = await caches.match(request);
  if (cached) return cached;
  const fresh = await fetch(request);
  const cache = await caches.open(RUNTIME);
  cache.put(request, fresh.clone());
  return fresh;
}

async function networkFirst(request) {
  try {
    const fresh = await fetch(request);
    const cache = await caches.open(RUNTIME);
    cache.put(request, fresh.clone());
    return fresh;
  } catch {
    const cached = await caches.match(request);
    return cached || caches.match("/offline.html");
  }
}

async function staleWhileRevalidate(request) {
  const cache = await caches.open(RUNTIME);
  const cached = await cache.match(request);
  const networkPromise = fetch(request)
    .then((response) => {
      cache.put(request, response.clone());
      return response;
    })
    .catch(() => cached);
  return cached || networkPromise;
}
```

**Fact:** a bad service worker is worse than no service worker. Users can be stuck on a cached error page until they clear site data. Always ship an update path.

---

## Caching strategies (choose per asset)

<img src="/images/posts/pwa/caching.svg" alt="Cache first, network first, stale-while-revalidate" width="960" height="460" />

| Asset | Strategy | Why |
|-------|----------|-----|
| Hashed JS/CSS (`app.abc123.js`) | Cache-first | Content-addressed; safe to keep forever until VERSION bump |
| App shell HTML | Network-first + offline.html | Prefer fresh shell; degrade gracefully |
| API GET lists | Stale-while-revalidate or network-first | UX vs freshness tradeoff |
| Auth / mutations | Network only | Never cache POSTs; do not cache private responses loosely |
| Images | Cache-first with size limits | Cap runtime cache or you will fill disk |

Production rules:

1. **Never** cache `Set-Cookie` / personalized HTML without thinking  
2. Vary caches by auth carefully - or exclude authenticated pages from SW caching  
3. Cap runtime caches (Workbox `ExpirationPlugin` or manual trim)  
4. Prefer **build-time precache manifests** (Workbox `InjectManifest` / Vite PWA plugin) over hand-maintained URL lists  

### Vite example (common in 2020s stacks)

```bash
npm i -D vite-plugin-pwa
```

```js
// vite.config.ts
import { VitePWA } from "vite-plugin-pwa";

export default {
  plugins: [
    VitePWA({
      registerType: "prompt", // better than auto silent takeover for many apps
      includeAssets: ["icons/*.png", "offline.html"],
      manifest: {
        name: "Northwind Inventory",
        short_name: "Northwind",
        theme_color: "#0f172a",
        background_color: "#0f172a",
        display: "standalone",
        start_url: "/",
        icons: [
          { src: "icons/icon-192.png", sizes: "192x192", type: "image/png" },
          { src: "icons/icon-512.png", sizes: "512x512", type: "image/png" },
          {
            src: "icons/icon-512-maskable.png",
            sizes: "512x512",
            type: "image/png",
            purpose: "maskable",
          },
        ],
      },
      workbox: {
        navigateFallback: "/index.html", // SPA shells only
        runtimeCaching: [
          {
            urlPattern: ({ url }) => url.pathname.startsWith("/api/"),
            handler: "NetworkFirst",
            options: {
              cacheName: "api",
              networkTimeoutSeconds: 5,
              expiration: { maxEntries: 64, maxAgeSeconds: 60 * 60 },
            },
          },
        ],
      },
    }),
  ],
};
```

`registerType: "prompt"` forces you to show "Update available" UI - usually safer than surprising users mid-checkout.

---

## Update lifecycle (the part demos skip)

<img src="/images/posts/pwa/lifecycle.svg" alt="Install activate fetch update service worker lifecycle" width="960" height="400" />

Typical flow:

1. User has SW **A** controlling the page  
2. You deploy SW **B** (new `VERSION` / new Workbox revision)  
3. Browser downloads **B**, runs `install`, then **B waits**  
4. You either:
   - prompt the user and call `skipWaiting` + reload, or  
   - wait until all tabs close (safer, slower rollout)

Client snippet when using a waiting worker:

```js
function promptUserToRefresh(registration) {
  const ok = window.confirm("A new version is available. Refresh now?");
  if (!ok) return;
  registration.waiting?.postMessage({ type: "SKIP_WAITING" });
}

navigator.serviceWorker.addEventListener("controllerchange", () => {
  window.location.reload();
});
```

In `sw.js`:

```js
self.addEventListener("message", (event) => {
  if (event.data?.type === "SKIP_WAITING") self.skipWaiting();
});
```

Also keep a **kill switch**: host a tiny `sw-kill.js` or document how support tells users to unregister if a bad SW ships (`chrome://serviceworker-internals` / Application panel).

---

## Installability and platform reality

### Chromium (Android / desktop)

Install criteria generally include: HTTPS, manifest with required fields + icons, and a service worker controlling the page (details evolve - test on target Chrome).

Handle `beforeinstallprompt` if you want an in-app Install button:

```js
let deferredPrompt;

window.addEventListener("beforeinstallprompt", (e) => {
  e.preventDefault();
  deferredPrompt = e;
  document.getElementById("install-btn").hidden = false;
});

document.getElementById("install-btn")?.addEventListener("click", async () => {
  if (!deferredPrompt) return;
  deferredPrompt.prompt();
  await deferredPrompt.userChoice;
  deferredPrompt = null;
});
```

### iOS / iPadOS Safari

- Add to Home Screen works, but behavior differs from Android  
- `apple-touch-icon` and `apple-mobile-web-app-capable` still matter  
- Push notifications and background sync support have historically lagged Chromium - verify on your target iOS version before promising features in marketing  

**Production advice:** test install + offline on a real iPhone, not only Lighthouse on desktop.

---

## Offline UX that does not lie

Shipping `/offline.html` is not enough.

Good production patterns:

1. **App shell** loads offline; show cached data with a "You're offline" banner  
2. Queue writes (IndexedDB outbox) and sync when online (`online` event or Background Sync where supported)  
3. Never show a success toast for a mutation that only sat in a local queue - say "Saved on device - will sync"  
4. For SPAs, precache the shell and critical routes; do not claim the whole site is offline-ready if only `/` works  

IndexedDB outbox sketch:

```js
async function queueMutation(payload) {
  const db = await openDB(); // idb helper or native IndexedDB
  await db.put("outbox", { id: crypto.randomUUID(), payload, createdAt: Date.now() });
}

window.addEventListener("online", () => {
  flushOutbox(); // POST queued items with idempotency keys
});
```

Use **idempotency keys** on the server so retries do not double-charge or double-create.

---

## Performance and Core Web Vitals still count

A PWA that installs but paints in 6 seconds is not production-grade.

Targets to measure (field data via CrUX / RUM, not only lab):

| Metric | Why it matters for PWAs |
|--------|-------------------------|
| LCP | Shell and hero must be fast on 3G / mid phones |
| INP | Install prompts + SW messages should not block input |
| CLS | Icon/font loading must not shove layout |
| Cache hit ratio | Prove repeat visits are actually faster |

Precache only what you need. Over-precache hurts first install on mobile data.

---

## Security extras for real apps

1. **Scope** the SW tightly (`scope: "/app/"` if marketing site should not be controlled)  
2. Do not put secrets in the SW or precache  
3. Treat cached API data as sensitive - clear caches on logout  
4. Restrict third-party script caching  
5. Review third-party CDNs - an XSS + SW can become a persistent XSS  

Logout helper:

```js
async function clearAppCaches() {
  const keys = await caches.keys();
  await Promise.all(keys.map((k) => caches.delete(k)));
  // also clear IndexedDB outbox / user stores
}
```

---

## Testing checklist (before you call it prod)

### Functional

- [ ] First visit online installs SW without console errors  
- [ ] Second visit works with DevTools → Network → Offline  
- [ ] Hard refresh after deploy eventually gets new assets  
- [ ] "Update available" prompt works on a waiting worker  
- [ ] Logout clears personalized caches  
- [ ] Install on Android Chrome + Add to Home Screen on iOS  

### Non-functional

- [ ] Lighthouse PWA category is a smoke test, not the release bar  
- [ ] Bundle size budget for precache (for example under a few MB critical path)  
- [ ] Error monitoring (Sentry etc.) works in standalone display mode  
- [ ] Analytics does not assume always-online  

### Failure drills

- [ ] Ship a deliberate bad SW to staging, then a fix - confirm recovery  
- [ ] Fill offline outbox, go online, confirm exactly-once server behavior  

---

## Architecture that scales

For larger teams, treat the PWA like any other release train:

| Concern | Practice |
|---------|----------|
| Versioning | Embed build id in SW (`VERSION`) and UI footer |
| Rollout | Percentage CDN / feature flag; avoid silent `skipWaiting` for all users on day one |
| Observability | Log SW activate, cache match/miss, update prompts accepted |
| Ownership | One team owns SW regressions - they are user-facing outages |
| Docs | Runbook: "How to force-unregister SW for support" |

Related on this blog: if your frontend ships from a [monorepo](/posts/monorepos/), generate the precache manifest in the same CI pipeline that hashes assets - never hand-edit production cache lists.

---

## Minimal production file set

```text
/
  index.html              # links manifest + register-sw.js
  offline.html            # honest offline fallback
  manifest.webmanifest
  sw.js                   # versioned caches + strategies
  register-sw.js
  icons/
    icon-192.png
    icon-512.png
    icon-512-maskable.png
  assets/
    app.[hash].css
    app.[hash].js
```

---

## Closing

A **production-grade PWA** is a product surface with:

1. HTTPS + solid headers  
2. A complete manifest and icons (including maskable)  
3. A service worker with **per-asset strategies**  
4. A deliberate **update UX** and kill switch  
5. Offline behavior that matches user expectations  
6. Security hygiene around caches and auth  
7. Tests on real Android and iOS devices  

Lighthouse can tell you the badges. Production is what happens on a subway phone with 2% battery during your worst deploy.

**Next step:** add `offline.html`, bump a `VERSION` constant, and implement an "Update ready - Refresh" banner before you add push notifications or advanced sync. Get the update path right first - everything else builds on it.
