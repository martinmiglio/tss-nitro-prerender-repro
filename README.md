# tss-nitro-prerender-repro

Minimal reproduction: **`@tanstack/react-start`'s `prerender` feature breaks on several non-Node `nitro` presets**.

[![CI](https://github.com/martinmiglio/tss-nitro-prerender-repro/actions/workflows/repro.yml/badge.svg)](https://github.com/martinmiglio/tss-nitro-prerender-repro/actions/workflows/repro.yml)

Every push runs a build per preset. **Red jobs = bug reproduces.** When upstream fixes it, the red jobs flip green.

## Reproduce locally

```bash
bun install

bun run build                          # preset = default (node-server) — ✅ succeeds
NITRO_PRESET=aws-lambda bun run build  # ❌ hard-fails during prerender
NITRO_PRESET=vercel bun run build      # ⚠️  prerenders, then build process never exits
NITRO_PRESET=netlify bun run build     # ⚠️  same post-success hang as vercel
```

## Versions (repro lockfile)

- `@tanstack/react-start` `^1.167.42`
- `nitro` `^3.0.260311-beta`
- `vite` `^8.0.8`
- `@vitejs/plugin-react` `^6.0.1`
- `react` / `react-dom` `^19.2.3`
- Node 20 (CI) / 24 (local) — both affected

The bug also reproduces in the official [`examples/react/start-basic`](https://github.com/TanStack/router/tree/main/examples/react/start-basic) example from `TanStack/router@main` when you add `prerender: { enabled: true, crawlLinks: true }` to `tanstackStart(...)` and `preset: 'aws-lambda'` (or `vercel`, `netlify`) to `nitro(...)` — see "Also reproduces upstream" below.

## Two distinct failure modes

### Mode A — hard prerender error (aws-lambda)

```
[prerender] Crawling: /
[prerender] Prerendered 0 pages:
Error: Failed to fetch /:
  at .../@tanstack/start-plugin-core/dist/esm/prerender.js
error: script "build" exited with code 1
```

On older `nitro@3.0.1-alpha.1` the same preset surfaces as:

```
Error [ERR_MODULE_NOT_FOUND]: Cannot find module '<cwd>/dist/server/server.js'
  imported from .../@tanstack/start-plugin-core/.../preview-server-plugin/plugin.js
```

Same root cause, different control-flow path.

### Mode B — build never exits (vercel, netlify)

```
[prerender] Prerendered 1 pages:
[prerender] - /
# ... and then the process hangs forever.
```

The prerender crawl actually succeeds, but `vite build` never terminates after the prerender for these presets. CI times the step out and reports failure.

## Root cause (code-level)

When `prerender.enabled: true`, TSS spins up a Vite preview server to crawl the built app (`start-plugin-core/dist/esm/vite/prerender.js`, calling `vite.preview(...)`).

Two middlewares register `configurePreviewServer` on that server:

- **`nitro:preview`** (`nitro/dist/vite.mjs:461-489`) — registers unconditionally regardless of preset. Calls `startPreview({ rootDir, loader: { nodeServer } })` and delegates via `srvx/node` to `preview.fetch(nodeReq)`.
- **`tanstack-start-core:preview-server`** (`@tanstack/start-plugin-core/dist/esm/vite/preview-server-plugin/plugin.js`) — also registers unconditionally (`order: "post"`). Imports `join(getServerOutputDirectory(server.config), 'server.js')` and expects `.default` to be an object with a `.fetch(req)` method. `getServerOutputDirectory` defaults to `dist/server` — a path nitro never writes to.

Both plugins try to serve requests; neither knows about the other; **neither is aware of the nitro preset**. For non-node presets, nitro emits a preset-specific entry (a Lambda handler export, a Vercel function, etc.) that is **not** a `{ default: { fetch } }` HTTP server, so:

- TSS's middleware hits `ERR_MODULE_NOT_FOUND` (nitro's outDir override is inactive during preview because `nitro:env` has `apply: !configEnv.isPreview` at `vite.mjs:538`) **or** imports a module whose `.default.fetch` isn't callable.
- Nitro's `startPreview` path returns an empty response because srvx can't adapt the preset's handler back to Node req/res.

The separate post-success hang on vercel/netlify is likely a child process spawned by `startPreview` that isn't being torn down after prerender finishes.

## Proposed fix shape

A real fix needs coordination between **both** plugins:

1. Nitro's `nitro:preview` `configurePreviewServer` should gate on preset — skip or error for non-node presets where `vite preview` is meaningless — **or** `startPreview` should refuse non-node outputs explicitly.
2. TSS's `previewServerPlugin` should detect when nitro owns the preview server (nitro's handler is already installed) and not register a competing middleware, **or** detect the artifact's export shape and skip gracefully.
3. Ideally TSS's prerender should not spawn a `vite preview` when the selected nitro preset can't be previewed — instead, either use nitro's native prerender or boot a temporary `node-server` preview build purely for the crawl.
4. Whatever owns the preview server must tear down cleanly after prerender so `vite build` exits (the vercel/netlify hang).

## Also reproduces upstream

The same bug reproduces in `TanStack/router@main` `examples/react/start-basic` when you change `vite.config.ts` to:

```ts
tanstackStart({
  srcDirectory: 'src',
  prerender: { enabled: true, crawlLinks: true },
}),
viteReact(),
nitro({ preset: 'aws-lambda' }), // or 'vercel', 'netlify'
```

Without the `preset` argument, the same config builds and prerenders all 25 routes successfully.

## Related issues

- [TanStack/router#6562](https://github.com/TanStack/router/issues/6562) — `preset: 'vercel'` + prerender. Filed 2026-01-31, no maintainer response. This repro generalizes that report: the bug also hits `aws-lambda` and `netlify`, with a distinct post-success hang mode on vercel/netlify and a hard prerender error on aws-lambda.
- [TanStack/router#5967](https://github.com/TanStack/router/issues/5967) (closed) — earlier SPA + prerender breakage, fixed by PR [#6256](https://github.com/TanStack/router/pull/6256). That fix did **not** address the preset case.
- [nitrojs/nitro#3905](https://github.com/nitrojs/nitro/issues/3905) (closed as "incomplete diagnosis") — the `ERR_MODULE_NOT_FOUND dist/server/server.js` shape.

## What's in this repo

Only what's needed to reproduce — no styling, no data loaders, no extra routes:

- `vite.config.ts` — `tanstackStart({ prerender: true }) + viteReact() + nitro({ preset: process.env.NITRO_PRESET })`
- `src/routes/__root.tsx` — minimal root document
- `src/routes/index.tsx` — one page
- `src/router.tsx` — `createRouter({ routeTree })`
- `.github/workflows/repro.yml` — CI matrix across presets (red jobs = bug present)

## Presets intentionally excluded

The `cloudflare` and `static` presets also fail on this minimal app, but with **unrelated** errors (`Nitro entry is missing` and `rollupOptions.input should not be an html file`). Those are preset-setup bugs, not the prerender interaction bug, so they're excluded from the matrix to keep the signal clean.
