# GitHub Pages Hosting — Findings

Analysis of what this Next.js project needs to build and deploy as a static site
on GitHub Pages. Based on `package.json`, `next.config.ts`, and
`.github/workflows/nextjs.yml` as of 2026-06-12.

## Current State

- **Framework:** Next.js 15.1.7 (App Router, React 19)
- **Deploy workflow:** `.github/workflows/nextjs.yml` — builds on push to `main`,
  uploads `./out` via `actions/upload-pages-artifact@v3`, deploys with
  `actions/deploy-pages@v4`
- **Config:** `next.config.ts` currently only sets `images.domains` and has
  none of the static-export settings GitHub Pages requires

## Package Manager

Both `yarn.lock` and `package-lock.json` exist in the repo root. The workflow's
detect step checks for `yarn.lock` first, so **CI uses yarn**.

> ⚠️ Hazard: dual lockfiles can drift apart, making local `npm install` and CI
> `yarn install` resolve different dependency versions. Pick one lockfile and
> delete the other.

## What the Workflow Already Handles

The `actions/configure-pages@v5` step runs with `static_site_generator: next`,
which at CI time automatically:

- Injects `basePath` into the Next.js config, derived from the repository name
- Sets `images.unoptimized: true` (no image optimization server exists on Pages)

It does **not** inject `output: 'export'`.

## What Is Missing in `next.config.ts`

The build as configured will fail to produce the `./out` artifact, because in
Next.js 15 the `out` directory is only generated with `output: 'export'`
(the standalone `next export` command was removed). Required changes:

1. **`output: 'export'`** — mandatory; must be added manually. Without it,
   `next build` produces `.next/` only and the artifact upload step fails.

2. **`basePath: '/CookIEs'`** (and typically `assetPrefix`) — for a project
   site hosted at `https://<user>.github.io/CookIEs/`. In CI, configure-pages
   injects this automatically, but hardcoding it (or gating on an env var such
   as `process.env.GITHUB_ACTIONS`) keeps local builds consistent with the
   deployed site.

3. **`images.unoptimized: true`** — the current `images.domains` setting relies
   on the Next.js image optimization server, which does not exist in a static
   export. configure-pages injects `unoptimized` in CI, but the `domains` key
   should be removed regardless (it is also deprecated in favor of
   `remotePatterns`).

4. **`trailingSlash: true`** (optional) — emits `page/index.html` instead of
   `page.html`, giving cleaner static routing on Pages.

### Example config

```ts
import type { NextConfig } from "next";

const isGithubActions = process.env.GITHUB_ACTIONS === "true";

const nextConfig: NextConfig = {
  output: "export",
  basePath: isGithubActions ? "/CookIEs" : "",
  assetPrefix: isGithubActions ? "/CookIEs/" : "",
  trailingSlash: true,
  images: {
    unoptimized: true,
  },
};

export default nextConfig;
```

Note: the existing `next.config.ts` is written in CommonJS style
(`module.exports` with a JSDoc type annotation). Any rewrite should use
`import type { NextConfig }` + `export default` as above.

## Static-Export Constraints to Keep in Mind

`output: 'export'` rules out server-side features. This project is currently
compatible — all pages are static, and locale handling
(`src/contexts/TranslationContext.tsx`) runs entirely client-side via
`localStorage` and `navigator.languages` — but future work must avoid:

- Server Actions, Route Handlers with dynamic behavior, middleware
- `next/image` optimization (must stay `unoptimized`)
- Dynamic routes without `generateStaticParams`
- Rewrites/redirects/headers in the Next config (not applied by Pages)

Also note that GitHub Pages serves a `404.html` for unknown paths; Next.js
exports one automatically from the App Router `not-found` convention if defined.
