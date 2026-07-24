# Frontend Architecture & Implementation Plan — Portfolio Platform (v2, post-ADR)

> Design + phased delivery plan for the **two frontend applications** consuming the Headless Portfolio CMS:
> **1) Public Portfolio Website** · **2) Admin Dashboard (CMS)**.
> Stack: Next.js (App Router) · TypeScript · Tailwind CSS v4 · shadcn/ui · Framer Motion (`motion`) · TanStack Query · Axios · React Hook Form · Zod · Lucide · **TipTap** (rich text) · **dnd-kit** (page builder) · **Vitest/RTL · Playwright · Storybook** (quality).
> **No code until approved.** After approval we build **one feature at a time**.
> **v2** integrates every fix from the Frontend Architecture Design Review — see the revision log first.

---

## Revision log — ADR fixes applied (v1 → v2)

| Ref | Issue (from ADR) | Resolution applied |
|-----|------------------|--------------------|
| **C1 🔴** | No testing/CI strategy at all | Added **§16 Testing & CI**: Vitest + RTL (unit/integration), **Playwright** (e2e), **jest-axe/axe** (a11y), **Storybook** (component workbench) + Playwright visual snapshots, CI gates (typecheck/lint/test/build/bundle-budget). → FD10 |
| **C2 🟠** | Admin block/rich-text editor unspecified (the defining CMS feature) | Specified **§9b Admin Editor**: page builder = **dnd-kit** (reorder) + block registry + per-block **RHF/Zod** config; rich text = **TipTap (ProseMirror JSON)**; markdown = textarea+preview; **MediaPicker**; **autosave + live preview** (draft mode). → FD11/FD14 |
| **C3 🟠** | Monorepo over-justified & mechanically under-designed; Tailwind-v4 "preset" error | **Committed to monorepo with full mechanics**: `transpilePackages`, **`server-only`/`client-only` boundaries** on `packages/api`, Tailwind-v4 **shared `@theme` CSS + `@source`** (not a v3 preset), shadcn monorepo config. Single-app route-groups noted as the valid simpler alternative. → FD1 |
| **C4 🟠** | No CMS render resilience or security | **§9 + §17**: **per-block Zod validation** at render, **sanitized rendering** of CMS content (render from JSON, never raw HTML), **CSP/security headers**, **iframe sandbox** for embeds, **contact spam** (honeypot + Turnstile). → FD13 |
| **C5 🟡** | Freshness coupled to backend webhooks (deferred) | **§8**: interim **time-based ISR** (`revalidate`) now → swap to on-demand **`revalidateTag`** when backend revalidation webhooks land (backend Phase 9). → FD7 |
| I4 | Domain composites coupled into design-system package | **`packages/ui` = domain-agnostic**; new **`packages/blocks`** holds domain composites (ProjectCard/BlogCard) + block components + shared `BlockRenderer`. → FD12 |
| I6 | Session bootstrap missing | **§6**: silent refresh on app load restores the in-memory access token from the refresh cookie; auth-state gating. → FD5 |
| I7 | `next/image`↔Cloudinary double-optimization; routing collision | **§13** Cloudinary `next/image` **custom loader**; **§5** reserved-slug guard for `/[slug]` vs fixed routes. |
| I8 | Missing analytics wiring / spam protection | **§6/§17**: first-party analytics events (page view, outbound click, resume download) → backend `AnalyticsEvent`; contact honeypot + Turnstile. |
| I9 | Overclaim: "new sections without code"; CMS-vs-fixed route ambiguity | Clarified: reorder/instances = no code; **new block *type* = one isolated component + registry entry** (architecture unchanged). Crisp rule for CMS-composed vs fixed-template routes (§1/§5). |

---

## Design intent

Visitors must immediately read **"this engineer builds production systems,"** not "this person makes websites." Engineering-first, premium, minimal, **dark-theme-first**: typography and content lead; motion is restrained; no excessive gradients. ahsanatta.com is a **quality benchmark only** — the system below is our own.

**Our differentiators:** monospace eyebrows + numbered sections (`01 — SELECTED WORK`); monospaced metric figures (`250+ APIs`, `p95 50% ↓`); projects as **engineering case studies**; hairline borders + a very subtle blueprint texture instead of shadows/gradients; short, purposeful, reduced-motion-aware motion.

---

## Key frontend decisions (v2)

