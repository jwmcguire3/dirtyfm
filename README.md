# DirtyFM

DirtyFM is a raw internet-radio, video, and Dirty News archive for Drift. The public site presents the dirty signal, video archive, Dirty News posts, individual post/comment pages, and public intake forms. The protected `/signal-control` area is the operator console for managing incoming signals, pending Dirty News submissions, posts, videos, homepage copy, and comments.

Before changing UI, routes, public copy, or visual components, read `docs/brand.md`. It is the source of truth for the DirtyFM voice and visual target: pirate radio, corrupted broadcast, punk flyer, late-night VHS, vandalized dossier, raw but readable.

## Current State

DirtyFM is a Next.js App Router project using React, TypeScript, and Tailwind CSS. The active production shape is:

- Hosting: Vercel
- Source/deploy workflow: GitHub connected to Vercel
- Auth mode: local Signal Control auth
- Content backend: Cloudflare KV
- Vercel KV access path: Cloudflare KV REST API
- Supabase: retained as optional future backend/auth code, but inactive unless explicitly selected by env mode
- Cloudflare/OpenNext: experimental support remains through `wrangler.jsonc`

The repo now has package-level `test` and `check` scripts. The KV content store has been split into domain modules while preserving `src/lib/kv/contentStore.ts` as the compatibility export surface. Signal Control has also been split so the route file mostly handles auth/data orchestration and composes admin components.

## What The Site Does

- `/` serves the main DirtyFM signal page.
- `/dirty-tv` serves current Dirty TV videos plus archived Dirty TV posts/videos.
- `/dirty-news` serves published Dirty News posts.
- `/dirty-news/[slug]` serves individual posts and visible comments.
- `/contact` lets public users send contact/intake signals.
- `/signal-control` is the protected admin console.

Public users can submit contact messages and Dirty News submissions, but they cannot directly publish posts. User-submitted content must never be rendered as trusted raw HTML.

## Production Environment

Production is hosted on Vercel and should use explicit mode values:

```text
DIRTYFM_AUTH_MODE=local
DIRTYFM_CONTENT_BACKEND=cloudflare-kv
```

Required local-auth variables:

```text
DIRTYFM_OPERATOR_EMAIL
DIRTYFM_OPERATOR_PASSPHRASE_HASH
DIRTYFM_SESSION_SECRET
```

Required Cloudflare KV REST variables on Vercel:

```text
CLOUDFLARE_ACCOUNT_ID
CLOUDFLARE_KV_NAMESPACE_ID
CLOUDFLARE_API_TOKEN
```

Supabase variables may exist in Vercel or local `.env` files for future work, but production should not use Supabase while `DIRTYFM_AUTH_MODE=local` and `DIRTYFM_CONTENT_BACKEND=cloudflare-kv`.

## Runtime Mode Selection

Runtime config validation lives in `src/lib/runtimeConfig.ts` and `src/lib/runtimeConfigCore.ts`.

Auth mode selection:

- `DIRTYFM_AUTH_MODE=local` uses local Signal Control auth.
- `DIRTYFM_AUTH_MODE=supabase` uses the retained Supabase auth/admin path.

Content backend selection:

- `DIRTYFM_CONTENT_BACKEND=cloudflare-kv` uses `src/lib/kv/*`.
- `DIRTYFM_CONTENT_BACKEND=supabase` uses retained Supabase data access code in `src/lib/db/*`.

Production/Vercel runtimes require explicit mode values. Missing or invalid production mode values should fail clearly instead of silently falling into the wrong backend/auth path.

## Cloudflare KV Notes

The active Vercel path uses Cloudflare KV through REST credentials:

```text
CLOUDFLARE_ACCOUNT_ID
CLOUDFLARE_KV_NAMESPACE_ID
CLOUDFLARE_API_TOKEN
```

The repo also includes `wrangler.jsonc` for Cloudflare/OpenNext deployment experiments. That file defines a `DIRTYFM_CONTENT` KV binding for Workers-style deployments. On Vercel there is no Workers KV binding, so the app uses the REST API path instead.

KV key names and index names are centralized under `src/lib/kv/keys.ts`. Do not change them casually; that can orphan existing production content.

## Main Code Map

