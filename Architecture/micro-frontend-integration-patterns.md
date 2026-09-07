# 🧩 Micro-Frontend Integration Patterns: The Definitive In-Depth Guide

> **Context:** This document is designed for engineering teams working with large-scale Micro-Frontend (MFE) architectures, particularly those already running Single-SPA in production. It breaks down the four fundamental integration models, their trade-offs, the tooling that's actually matured by 2026, and how hybrid approaches are shaping modern enterprise frontend development.

---

## 📑 Table of Contents

- [🎯 The Core Problem: Where & When to Stitch](#-the-core-problem-where--when-to-stitch)
- [1. Build-Time Integration](#1-build-time-integration)
- [2. Server-Side Composition (Edge Composition / SSI)](#2-server-side-composition-edge-composition--ssi)
- [3. Runtime (Client-Side) Integration](#3-runtime-client-side-integration)
- [4. Isolation-First Integration: iframes & Web Components](#4-isolation-first-integration-iframes--web-components)
- [📊 Quick Comparison Matrix](#-quick-comparison-matrix)
- [🧬 Where Hybrid Approaches Show Up in Practice](#-where-hybrid-approaches-show-up-in-practice)
- [🚀 Deep Dive: Operating 30+ Apps in Single-SPA / Module Federation](#-deep-dive-operating-30-apps-in-single-spa--module-federation)
- [🔬 Testing & Observability at Scale](#-testing--observability-at-scale)
- [🔒 Security Considerations for MFEs](#-security-considerations-for-mfes)
- [🚫 When *Not* to Use Micro-Frontends](#-when-not-to-use-micro-frontends)
- [📚 Deep-Dive Reference Index](#-deep-dive-reference-index)

---

## 🎯 The Core Problem: Where & When to Stitch

Micro-frontend integration is fundamentally about **where** and **when** independently built and deployed frontend fragments get stitched together into one cohesive user experience.

The architectural models differ mainly in *when* that stitching happens in the application lifecycle:

1. **Build-Time:** Stitching during CI/CD.
2. **Server-Side:** Stitching on the server/edge before reaching the browser.
3. **Runtime (Client-Side):** Stitching inside the browser after the shell loads.
4. **Isolation-First:** Stitching happens in the browser too, but via a hard boundary (iframe/Shadow DOM) rather than a shared JS runtime.

> **2026 Reality Check:** Micro-frontends have moved past the hype phase. The tooling matured significantly around 2024–2025, but the honest industry consensus is now: **MFEs are not a default architecture — they're a solution to a specific organizational problem** (independent team deploy cadences at scale). Section 10 covers when *not* to reach for this.

---

## 1. Build-Time Integration

**How it works:** Each micro-frontend is published as an internal package (e.g., npm). The "shell" application imports and bundles them together at build time, producing a single compiled artifact.

```text
MFE-A (npm package) ─┐
MFE-B (npm package) ─┼─→ Shell build process → single bundle → deploy
MFE-C (npm package) ─┘
```

### Characteristics
* Uses standard package management (`package.json` dependencies).
* Shell app does `import MfeA from '@org/mfe-a'` and bundles everything via Webpack/Vite.
* Deployment is monolithic — one build, one deploy, even though development was distributed.

### Evaluation

| Pros ✅ | Cons ❌ |
|---|---|
| **Simplest mental model** — feels like a standard monolith app. | **Deployment coupling (The Killer)** — If MFE-B ships a fix, the *entire shell* must be rebuilt and redeployed. You lose the core promise of MFEs. |
| **No runtime overhead** — No shared-dependency version conflicts at runtime; everything resolved once at build. | **Blocked Cadences** — Teams are blocked by each other's release cadence and CI pipelines. |
| **Strong type-safety** — Compiles together; easy to catch integration bugs at build time. | **Bundle bloat** — Everything ships together regardless of route usage. |
| | **Poor Scalability** — Degrades quickly past a handful of teams. |

> 💡 **When it's reasonable:** Small number of MFEs, tightly coordinated teams, or a "distributed development, unified release" org model (e.g., internal tools or design systems where release coordination isn't painful). Teams increasingly manage this flavor inside an **Nx or Turborepo monorepo** rather than separate npm packages — same coupling trade-off, but with shared caching and affected-project builds softening the CI pain.

---

## 2. Server-Side Composition (Edge Composition / SSI)

**How it works:** Each micro-frontend renders its markup fragment independently (often via SSR). A server layer (or edge/CDN layer) assembles the fragments into a single HTML response *before* it reaches the browser.

**Common Implementations:**
* **ESI (Edge Side Includes):** At the CDN/reverse-proxy layer (Varnish, Fastly, Akamai).
* **Application-level composition:** A Node.js "composer" service calls each MFE's SSR endpoint and stitches HTML (e.g., Zalando's Project Mosaic, Amazon's "Tailor" pattern).
* **Podium:** NRK's well-known open-source implementation of this pattern.

```text
Browser → CDN/Edge → Composer Layer
                         ├─→ fetch MFE-A HTML fragment (SSR)
                         ├─→ fetch MFE-B HTML fragment (SSR)
                         └─→ fetch MFE-C HTML fragment (SSR)
                       → assembled HTML → Browser
```

### Evaluation

| Pros ✅ | Cons ❌ |
|---|---|
| **Best first-load performance** — Browser receives fully composed HTML; no client-side waterfall. | **High infra complexity** — Requires a composition layer, fragment caching strategy, and layout/CSS coordination. |
| **SEO-critical friendly** — Great for content-heavy retail/media sites (Zalando, IKEA). | **Hydration complexity** — Cross-fragment interactivity is tricky; requires "islands" style hydration boundaries. |
| **Independent deploys** — Each team owns SSR of their fragment independently. | **Hard local dev experience** — Fragments need to be served or mocked locally to test composition. |
| **Resilient** — Works even with JS disabled or on slow devices. | **Shared state is hard** — Post-hydration communication across fragments requires a client-side event bus; not natively solved here. |

---

## 3. Runtime (Client-Side) Integration

**How it works:** The browser loads a thin shell/root app, which then dynamically fetches and mounts each MFE's JS bundle at runtime, based on the current route or context.

```text
Browser loads shell (root-config)
        ↓
  Router determines active route
        ↓
  Shell dynamically imports MFE bundle (via SystemJS / Module Federation / Native Federation)
        ↓
  MFE mounts into a DOM container, shell manages lifecycle (bootstrap/mount/unmount)
```

There are now **three** distinct flavors worth knowing, not two — the third (Native Federation) matured significantly through 2025–2026.

### Flavor A: Single-SPA Style
* `root-config` + Import Maps (SystemJS) resolve each MFE's entry URL at runtime.
* Each MFE exposes `bootstrap/mount/unmount` lifecycle hooks.
* Shell orchestrates which MFE is active per route.
* **2026 status:** framework-agnostic, mature, huge community — but the programming model is imperative (you register apps and wire lifecycle hooks by hand), and its developer experience now visibly lags Module Federation 2.0. It remains the strongest choice for **brownfield** setups integrating legacy or mixed-framework apps; for greenfield builds, most teams default to MF 2.0 instead (see Flavor B).

### Flavor B: Module Federation (now "2.0")
* Apps expose/consume modules directly via federation config.
* Allows more granular sharing — not just whole apps, but individual *components*.
* Shared singleton deps (React, etc.) are resolved at runtime gracefully.

**What changed in Module Federation 2.0** (stable since April 2024, and the default recommendation by 2026): it's no longer just a Webpack 5 plugin. It ships as a decoupled **Federation Runtime** with:
* **Dynamic TypeScript type hints** — consuming a remote module in TS used to mean losing static types (or maintaining hand-copied shared type packages). MF 2.0 generates and hot-reloads types from remotes automatically, closer to an `npm link` experience.
* **A Manifest** — a metadata file describing available remotes/versions, replacing hand-maintained remote URLs.
* **A Runtime Plugin System** — hook into resolution, loading, and error handling programmatically.
* **First-class Node.js support** — remote modules can now be consumed by SSR layers and BFF services too, not just browser bundles, unifying module delivery across frontend and backend.
* **Bundler-agnostic** — natively supported in Rspack and Vite now, not locked to Webpack.

```javascript
// webpack.config.js — Module Federation 2.0 host
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'shell',
      remotes: {
        dashboard: 'dashboard@https://cdn.example.com/dashboard/mf-manifest.json',
      },
      shared: {
        react: { singleton: true, requiredVersion: '^18.0.0' },
      },
    }),
  ],
};
```

**Real production numbers (2026 benchmarks):** with a proper setup, a 3-remote configuration adds roughly ~50–80ms to first meaningful paint versus an equivalent monolith — shared React loads once (~130KB gzipped) instead of per-remote, and each remote gets its own cache fingerprint, so a Cart deploy doesn't invalidate the Catalog cache.

### Flavor C: Native Federation
* A **bundler-agnostic** implementation of the same mental model as Module Federation, but built entirely on **browser-native ECMAScript Modules and Import Maps** — no bundler-specific runtime required.
* Originated in the Angular ecosystem (`@softarc/native-federation` by Manfred Steyer) and has first-class Angular CLI support, but works with any framework and any bundler (Vite, Rollup, esbuild) since it doesn't depend on Webpack internals.
* **SSR/hydration support** shipped from v18.2.3 onward — a gap that used to block server-rendered Native Federation apps.
* **Trade-off:** because it leans on the browser's native module loader rather than a bundler's optimized runtime, it's measurably slower than Module Federation's resolution path — the honest 2026 read is "good for Angular projects specifically, less battle-tested for React/Vue at scale."
* **Can combine with Module Federation:** several teams run Native Federation and Module Federation side-by-side during a migration, since their host/remote mental model is intentionally near-identical.

### Evaluation (applies to all three runtime flavors)

| Pros ✅ | Cons ❌ |
|---|---|
| **True independent deployability** — Ship bundle, update import map/manifest, done. No shell rebuild. | **Client-side waterfall** — Shell loads → fetches MFE → mounts. Slower first paint than server composition (requires prefetching/preloading to mitigate). |
| **Flexible granularity** — Mount/unmount per route, or co-mount multiple MFEs (e.g., Header + Body + Footer simultaneously). | **Shared dependency hell** — React/ReactDOM version mismatches require `singleton: true` (Module Fed) or strict externals (Single-SPA). Otherwise: bundle bloat and hook errors. |
| **Good dev/prod parity** — Same runtime composition mechanism in both environments. | **Runtime failures** — A bad deploy of one MFE can break the UI with no compile-time safety net across boundaries. |
| **Incremental migration** — Strangler-fig pattern; swap one MFE without touching others. | **CSS Isolation** — CSS leakage needs active management (Shadow DOM, CSS Modules, strict naming conventions). |
| **MF 2.0 specifically:** typed remotes, Node.js-shared modules, per-remote cache fingerprints. | **SEO requires work** — Default is CSR; needs SSR-per-shell or prerendering add-on. |

---

## 4. Isolation-First Integration: iframes & Web Components

This is the model most write-ups skip, but it's a legitimate fourth pattern — and the one most teams *start* with before they understand the trade-offs of the other three.

### iframes
**How it works:** Each MFE is a genuinely separate document, embedded via `<iframe src="...">`. This is the strongest possible isolation boundary the browser offers.

| Pros ✅ | Cons ❌ |
|---|---|
| **Total isolation** — separate JS realm, separate CSS scope, separate memory. Zero risk of one MFE's bug crashing another. | **No shared routing** — the browser URL and the iframe's internal URL diverge unless manually synced. |
| **Independent everything** — different frameworks, different React versions, no singleton coordination needed at all. | **Cross-frame communication is painful** — `postMessage` only; no direct DOM/JS access across the boundary. |
| **Simplest security model** — sandbox attribute, CSP framing controls, no shared execution context to audit. | **UX seams** — resizing to content, focus management, scroll behavior, and accessibility (screen readers announcing frame boundaries) all need active work. |
| | **SEO is effectively broken** without server-side tricks — crawlers don't traverse iframe content the same way. |

> **Where this still shows up in 2026:** embedding a genuinely third-party or security-sensitive widget (payment forms, an acquired company's legacy app you haven't migrated yet) — situations where isolation matters more than integration quality.

### Web Components (Custom Elements)
**How it works:** Each MFE compiles to a **Custom Element** (`<mfe-cart></mfe-cart>`) with an internal Shadow DOM boundary for CSS/DOM encapsulation, then any shell — regardless of *its* framework — can mount it like a native HTML tag.

* **Orthogonal, not competing:** Web Components aren't really a 4th timing model on their own — they're a *packaging format* that Runtime Integration (Section 3) can use instead of raw JS bundles. Angular Elements is a common way to expose a Single-SPA or Module Federation MFE as a custom element.
* **Why teams reach for it:** true framework-agnostic mounting (a Vue shell can mount a React-built custom element with zero React knowledge required), and Shadow DOM gives you real CSS isolation without naming-convention discipline.
* **Trade-off:** cross-framework event/prop passing through a custom element's API is clunkier than native framework props, and Shadow DOM has its own quirks with global CSS (design tokens, third-party UI libraries expecting to reach into `document`).

---

## 📊 Quick Comparison Matrix

| Dimension | Build-Time | Server Composition | Runtime (MF/SS/NF) | Isolation-First (iframe) |
|---|---|---|---|---|
| **Deploy Independence** | ❌ Low | ✅ High | ✅ High | ✅ High |
| **First-Load Perf** | ⚪ Good (single bundle) | ✅ Best | ⚠️ Waterfall risk | ⚠️ Extra document overhead |
| **Infra Complexity** | ✅ Low | ❌ High | ⚠️ Medium | ✅ Low |
| **Dev Experience** | ✅ Simple | ❌ Hardest | ⚠️ Medium (tooling helps a lot in 2026) | ✅ Simple |
| **SEO out of the box** | ✅ Yes | ✅ Yes | ❌ Needs SSR/prerender | ❌ Effectively broken |
| **Shared Dep Risk** | ✅ None (resolved at build) | ✅ None (self-contained) | ❌ Real risk (singleton mgmt) | ✅ None (fully isolated) |
| **Cross-fragment UX** | ✅ Native | ⚠️ Needs islands/event bus | ✅ Native (same document) | ❌ postMessage only |
| **Granularity** | App-level | Fragment/page-region | App or component-level | Frame-level only |

---

## 🧬 Where Hybrid Approaches Show Up in Practice

Mature MFE setups are rarely purebred. They blend patterns to mitigate the cons of specific models.

### 1. Runtime Shell + Build-Time Shared Libraries
* **Pattern:** Your Single-SPA root config does runtime composition, but a shared design-system/component library is still consumed as a build-time `npm` dependency by each MFE individually.
* **Why:** Keeps UI consistency high while maintaining independent deployment of business logic.

### 2. Server Shell + Runtime Interactive Islands
* **Pattern:** SSR the initial page structure for performance/SEO, then hydrate individual "islands" with client-side runtime-loaded MFEs for interactivity.
* **Why:** This is the **Islands Architecture** (popularized by Astro) applied to MFEs. You get the SEO/Perf of Server Composition with the dynamic interactivity of Runtime. Note: Astro Islands themselves aren't strictly micro-frontends (no independent team-owned deployability), but the composition *mechanism* — independently hydrated fragments on one page — is the same idea.

### 3. Module Federation + Single-SPA Together
* **Pattern:** Single-SPA for lifecycle/routing orchestration, Module Federation as the actual mechanism for fetching/sharing modules.
* **Why:** This is increasingly common — and by 2026, arguably the default combination for large orgs. Single-SPA provides the app-level routing and mounting lifecycle, while Module Federation (2.0 specifically) solves the shared-singleton dependency problem, typed remotes, and manifest-based versioning far more elegantly than raw SystemJS import maps.

### 4. Federation + Web Components at the Boundary
* **Pattern:** MFEs are federated via Module Federation/Native Federation, but each one's *public mount API* is a Custom Element rather than a raw `bootstrap/mount/unmount` export.
* **Why:** Gives you MF's dependency-sharing and versioning benefits while keeping the mounting contract framework-agnostic and self-describing (a plain HTML tag), which eases onboarding non-JS-framework consumers (e.g., a CMS-driven page builder).

---

## 🚀 Deep Dive: Operating 30+ Apps in Single-SPA / Module Federation

*If you are running 30+ apps under Single-SPA already, the dependency-versioning and orchestration pain points in the Runtime cons column are likely your most concrete live issues. Here is a breakdown of how to manage them — including where Module Federation 2.0 changes the playbook if you're layering it in.*

### 1. Singleton Management (The React Problem)
Running multiple React versions is the #1 cause of hooks crashes (`Invalid hook call`).
* **The Fix:** You must force React and React-DOM to be singletons.
* **Single-SPA way:** Mark them as `externals` in Webpack/Rollup and map them once in your root import map.
* **Module Fed way:** Use `shared: { react: { singleton: true, requiredVersion: '^18.0.0' } }`.
* **Governance:** Enforce this via an organizational ESLint plugin or shared Webpack/Rspack preset that errors if a team accidentally bundles React into their MFE bundle.

### 2. Import Map / Manifest Versioning Strategies
When a team updates an MFE, how does the browser know to fetch the new version?
* **Cache Busting:** Never cache the import map itself (`Cache-Control: no-cache`), but heavily cache the specific bundle URLs (e.g., `mfe-a.v1.2.3.hash.js`).
* **Manifest Generation:** Do not edit import maps manually in production. Use a CI/CD pipeline where MFE builds write their output URL to a database/API, and the root-config server dynamically generates the `importmap` JSON on request. If you're on Module Federation 2.0, this is largely what its **Manifest** feature already gives you out of the box, rather than a hand-rolled service.
* **Managed deployment platforms:** by 2026, tools like **Zephyr Cloud** have emerged specifically to automate MF remote versioning, rollback, and CDN publishing — worth evaluating before building a bespoke manifest service in-house, especially past a few dozen remotes.
* **Rolling Back:** Because bundles are immutable and cached, a rollback is simply reverting the import map/manifest entry to the previous hash — zero deployment needed.

### 3. Shared State & Cross-MFE Communication
Avoid a shared global store (like a monolithic Redux) at all costs. It creates invisible coupling.
* **Custom Events:** The browser's native `CustomEvent` API is highly underrated. MFE-A dispatches `new CustomEvent('user:logged-in', { detail: user })`, MFE-B listens for it. Zero coupling.
* **Single-SPA Parcel Bus:** If using Single-SPA parcels, leverage the built-in event bus, but keep payloads minimal and abstract.
* **Shared API Layer / BFF:** If multiple MFEs need the same data (e.g., user profile), fetch it once in the root-config and pass it down via custom props during the `mount` lifecycle, rather than having each MFE fetch it independently.

---

## 🔬 Testing & Observability at Scale

Past a handful of MFEs, "does it work in my local dev environment" stops being a meaningful signal. Two things become load-bearing:

### Contract Testing
Each MFE exposes a contract (props it accepts, custom events it emits/listens for, its lifecycle export shape). Treat this like an API contract:
* Use consumer-driven contract testing (e.g., Pact-style) between the shell and each remote so a remote team can verify they haven't broken the shell's expectations **before** deploying, without needing the whole integrated system running in CI.
* For visual/DOM-shape regressions, per-MFE Cypress/Playwright component tests catch far more than full end-to-end runs, and run drastically faster in CI.

### Observability Per Fragment
A single unhandled error in one MFE can take down the whole page if you don't isolate blast radius:
* **Per-MFE error boundaries** — in Single-SPA, wrap each parcel's mount in a try/catch that reports and gracefully unmounts just that fragment, rather than letting an error propagate to the shell.
* **Distributed tracing** — tag outgoing requests from each MFE with an org-wide trace/span ID so a slow page load can be attributed to the specific remote (and even the specific team) responsible, not just "the frontend."
* **Version tagging in RUM (Real User Monitoring):** since remotes deploy independently, your monitoring needs to know *which version of which remote* was active for a given session — otherwise a regression report is nearly impossible to bisect.

---

## 🔒 Security Considerations for MFEs

Distributing ownership doesn't remove the security surface — it just spreads it across teams who may not all be thinking about it.

* **Content-Security-Policy with multiple origins:** if remotes are served from different subdomains/CDNs, your CSP `script-src`/`connect-src` needs to allow all of them explicitly — a common source of "works locally, CSP-blocked in prod" incidents when a new remote's origin isn't added.
* **Subresource Integrity (SRI):** for remote entry files (`remoteEntry.js`, manifest JSON) served from a CDN, SRI hashes protect against a compromised CDN silently serving altered code — worth the operational overhead for anything handling auth or payments.
* **Sandboxing trade-off:** Runtime Integration (Section 3) shares a JS realm across all MFEs by design — a malicious or compromised remote has full access to the page, cookies, and other MFEs' state. If you're integrating a genuinely untrusted or third-party fragment, the iframe pattern (Section 4) is the only one of the four that actually sandboxes it.
* **Governance for singleton deps:** a shared React singleton (Section 8, point 1) means one MFE pinning a vulnerable version affects everyone — dependency-version governance is a security control here, not just a stability one.

---

## 🚫 When *Not* to Use Micro-Frontends

Worth stating plainly, since most guides only sell the pattern: teams that have run MFEs in production at multiple clients report ripping the architecture back out entirely and replacing it with a monolith in at least one case. Reasons that recur:

* **Fewer than ~3–4 independently-deployed teams.** The coordination overhead of any of the four models outweighs the deploy-independence benefit below this scale — a well-organized monolith with clear module boundaries gets you most of the win with none of the runtime complexity.
* **No real organizational boundary to justify it.** If "the frontend team" is really one team splitting work by feature rather than genuinely separate teams with separate release cadences, MFEs solve a problem you don't have.
* **SEO-critical + tight performance budgets + small team.** Server Composition or Runtime Integration both add real complexity for a benefit (independent deploys) that a small team won't cash in on.
* **The org can't commit to shared-dependency governance.** Every runtime/build-time model above depends on *someone* enforcing singleton versions, CSS isolation conventions, and contract stability. Without that governance function staffed, MFEs degrade into the "1990s script-tag soup" failure mode faster than a monolith degrades into spaghetti.

---

## 📚 Deep-Dive Reference Index

*Curated primary sources for deep reading:*
* [Module Federation 2.0 — official docs](https://module-federation.io/)
* [InfoQ — Module Federation 2.0 Reaches Stable Release](https://www.infoq.com/news/2026/04/module-federation-2-stable/)
* [Native Federation (`@softarc/native-federation`)](https://www.npmjs.com/package/%40softarc/native-federation)
* [Angular Blog — Micro Frontends with Angular and Native Federation](https://blog.angular.dev/micro-frontends-with-angular-and-native-federation-7623cfc5f413)
* [Single-SPA — official docs](https://single-spa.js.org/)
* [Podium — NRK's server-side composition framework](https://podium-lib.io/)
* [MDN — Import Maps](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/script/type/importmap)
* [MDN — Subresource Integrity](https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity)

---

**⭐ Star this repository if it helped you architect (or rescue) a micro-frontend rollout!**