| # | Decision | Resolution | Rationale |
|---|----------|------------|-----------|
| FD1 | App topology | **Monorepo: `apps/web` + `apps/admin` + `packages/*`** (Turborepo + npm workspaces), with mechanics specified: `transpilePackages`, `server-only`/`client-only` on `packages/api`, Tailwind-v4 shared `@theme`+`@source`, shadcn monorepo config | Independent deploys + subdomains (apex, `admin.<root>`) + clean separation; ADR risk was *under-specification*, now resolved. Single-app route-groups is the valid simpler fallback |
| FD2 | Theme | **Dark-first**, light later via `next-themes` + CSS vars (Tailwind v4 `@custom-variant dark`, shared `@theme`) | Requirement; tokens theme-agnostic so light is additive |
| FD3 | Typefaces | **Geist Sans** (UI/headings) + **Geist Mono** (eyebrows, metrics, code) via `next/font` | Modern, technical, self-hosted, zero-CLS |
| FD4 | Accent | **Near-monochrome dark + ONE cool accent** (sky/cyan), sparingly; semantic colors for admin | Premium restraint; single swappable token |
| FD5 | Data transport | **Public (RSC) `fetch`** (Next cache/ISR tags) + **Admin Axios** (interceptors, refresh, `withCredentials`) + TanStack Query; **silent-refresh session bootstrap on load** | Axios in RSC forfeits Next cache; split keeps ISR public + rich client admin; both behind one typed service layer |
| FD6 | Client state | **TanStack Query** (server) · **RHF+Zod** (forms) · **URL** (list state) · **Zustand** (ephemeral admin UI) · `next-themes`; **debounced autosave + dirty-state guard** for editors | Minimal global state; CMS needs autosave |
| FD7 | Rendering | Public **SSG+ISR** (interim time-based `revalidate` → on-demand `revalidateTag` when backend webhooks land) · SSR where needed · CSR admin behind middleware | Fast/cacheable public, interactive admin, no stale content in early phases |
| FD8 | CMS rendering | **`BlockRenderer` registry** + `SectionWrapper` (visibility/order/theme/bg/spacing/animation) + **per-block Zod validation** + safe fallback | Everything dynamic, type-safe, Open/Closed, resilient to bad data |
| FD9 | Shared contracts | **`packages/types`** (DTO/response types mirror backend) + shared **Zod** schemas (forms ↔ validation) | One source of truth across both apps |
| FD10 | Quality | **Vitest + RTL** · **Playwright** (e2e + visual) · **jest-axe** · **Storybook** · CI gates + bundle budgets | Production-grade requires an automated safety net (was absent) |
| FD11 | Admin editor | Page builder = **dnd-kit** + block registry + per-block RHF/Zod config; rich text = **TipTap**; markdown = textarea+preview; **MediaPicker**; **autosave + live preview** | The editor *is* the CMS; must be a real product, not CRUD |
| FD12 | Component boundaries | **`packages/ui` = domain-agnostic** primitives/generic composites; **`packages/blocks` = domain composites + block components + `BlockRenderer`** (shared by web + admin preview) | Keeps the design system reusable; removes `ui`↔`types` coupling |
| FD13 | CMS security | Render CMS content **from JSON (not raw HTML)** + sanitize (`isomorphic-dompurify`); **CSP/security headers**; **iframe sandbox** for embeds; contact **honeypot + Turnstile** | Defense-in-depth against stored XSS/untrusted embeds |
| FD14 | Rich-text format | **TipTap / ProseMirror JSON** as the single AST end-to-end (concretizes backend D7); shared serializer/renderer in `packages/blocks` | One format editor↔storage↔renderer; no impedance mismatch |

---

## 1. Information Architecture

Content is API-driven, with **two rendering modes** and a **crisp rule for which is which**:
- **Fixed-template routes** (dedicated templates fed by *collection* APIs, stable URLs): `/projects`, `/projects/[slug]`, `/blog`, `/blog/[slug]`, `/experience`, `/skills`, `/services`, `/education`, `/certificates`, `/testimonials`, `/contact`, `404`. **About & Home** are **CMS-composed** (blocks) so they're editable without deploys.
- **CMS page routes** (`/[slug]`): a `Page` of `Sections`→`Blocks` rendered by the block registry. **Rule:** collection URLs (`projects`, `blog`, …) are **reserved**; the `/[slug]` catch-all only resolves CMS pages whose slug is not a reserved segment (validated in admin, §5).

