# nitro-only preview 404 repro

Framework-free reproduction of the nitro preview bug that surfaces in TanStack Start + prerender.
Nitro + Vite, no framework plugins, no other wrappers.

## Bug

With `preset: "aws-lambda"`, `nitro preview` returns **HTTP 404** on every route. With `preset: "node-server"` (swap the line in `nitro.config.ts`), the same routes return **200 OK**.

Root cause: aws-lambda's runtime exports only `{ handler }`. srvx's `loadServerEntry` fetch-extraction cascade (`src/loader.ts:163-178`) has no branch that matches `{ handler }`, so `entry.fetch` returns `undefined`. `nitro/src/preview.ts:74-75` keeps its default `() => new Response("Not Found", { status: 404 })` stub, and serves it for every request.

## Reproduce

```bash
cd nitro-only
npm install
npx nitro build
npx nitro preview &
sleep 1
curl -i http://localhost:3000/hello
# HTTP/1.1 404
# content-type: text/plain;charset=UTF-8
# Not Found

kill %1
```

Change `preset: "aws-lambda"` to `preset: "node-server"` in `nitro.config.ts`, rebuild, and the same curl returns `200 OK` with body `hello`.

## Versions

Pinned in `package.json` at the nitro beta where the bug was first investigated; also reproduces on `nitro@nightly` at audit time.

## Fix in flight

[nitrojs/nitro#4052](https://github.com/nitrojs/nitro/pull/4052) adds `export default { fetch: nitroApp.fetch }` to the aws-lambda runtime, which triggers srvx's `mod.default.fetch` branch.
