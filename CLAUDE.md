# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev      # vite dev server (local worker + react-app)
npm run build    # tsc -b && vite build
npm test         # vitest run (test/index.test.ts, via @cloudflare/vitest-pool-workers)
npm run check    # tsc -b && vite build && wrangler deploy --dry-run
npm run deploy   # npm run build && wrangler deploy
npm run cf-typegen  # regenerate worker-configuration.d.ts after editing wrangler.jsonc
```

Run a single test with vitest's `-t` filter, e.g. `npx vitest run -t "serves /llms.txt"`.

Tests exercise the real Worker via `SELF.fetch` under Miniflare (config: `vitest.config.ts`, `wrangler.jsonc` bindings). The admin-write tests need `ADMIN_TOKEN`, which is injected as a test-only Miniflare binding (`test-token`) in `vitest.config.ts` — don't confuse that with the real `wrangler secret put ADMIN_TOKEN` used in production.

## Architecture

One enriched content store (`Resource[]`) projected onto many agent-discovery surfaces — llms.txt, a typed JSON index, per-page Markdown, robots.txt, Content-Signal headers, JSON-LD, and an optional Web Bot Auth identity surface. All surfaces are just different renderings of the same underlying data.

```
src/worker/index.ts        Hono app — routes for every surface + the JSON API
src/enrichment/index.ts    Workers AI enrichment: raw page -> structured Resource
src/enrichment/surfaces.ts Pure render functions, one per surface (no I/O)
src/lib/store.ts           KV-backed enriched store (get / upsert / clear)
src/lib/content.ts         Sample content (zero-config demo data)
src/lib/types.ts           Shared types (Resource, RawResource, Env, SiteConfig)
src/lib/web-bot-auth.ts    Optional agent-identity module, gated by ENABLE_WEB_BOT_AUTH
src/react-app/             Surface-explorer UI (bundled to dist/client, served as static assets)
test/index.test.ts         Worker tests, hit through SELF.fetch
```

Request flow: a page's raw content is enriched once by Workers AI (title, summary, key points, topic tags) and cached in KV (`VISIBILITY_CACHE`, TTL via `ENRICHMENT_CACHE_TTL`). `enrichResource` never hard-fails — on any model/parse error it falls back to `fallbackEnrichment` so surfaces always render something. Every surface route in `src/worker/index.ts` reads from that same store and calls a pure renderer in `surfaces.ts`.

`wrangler.jsonc` routes specific paths (`/.well-known/api-catalog`, `/auth.md`, `/auth.txt`, `/llms.txt`, `/llms-full.txt`, `/index.json`, `/robots.txt`, `/jsonld`, `/api*`) through `run_worker_first` so the Worker intercepts them ahead of the static `dist/client` assets (which handle the React UI as an SPA fallback).

Writes (`POST /api/resources`, refresh endpoints) are disabled until an `ADMIN_TOKEN` secret is set (`wrangler secret put ADMIN_TOKEN`); unauthenticated requests to these routes are rejected.

### Conventions

- Keep `surfaces.ts` pure — render functions take a `RenderCtx` and return strings/objects, no I/O. This is what makes surfaces easy to test and to add to.
- Web Bot Auth is about agent *identity*, not content *readability* — keep it in its own module (`web-bot-auth.ts`), gated by `ENABLE_WEB_BOT_AUTH`, and don't wire it into the core surfaces.
- To add a new surface: write a pure renderer in `surfaces.ts`, add a route in `worker/index.ts` (send the `Content-Signal` header for text/JSON surfaces, add `cors()` if agents fetch it cross-origin), list it in the `/api/site` `surfaces` array so the UI shows it, and add a test in `test/index.test.ts`.
- After editing `wrangler.jsonc`, rerun `npx wrangler types` (`npm run cf-typegen`).

This is a Cloudflare Workers template (Workers + Workers AI + KV), deployed with `wrangler deploy`.