Global chrome (header/footer nav, social, branding) comes from `Navigation` + `Settings` + `SocialLinks` APIs — nothing hardcoded.

**Admin:** Dashboard · Content (Pages, Sections) · Projects · Blogs · Media Library · Navigation · SEO · Users · Roles · Analytics · Settings — modeled on Payload/Strapi/Linear/Vercel (command palette, list→detail editors, inline publish/status, media picker, **live preview**).

---

## 2. Design System

**Color (dark-first, CSS vars in a shared `@theme`):** bg `#0A0A0B` · surface `#101012` · surface-2 `#17171A` · elevated `#1E1E22`; borders `rgba(255,255,255,.08/.12)`; text `#EDEDED`/`#A1A1A6`/`#6E6E76`; **one accent** (sky/cyan) + accent-fg; admin semantic (success/warn/danger/info). Light = same tokens, inverted (later).

**Typography (Geist Sans + Mono, ~1.2):** display clamp(2.75→3.5) · h1 clamp(2→2.5) · h2 2 · h3 1.5 · h4 1.25 · body-lg 1.125 · body 1 · sm .875 · xs .75. Headings tight-tracked 500–600; eyebrows/metrics/code in **Geist Mono**, uppercase, letter-spaced.

**Spacing** 4px base; section rhythm 96–160 / 64–96 mobile; container 1200px, 24px gutters. **Grid** 12-col mobile-first. **Radii** 4/8/12; **minimal shadows** (elevation via surface tints + hairline borders). **Motion tokens** 150/200/300ms, standard/ease-out, 8–16px offset, reduced-motion aware. **Icons** Lucide 1.5px, 16/20/24, functional.

**Tailwind v4 token sharing (ADR fix):** tokens live in a shared **CSS `@theme` file** in `packages/config` (not a v3 JS `preset`); each app imports it and adds `@source "../../packages/{ui,blocks}"` so classes in shared packages are scanned. Dark via `@custom-variant dark (&:where(.dark, .dark *))`.

---

## 3. Component Hierarchy

- **`packages/ui` (domain-agnostic):** Button, Input, Textarea, Select, Checkbox, Switch, Badge, Avatar, Tooltip, Dropdown, Dialog, Drawer/Sheet, Tabs, Card, Separator, Skeleton, Toast, Popover, Command (⌘K), Pagination, EmptyState/ErrorState/LoadingState, Prose (renders TipTap JSON), CodeBlock.
- **`packages/blocks` (domain-aware, FD12):** SectionHeading (mono eyebrow+number), MetricCard, ProjectCard, BlogCard, ServiceCard, TestimonialCard, Timeline, TechBadgeGroup, Gallery/Lightbox, **all Block components** (Heading/RichText/Markdown/Image/Gallery/Timeline/ProjectGrid/Stats/CTA/Code/Video/Quote/FAQ/Embed/Divider/Spacer), and the shared **`BlockRenderer`/`SectionWrapper`** used by both web (render) and admin (preview).
- **Layouts:** web — `SiteHeader`/`SiteFooter`; admin — `AdminShell` (sidebar + topbar + ⌘K), `DataTable`, `PageHeader`, `FormLayout`, `MediaPicker`, `PublishBar`, `BlockEditorCanvas`.

App-specific compositions live in each app's `features/`.

---

## 4. Folder Structure (monorepo, FD1)

```
portfolio-frontend/
├─ package.json (workspaces) · turbo.json · tsconfig.base.json · .storybook/
├─ apps/
│  ├─ web/    app/ · features/ · components/ · lib/ hooks/ providers/ styles/ · next.config.ts (headers/CSP, transpilePackages, image loader)
│  └─ admin/  app/ ((auth)/login · (dashboard)/* · middleware.ts) · features/ · components/ · lib/ hooks/ providers/ stores/ · next.config.ts
└─ packages/
   ├─ ui/        # design system (shadcn) — domain-agnostic
   ├─ blocks/    # domain composites + block components + BlockRenderer (+ Zod block schemas)
   ├─ api/       # api/server (fetch, server-only) · api/client (axios, client-only) · services · queries · queryKeys · ApiError
   ├─ types/     # shared DTO/response types (mirror backend) + shared Zod schemas
   ├─ config/    # @theme tokens (CSS) · tailwind base · eslint · tsconfig · motion tokens
   └─ utils/     # pure helpers (cn, format, slug, seo, cloudinary loader)
```

