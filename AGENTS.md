# Repository Guidelines

## Project Structure & Module Organization

Single-page static Astro site: `src/pages/index.astro` composes nine section components from `src/components/` inside `src/layouts/Layout.astro`. There are no other routes.

- **This is a brand skin, not a standalone deploy.** [astro.config.mjs](astro.config.mjs) sets `base: '/pawpay/'` and `outDir: '../fusion_formv_v1/public/pawpay'` — `npm run build` writes into the sibling `fusion_formv_v1` repo (Cloudflare Pages project `aurum-leads`), which serves this landing at `pawpayfund.com/` and hosts the shared lead form. Nothing deploys from this directory. Changing `base` or `outDir` breaks the host wiring.
- **Templated copy.** Every user-visible string and number is read directly from `import.meta.env.PUBLIC_*` inside each component, not passed as props. Fallbacks are literal placeholder tokens (`'${brand}'`, `'${redirectUrl}'`) that an external generator string-replaces — do not "fix" them into real defaults. `PUBLIC_REVIEWS` is a JSON array of `{name, location, rating, text|quote, amount?, purpose?}`, parsed defensively in `Testimonials.astro` with `FALLBACK_REVIEWS`.
- **Lead handoff contract.** Every CTA points at `applyUrl = '/?requested_amount=1500'` — the host project's lead-form SPA at the *origin root*, hardcoded in eight files (`index.astro` plus seven components). The Hero CTA overrides it at runtime: validates a 5-digit ZIP, then navigates to `/?requested_amount=<amount>&zip=<zip>` ([Hero.astro:326](src/components/Hero.astro:326)). `applyUrl` must stay root-relative and **must not** be `BASE_URL`-prefixed — the SPA lives above `/pawpay/`. Changing the param names breaks attribution on the buyer side.
- **ZIP lookup.** Resolved client-side by fetching `/zip-cities.json` from `public/` ([Hero.astro:221](src/components/Hero.astro:221)). [zip-cities.ts](src/data/zip-cities.ts) holds the same table plus `lookupZip`/`isValidUsZip` but is imported nowhere — change `public/zip-cities.json` to affect runtime.
- **Legal/marketing copy** lives in one `popupContent` map in [Footer.astro](src/components/Footer.astro); footer links and inline disclosure text open the shared modal via `data-popup="<key>"` on a button. Add the key before adding a trigger.
- **No client-side tracking, deliberately.** Voluum, Google Ads `gtag`, Facebook Pixel, GTM consent mode, and the cookie-consent banner were all stripped from this skin; attribution is handled by owned server-side click-session tracking in `fusion_formv_v1`. Do not add a third-party pixel, tag manager, or analytics script here.
- **Not build inputs:** `extracted.json`, `values_dump.json`, `index_html.txt`, `favicon_svg.txt`, `astro.config copy.mjs`, and empty `project/`. `dist/` is also dead — leftover from before `outDir` was repointed; the live output is in `fusion_formv_v1/public/pawpay/`.

## Build & Development Commands

```bash
npm install
npm run dev      # astro dev  (served under /pawpay/)
npm run build    # astro build -> ../fusion_formv_v1/public/pawpay  (NOT dist/)
npm run preview  # serve the build output
```

`build` writes outside this repo into `fusion_formv_v1/public/pawpay/` (gitignored there) — verify that path exists before building, and expect the host repo's working tree to change. Production builds inline all stylesheets and drop `console`/`debugger` ([astro.config.mjs](astro.config.mjs)), so debug in `dev`, not `preview`.

A `.env` supplying the `PUBLIC_*` values is required. Without it the page renders literal `${brand}`-style tokens — a visually broken page that still returns HTTP 200, so status checks won't catch it. Compare the rendered `<title>` to verify a skin is live and correct.

## Coding Style & Conventions

No linter, formatter, or test suite is configured; `tsconfig.json` extends `astro/tsconfigs/base` (non-strict).

- **Every internal asset URL must go through `import.meta.env.BASE_URL`.** Astro's `base` does not rewrite hardcoded `href="/…"` / `src="/…"`, and a miss silently serves the host SPA at HTTP 200 instead of 404. See `Layout.astro` (favicon) and [Hero.astro:220](src/components/Hero.astro:220) (`fetch(import.meta.env.BASE_URL + 'zip-cities.json')`). The one intentional exception is `applyUrl`, above.
- Colors, radius, and shadows are HSL custom properties in `:root` of [global.css](src/styles/global.css), surfaced through [tailwind.config.mjs](tailwind.config.mjs). Use `bg-primary` / `text-foreground`, not raw hex.
- Shared component classes (`.btn-primary`, `.premium-card`, `.amount-btn`, `.section-title`) belong in the `@layer components` block of global.css rather than repeated utility strings.
- Client JS is plain inline `<script>` in the owning `.astro` file, receiving server values via `define:vars`. No UI framework, no islands.
- Sections opt into scroll animation with `class="reveal"` (`reveal-d1`–`reveal-d3` for stagger); an IntersectionObserver in `index.astro` adds `.visible`.
