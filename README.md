# tss-nitro-prerender-repro

Minimal reproduction: **`@tanstack/react-start`'s `prerender` feature is broken on every non-Node nitro preset** (aws-lambda, vercel, netlify, cloudflare, static). It only works on the default `node-server` preset.

## Reproduce

```bash
bun install
bun run build                          # preset = default (node-server) — succeeds
NITRO_PRESET=aws-lambda bun run build  # fails
NITRO_PRESET=vercel bun run build      # fails
NITRO_PRESET=netlify bun run build     # fails
NITRO_PRESET=cloudflare bun run build  # fails
NITRO_PRESET=static bun run build      # fails (nitro runs its own prerender, TSS prerender sees 0 routes)
```

CI runs the full matrix on every push — see `.github/workflows/repro.yml` and the latest workflow run for real logs per preset.

## Versions

- `@tanstack/react-start` `^1.167.42`
- `nitro` `^3.0.260311-beta`
- `vite` `^8.0.8`
- `@vitejs/plugin-react` `^6.0.1`
- `react` / `react-dom` `^19.2.3`
- Node 24.x

Also reproduces against the **official `examples/react/start-basic`** example from `TanStack/router@main` with prerender + non-node preset added — see "Also reproduces upstream" below.

## Failure matrix (observed)

| Preset | Outcome | Symptom |
|---|---|---|
| (default) node-server | ✅ success | all routes prerender |
| aws-lambda | ❌ fail | `Error: Failed to fetch /:` (empty status text) |
| vercel | ❌ fail | prerender fetch **hangs indefinitely** (preview server accepts connection, never responds) |
| netlify | ❌ fail | prerender fetch **hangs indefinitely** (same as vercel) |
| cloudflare | ❌ fail | build errors out (preset-specific module resolution) |
| static | ❌ fail | `[prerender] Prerendered 0 pages` (nitro's own prerender runs, TSS's is a no-op and reports zero) |

(Two of the failure modes — vercel and netlify — hang forever rather than fail fast, so CI enforces a per-job timeout.)

With older `nitro@3.0.1-alpha.1` + aws-lambda, the error surfaces instead as
`ERR_MODULE_NOT_FOUND: <cwd>/dist/server/server.js` from
`@tanstack/start-plugin-core/dist/esm/vite/preview-server-plugin/plugin.js`.
That's the same bug via a different control-flow path — see "Root cause" below.

## Root cause (code-level)

When `prerender.enabled: true`, TSS spins up a Vite preview server to crawl the built app (`start-plugin-core/dist/esm/vite/prerender.js:~32`, calling `vite.preview(...)`).

Two middlewares register `configurePreviewServer` on that server:

- **`nitro:preview`** (`nitro/dist/vite.mjs:461-489`) — registers unconditionally regardless of preset. Calls `startPreview({ rootDir, loader: { nodeServer } })` and delegates requests via `srvx/node` to `preview.fetch(nodeReq)`.
- **`tanstack-start-core:preview-server`** (`@tanstack/start-plugin-core/dist/esm/vite/preview-server-plugin/plugin.js`) — also registers unconditionally. Imports `join(getServerOutputDirectory(server.config), 'server.js')` and expects `.default` to be an object with a `.fetch(req)` method. `getServerOutputDirectory` defaults to `dist/server` — a path nitro never writes to.

Both plugins try to serve requests, neither knows about the other, and **neither is aware of the nitro preset**. For non-node presets, nitro emits a preset-specific entry (a Lambda handler export, a Vercel function, etc.) that is **not** a `{default: {fetch}}` HTTP server, so:

- TSS's middleware either hits `ERR_MODULE_NOT_FOUND` (when nitro's outDir override is inactive during preview because `nitro:env` has `apply: !configEnv.isPreview` at `vite.mjs:538`) or imports a module whose `.default.fetch` isn't callable.
- Nitro's `startPreview` path returns an empty response because srvx can't adapt the preset's handler back to Node req/res.

Either way: the prerender crawler gets a bad response and the build fails.

## Proposed fix shape

A real fix needs coordination between **both** plugins:

1. Nitro's `nitro:preview` `configurePreviewServer` should gate on preset (skip or error for non-node presets where `vite preview` is meaningless), **or** `startPreview` should refuse non-node outputs explicitly.
2. TSS's `previewServerPlugin` should detect when nitro is the owner of the preview server (nitro's handler is already installed) and not register a competing middleware, **or** detect the artifact's export shape and skip gracefully.
3. Ideally TSS's prerender should not spawn its own `vite preview` when the selected nitro preset can't be previewed — instead, either use nitro's native prerender (the `static` preset path) or boot a temporary `node-server` preview build purely for the crawl.

## Also reproduces upstream

The same bug reproduces in `TanStack/router@main` `examples/react/start-basic` when you change `vite.config.ts` to:

```ts
tanstackStart({
  srcDirectory: 'src',
  prerender: { enabled: true, crawlLinks: true },
}),
viteReact(),
nitro({ preset: 'aws-lambda' }), // or 'vercel', 'netlify', etc.
```

Without the `preset` argument, the same config builds and prerenders all 25 routes successfully.

## Related issues

- [TanStack/router#6562](https://github.com/TanStack/router/issues/6562) — `preset: 'vercel'` + prerender. Filed 2026-01-31, no maintainer response. This repro generalizes that report across all non-node presets.
- [TanStack/router#5967](https://github.com/TanStack/router/issues/5967) (closed) — earlier SPA + prerender breakage, fixed by PR [#6256](https://github.com/TanStack/router/pull/6256). That fix did **not** address the preset case.
- [nitrojs/nitro#3905](https://github.com/nitrojs/nitro/issues/3905) (closed as "incomplete diagnosis") — the `ERR_MODULE_NOT_FOUND dist/server/server.js` shape.

## What's in this repo

Only what's needed to reproduce — no styling, no data loaders, no extra routes:

- `vite.config.ts` — `tanstackStart({ prerender: true }) + viteReact() + nitro({ preset: process.env.NITRO_PRESET })`
- `src/routes/__root.tsx` — minimal root document
- `src/routes/index.tsx` — one page
- `src/router.tsx` — `createRouter({ routeTree })`
- `.github/workflows/repro.yml` — CI matrix across presets