Every app sets `transpilePackages: ['@repo/ui','@repo/blocks','@repo/api','@repo/utils']`. `packages/api` enforces the boundary with **`server-only`** (in `api/server`) and **`client-only`** (in `api/client`) so server code can never leak into a client bundle. The current `portfolio-frontend` app becomes `apps/web` in P0.

---

## 5. Routing Structure

**web:** `/` (CMS home) · `/about` (CMS) · `/projects` · `/projects/[slug]` · `/blog` · `/blog/[slug]` · `/experience` · `/skills` · `/services` · `/education` · `/certificates` · `/testimonials` · `/contact` · `/[slug]` (CMS pages) · `not-found.tsx` · per-segment `error.tsx` + root `global-error.tsx`. `generateStaticParams` for known slugs; ISR + preview (`draftMode()`); `sitemap.ts`/`robots.ts`; `generateMetadata` per route.
**Reserved-slug guard (ADR fix):** fixed segments (`projects`,`blog`,`about`,`contact`,…) are reserved; admin blocks creating a CMS page with a reserved slug, and `/[slug]` 404s on reserved values — no silent shadowing.

**admin:** `(auth)/login` · `(dashboard)/` → dashboard, `content/pages[/id]`, `content/pages/[id]/builder`, `projects[/id]`, `blogs[/id]`, `media`, `navigation`, `seo`, `users`, `roles`, `analytics`, `settings`. `middleware.ts` gates the dashboard on refresh-cookie **presence** (opaque httpOnly → real validation via `/auth/me` on load); admin is `noindex`, CSR.

---

## 6. API Integration Strategy (FD5)

**Never call APIs from components.** Layered in `packages/api`:
- **`api/server`** (`server-only`): `serverFetch` — native `fetch` with `next:{tags,revalidate}`, base URL, envelope-unwrap, typed errors — for RSC.
- **`api/client`** (`client-only`): `axiosClient` — `baseURL`, `withCredentials`, request interceptor (attach in-memory access token), response interceptor (401 → single `/auth/refresh` + retry → else redirect), error normalize.
- **`services`**: one module per resource returning typed models (`packages/types`).
- **`queries`**: TanStack Query hooks + `queryKeys` factory, mutations with optimistic updates + `invalidateQueries`.
- **Session bootstrap (ADR fix):** on admin load, silent `POST /auth/refresh` restores the in-memory access token from the cookie; an `AuthProvider` exposes auth state and gates the shell.
- **Error handling:** normalized `ApiError` from `{ statusCode, message, error, requestId }` → toasts (admin) / error boundaries (web); 422 → RHF field errors; 403 → forbidden UI.
- **First-party analytics (ADR fix):** thin client posts `PAGE_VIEW`/`OUTBOUND_CLICK`/`RESUME_DOWNLOAD` to `POST /public/analytics/event` (batched, sendBeacon).

---

## 7. State Management Strategy (FD6/FD7)

Public = Server Components fetch via `serverFetch`; no client data-fetching for content. Admin = **TanStack Query** for all server state (cache, optimistic, invalidation) + **RHF+Zod** forms (schemas from `packages/types`) + **URL** for list state + **Zustand** for ephemeral UI (sidebar, ⌘K, unsaved-changes) + **debounced autosave** in editors. `next-themes` for theme. Skeletons + optimistic writes throughout admin.

---

## 8. Dynamic Rendering Strategy

SSG+ISR default. **Freshness (ADR fix):** ship **time-based `revalidate`** (e.g. 60–300s) now; switch to **on-demand `revalidateTag('projects'|…)`** once backend publish webhooks exist (backend Phase 9). `generateStaticParams` pre-renders known slugs; unknown → on-demand ISR. **Draft preview** via `draftMode()` + signed admin URL (unpublished content, no public-cache pollution). Streaming/Suspense for fast first paint.

---

## 9. CMS Rendering Strategy (FD8/FD13)

`SectionWrapper` applies `isVisible`/`order`/`theme`(scoped `data-theme` var wrapper)/`background`/`spacing`/`animation`(→ motion variants). **`BlockRenderer`** maps `block.type` → component via a **registry**; each block **Zod-validates its `data` at render** (`packages/blocks` schemas keyed by `schemaVersion`) → invalid/unknown → **safe fallback, never crash**. Reference blocks (ProjectGrid) receive server-resolved (batched) collection data. **CMS content is rendered from JSON (TipTap AST), not raw HTML**; any HTML path is `isomorphic-dompurify`-sanitized (defense-in-depth over backend write-sanitization). **Honest extensibility:** reorder + new block *instances* need zero code; a new block *type* = one isolated component + registry entry (architecture unchanged, Open/Closed).

