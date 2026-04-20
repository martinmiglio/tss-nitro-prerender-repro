# tss-nitro-prerender-repro

Minimal reproduction of a `@tanstack/react-start` + `nitro` interaction bug where
**`vite build` fails during prerender** with:

```
Error [ERR_MODULE_NOT_FOUND]: Cannot find module '<cwd>/dist/server/server.js'
  imported from .../@tanstack/start-plugin-core/dist/esm/vite/preview-server-plugin/plugin.js
```

## Reproduce

```bash
bun install
bun run build
```

CI runs this on every push — see `.github/workflows/repro.yml` and the latest run for a real log.

## Versions

- `@tanstack/react-start` 1.167.5
- `@tanstack/react-router` 1.167.5
- `@tanstack/router-plugin` 1.167.5
- `nitro` 3.0.1-alpha.1
- `vite` ^8.0.8
- `@vitejs/plugin-react` ^6.0.1
- `react` / `react-dom` ^19.2.3
- Node 24.x

## What the log shows

1. Client build succeeds (writes to `dist/client/`).
2. SSR build succeeds, but writes to **`node_modules/.nitro/vite/services/ssr/server.js`** (nitro's per-service outDir), not `dist/server/`.
3. TSS's prerender step spins up an internal `vite preview` server whose middleware tries to `import(join(getServerOutputDirectory(config), 'server.js'))`.
4. `getServerOutputDirectory` reads `config.environments.ssr.build.outDir`, which in preview mode falls back to the default `dist/server` — because nitro's `nitro:env` plugin (which sets that outDir) is gated with `apply: (_, configEnv) => !configEnv.isPreview`.
5. Nothing was ever built at `dist/server/server.js`, so the import fails.

## Pointers in the source

- `node_modules/nitro/dist/vite.mjs` — `apply: (_config, configEnv) => !configEnv.isPreview` on the env plugin.
- `node_modules/@tanstack/start-plugin-core/dist/esm/vite/preview-server-plugin/plugin.js` — the `configurePreviewServer` hook that imports the server bundle.
- `node_modules/@tanstack/start-plugin-core/dist/esm/vite/output-directory.js` — `getServerOutputDirectory` with its `dist/server` fallback.

## Why this repo exists

Filed to help TanStack maintainers narrow the fix. See related upstream issues: #5967, #6275, #6562, #6587.