```text
src/app/                         Next.js App Router routes
src/app/signal-control/actions.ts Protected admin server actions
src/components/dirty/             Public DirtyFM UI components
src/components/admin/             Admin and Signal Control UI components
src/data/                         Static/archived seed content
src/lib/authMode.ts               Auth mode entry point
src/lib/contentBackend.ts         Content backend entry point
src/lib/runtimeConfig*.ts         Runtime mode parsing and validation
src/lib/db/                       Supabase-capable data/auth paths
src/lib/kv/                       Active Cloudflare KV content backend
```

Important current KV modules:

```text
src/lib/kv/contentStore.ts         Compatibility barrel for KV exports
src/lib/kv/keys.ts                 KV keys, indexes, ID helpers
src/lib/kv/store.ts                KV namespace adapter: binding, REST, memory fallback
src/lib/kv/homeSettings.ts         Homepage copy settings
src/lib/kv/posts.ts                Dirty News posts and publishing workflow
src/lib/kv/videos.ts               Dirty TV video records
src/lib/kv/comments.ts             Comments and comment moderation helpers
src/lib/kv/contactSubmissions.ts   Contact/intake submissions
src/lib/kv/postSubmissions.ts      Dirty News submission queue
src/lib/kv/dashboard.ts            Signal Control dashboard aggregation
```

Important Signal Control components:

```text
src/components/admin/signal-control/CommentList.tsx
src/components/admin/signal-control/ContactList.tsx
src/components/admin/signal-control/EmptyWire.tsx
src/components/admin/signal-control/HomeSettingsPanel.tsx
src/components/admin/signal-control/PostEditorList.tsx
src/components/admin/signal-control/PostList.tsx
src/components/admin/signal-control/SignalControlLoginPanel.tsx
src/components/admin/signal-control/SignalControlStats.tsx
src/components/admin/signal-control/SubmissionList.tsx
src/components/admin/signal-control/VideoEditorList.tsx
```

## Local Development

Install dependencies:

```bash
npm install
```

Run the dev server:

```bash
npm run dev
```

Build locally:

```bash
npm run build
```

Start a built app:

```bash
npm run start
```

Lint:

```bash
npm run lint
```

Typecheck:

```bash
npm run typecheck
```

Run tests:

```bash
npm run test
```

Run the full completion gate:

```bash
npm run check
```

Current `npm run check` runs lint, typecheck, and tests.

The test script uses Node's built-in test runner with `--experimental-test-isolation=none` because the existing `.mjs` tests import TypeScript files directly and run reliably in one process.

## Environment Setup

Copy `.env.example` for local setup and fill in only what the selected mode needs.

For the current production-like local shape:

```text
DIRTYFM_AUTH_MODE=local
DIRTYFM_CONTENT_BACKEND=cloudflare-kv
```

Then provide local operator credentials and Cloudflare KV REST credentials. Leave Supabase values blank unless intentionally testing the Supabase path.

Do not commit secrets. Keep real values in local env files or Vercel environment variables.

## Deployment Flow

Normal flow:

1. Make changes locally.
2. Run `npm run check`.
3. Commit to GitHub.
4. Let Vercel build and deploy from the connected GitHub branch/project.

Cloudflare/OpenNext deployment remains experimental unless the deployment target is intentionally changed.

## Security Rules

- Do not expose secrets to the client.
- Do not put private values in `NEXT_PUBLIC_*`.
- Do not rely on an unlisted admin URL as security.
- Do not implement admin mutations without authentication and authorization.
- Public users must never publish posts directly.
- Never render user-submitted raw HTML.
- Comments can be visible by default, but admin tools must allow hiding/deleting them.
- Keep Cloudflare API tokens server-only.
- Keep local operator passphrase hash and session secret server-only.

## Brand Rules

DirtyFM should feel like pirate radio, corrupted broadcast, punk flyer, late-night VHS, and a vandalized dossier. Keep it loud and usable.

Do not make it clean, corporate, generic, SaaS-like, podcast-bro, campaign-like, or polished startup dark mode.

For brand decisions, page copy, visual component changes, and UI language, read `docs/brand.md` first.

## Contributor Notes

- Prefer the Cloudflare KV/local-auth path unless the task explicitly targets Supabase.
- Keep Supabase code available unless a future migration intentionally removes it.
- Preserve archived Dirty News text, dates, bylines, images, comments, and legacy metadata exactly unless a task explicitly says otherwise.
- Preserve current public routes.
- Preserve KV key names.
- Prefer small, focused changes with `npm run check` passing before completion.