### 9b. Admin Editor (FD11/FD14) — the CMS core

- **Page builder:** a `BlockEditorCanvas` — **dnd-kit** to add/reorder/nest sections & blocks, a block picker (same registry as the renderer), and a **per-block config panel** (RHF + the block's Zod schema). Section settings (theme/bg/spacing/animation/visibility) edited inline.
- **Rich text:** **TipTap** editor → **ProseMirror JSON** (= backend D7 AST); the public `Prose` renderer consumes the same JSON. Markdown blocks = textarea + live preview.
- **Media:** `MediaPicker` modal integrates the Media Library (browse/upload/alt).
- **Live preview:** side-by-side pane renders the exact `BlockRenderer` (shared `packages/blocks`) via draft mode.
- **Safety:** **debounced autosave** (draft), optimistic saves, dirty-state navigation guard, optimistic-lock `version` conflict → merge prompt (backend D12).

---

## 10. Responsive Strategy

Mobile-first, Tailwind breakpoints (sm 640 → 2xl 1536). Fluid type via `clamp()`; responsive 12-col grids (1→2→3); container queries for self-contained composites. **Admin:** sidebar collapses <lg; DataTable → scroll/stacked; single-column forms on mobile; touch targets ≥44px. Verified at 320/768/1024/1440.

---

## 11. SEO Strategy

`generateMetadata` per route from the entity's `seo` JSON (title/description/canonical/OG/Twitter/noIndex) with `Settings.defaultSeo` fallback. **Dynamic OG** via `next/og`. **JSON-LD:** Person + WebSite (home), Article (blog), BreadcrumbList, CreativeWork (projects). Dynamic `sitemap.ts` (from APIs), `robots.ts`, canonical everywhere; admin `noindex`. Semantic headings + descriptive links. **i18n forward-note:** locale-neutral copy + `next-intl`-ready routing if the backend i18n arrives (parity with backend "i18n-ready").

---

## 12. Accessibility Strategy

Semantic landmarks; Radix-backed primitives (correct roles/focus). Full keyboard nav; visible `focus-visible` rings; focus trap+restore in modals/drawers/⌘K; skip-to-content. **WCAG AA** contrast verified on the dark palette. `aria-*` only where semantics fall short; live regions for toasts/async. **`prefers-reduced-motion` disables non-essential motion.** Forms: labels, `aria-invalid`, error text wired to inputs. Automated **axe** checks in CI (§16).

---

## 13. Performance Strategy

RSC-first (minimal public client JS) · `next/image` with a **Cloudinary custom loader** (ADR fix — transform via Cloudinary URL params, no double-optimization) · `next/font` (subset, no CLS) · route prefetch · **dynamic imports** for heavy client widgets (TipTap editor, dnd-kit canvas, lightbox, charts) so admin weight never touches public · code-split per route · ISR/edge caching + backend `Cache-Control` · Suspense streaming · **bundle analyzer + budgets in CI**. Targets: public Lighthouse ≥95 / INP-friendly; admin fast TTI via lazy feature chunks.

---

## 14. Animation Guidelines (`motion`)

Restraint by default. Allowed: section reveal (fade + 8–16px rise, once, ~250ms ease-out, `viewport={{once:true}}`), micro-interactions on interactive elements, subtle list stagger (≤60ms). Forbidden: parallax, autoplay loops, long/bouncy easings, decorative moving gradients, animating large layout. Central `motionTokens`+variants in `packages/config`/`ui`; every animation gated by `useReducedMotion()`. CMS `AnimationType` selects from a fixed, tasteful variant set.

---

## 15. Component & CMS extensibility (verification)

- **Dynamic pages from CMS:** ✅ `BlockRenderer`.
- **Reorder sections w/o code:** ✅ `order` field.
- **New section w/o architecture change:** ✅ instances/reorder = zero code; **new block *type* = one component + registry entry** (unavoidable; architecture unchanged, Open/Closed).
- **Design system reusable/scalable:** ✅ `ui` domain-agnostic; domain in `blocks` (FD12).
- **Future themes/layouts:** ✅ theme-agnostic tokens (light additive), scoped per-section theming; layouts via blocks.
- **Admin = pro CMS not CRUD:** ✅ dnd-kit builder + TipTap + live preview + media library + ⌘K + autosave (FD11).

---

## 16. Testing & CI (ADR fix — was absent)

- **Unit/integration:** **Vitest + React Testing Library** — components (`ui`/`blocks`), hooks, services, block Zod schemas, the `BlockRenderer` registry (incl. unknown/invalid-data fallback).
- **E2E:** **Playwright** — public happy paths (home/project/blog/contact), admin auth + a CRUD flow + page-builder reorder/publish → public reflects it.
- **Accessibility:** **jest-axe** on components + Playwright+axe on key pages.
- **Visual:** Playwright screenshot snapshots for core components/pages (+ Storybook stories as the source).
- **Workbench:** **Storybook** for `packages/ui`+`blocks` (design-system docs + visual review).
- **CI gates:** typecheck → lint → unit → build → e2e → **bundle budget**; PR-blocking. Env-var validation (`@t3-oss/env-nextjs` or Zod) at build.

---

## 17. Security (ADR fix)

- **CMS XSS:** render from TipTap JSON, never raw HTML; sanitize any HTML path (`isomorphic-dompurify`) — defense-in-depth over backend write-sanitization.
- **CSP + security headers** via `next.config` `headers()`/middleware (script/style/img/connect/frame allowlists incl. Cloudinary + analytics); HSTS at the proxy.
- **Untrusted embeds:** Video/Embed blocks in **sandboxed iframes** with an allowlist.
- **Auth:** access token in memory only; refresh via first-party cookie; silent-refresh bootstrap; middleware = presence check, API = true gate; admin `noindex`.
- **Contact spam:** honeypot + **Cloudflare Turnstile**; client rate-feedback; backend still authoritative.
- **Supply chain:** lockfile + `npm audit`/Dependabot in CI; no secrets in `NEXT_PUBLIC_*`.

---

## 18. Development Roadmap (feature-at-a-time, paired with backend phases)

- **P0 — Monorepo & Design System:** Turborepo + npm workspaces; migrate current app → `apps/web`; scaffold `apps/admin`; `packages/config` (shared `@theme` tokens + Tailwind base + motion), `packages/ui` (shadcn init + primitives + **Storybook**), `packages/blocks` (shell + registry), `packages/types`, `packages/api` (server/client split + ApiError). Fix Geist wiring. **Vitest + Playwright + CI** skeleton. No pages yet.
- **P1 — Public shell & CMS home:** `SiteHeader`/`SiteFooter` (Nav/Settings APIs), `SectionWrapper` + `BlockRenderer` (with Zod validation), CMS-home + about, base SEO/metadata, sitemap/robots, CSP headers.
- **P2 — 🎯 Projects vertical slice:** list + **case-study detail** (Overview→Problem→Architecture→Responsibilities→Challenges→Solutions→Perf→Impact→Tech→Gallery) on the public API — pairs with **backend Phase 3**. First demoable end-to-end + e2e test.
- **P3 — Remaining public collections:** Experience(Timeline), Skills, Services, Education, Certificates, Testimonials, Contact (RHF+Zod+Turnstile), 404; first-party analytics wiring.
- **P4 — Blog:** list + detail (Prose from TipTap JSON, reading time, TOC, JSON-LD Article).
- **P5 — Admin foundation:** auth (login + refresh interceptor + middleware + session bootstrap), `AdminShell` (sidebar+topbar+⌘K), DataTable/form patterns, Dashboard.
- **P6 — Admin CRUD (one resource at a time):** Projects → Blogs → Media Library → Users/Roles → SEO → Navigation → Settings → Analytics; optimistic mutations + skeletons + autosave.
- **P7 — Admin Page Builder:** dnd-kit canvas + block config panels + TipTap + **live preview** (draft mode).
- **P8 — Polish & ship:** dynamic OG, structured-data + a11y (axe) + Lighthouse/bundle-budget passes, ISR revalidation receiver, deploy (apex + `admin.` subdomain), full e2e/visual suite green.

---

## Next steps

1. **Review & approve this v2 plan** (override any FD1–FD14).
2. On approval, implement **P0 (monorepo + design system + quality skeleton) only**, verify, sign-off — then one feature at a time. **No code until you approve.**
