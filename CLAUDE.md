# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

A personal blog built on the [Fuwari](https://github.com/saicaca/fuwari) Astro template, customized for Chinese (`zh_CN`) content. Stack: **Astro 5** + **Svelte 5** + **Tailwind CSS 3**, with Pagefind search, Swup page transitions, and a Sveltia CMS admin for authoring.

- **Deploy target:** GitHub Pages at `https://zhaoruizhe81-bot.github.io/blog/` (note the `/blog` base path). Push to `main` triggers `.github/workflows/deploy.yml`.
- **`astro.config.mjs`** sets `site`, `base: "/blog"`, and `trailingSlash: "always"`. These three together define the URL contract — all internal links must go through the `url()` helper (see below).

## Commands

Package manager is **pnpm** (v9.14.4 pinned via `packageManager` and enforced by a `preinstall` only-allow guard; `npm`/`yarn` installs will fail). Node ≥ 20.

| Command | Purpose |
|---|---|
| `pnpm dev` | Dev server at `localhost:4321` |
| `pnpm build` | `astro build` then `pagefind --site dist` (search index built post-build) |
| `pnpm preview` | Preview the production build |
| `pnpm check` | `astro check` (Astro type/template diagnostics) |
| `pnpm type-check` | `tsc --noEmit --isolatedDeclarations` |
| `pnpm format` | `biome format --write ./src` |
| `pnpm lint` | `biome check --write ./src` (format + lint + autofix) |
| `pnpm new-post <filename>` | Scaffold a post in `src/content/posts/` |

There is **no test runner** in this project.

## Architecture

### Configuration & personalization
All user-facing customization lives in **`src/config.ts`**, which exports `siteConfig`, `navBarConfig`, `profileConfig`, `licenseConfig`, and `expressiveCodeConfig`. Types come from `src/types/config.ts`. Edit `src/config.ts` to change title, theme hue, banner, navbar, profile, license — not the layouts.

### Content model
- Two Astro content collections in **`src/content/config.ts`**: `posts` (blog entries) and `spec` (the About page content).
- Post frontmatter is Zod-validated; fields: `title`, `published`, `updated`, `draft`, `description`, `image`, `tags`, `category`, `lang`. `prevTitle/prevSlug/nextTitle/nextSlug` are internal-use fields populated at build time by `getSortedPosts()` — **do not set them in frontmatter**.
- **Drafts are filtered out only in production builds** (`getRawSortedPosts` checks `import.meta.env.PROD`), so drafts render locally but not on the deployed site.

### Content utilities (`src/utils/content-utils.ts`)
`getSortedPosts()` sorts posts by `published` desc and injects prev/next links. `getSortedPostsList()` is a lighter variant that strips `body`. `getTagList()` / `getCategoryList()` aggregate taxonomy. Reuse these instead of calling `getCollection("posts")` directly, so draft-filtering and sorting stay consistent.

### Markdown pipeline (`astro.config.mjs`)
Posts are rendered through a custom remark/rehype chain. Custom plugins live in **`src/plugins/`**:
- `remark-reading-time.mjs`, `remark-excerpt.js` — word count and excerpt injected into `remarkPluginFrontmatter`.
- `rehype-component-admonition.mjs` — `:::note`/`tip`/`important`/`caution`/`warning` admonitions (also GitHub `> [!NOTE]` syntax via `remark-github-admonitions-to-directives`).
- `rehype-component-github-card.mjs` — `::github{repo="owner/repo"}` cards (fetches GitHub API at build time).
- `remark-directive-rehype.js` — handles `:spoiler[text]` spans.
- **Expressive Code** custom plugins in `src/plugins/expressive-code/` (language-badge, custom-copy-button). The EC theme **must be a dark theme** — light code backgrounds are unsupported. EC background/colors are overridden in `astro.config.mjs` via CSS vars.

### URL handling (`src/utils/url-utils.ts`)
Because of the `/blog` base path, **always build internal URLs with the `url()` helper** (`url("/posts/slug/")`), never hardcode paths. `getPostUrlBySlug`, `getTagUrl`, `getCategoryUrl` already wrap `url()`. Swup transitions swap `main` and `#toc` containers between navigations.

### Images (`src/components/misc/ImageWrapper.astro`)
Local images are resolved at build time via `import.meta.glob` relative to a `basePath`. Post cover images use `basePath="content/posts/<dir>/"`. The CMS uploads media to `src/assets/images/posts`. Remote (`http(s)://`) and `/`-prefixed public paths bypass the glob import.

### Theming (`src/utils/setting-utils.ts`)
Light/dark/auto mode and a configurable hue are persisted in `localStorage`. The default hue is passed to the client via a hidden `#config-carrier` element (`ConfigCarrier.astro` → read by `getDefaultHue()`); the hue drives the `--hue` CSS variable powering the accent color. `applyThemeToDocument` toggles the `dark` class on `<html>` and sets the Expressive Code `data-theme`.

### i18n (`src/i18n/`)
`i18nKey.ts` enumerates all UI strings; `translation.ts` maps `siteConfig.lang` → a language file in `languages/`. `i18n(key)` is the single entry point. Add new strings by extending the `I18nKey` enum **and** every language file (TypeScript will error otherwise).

### CMS admin (`public/admin/`)
A **Sveltia CMS** instance (`index.html` loads it from CDN; `config.yml` defines the `posts` collection and GitHub backend `zhaoruizhe81-bot/blog`). Accessible at `/blog/admin/` on the deployed site. Auth uses a GitHub PAT (see recent git history for the migration off Decap/OAuth). The `decap-cms` entry in `package.json` is vestigial — the admin runs from CDN and is not bundled.

## Conventions

- **Path aliases** (tsconfig): `@components`, `@assets`, `@constants`, `@utils`, `@i18n`, `@layouts`, and `@/*`. Prefer these over relative imports.
- **Formatting** (biome.json): tabs, double quotes. Biome deliberately relaxes `useConst`/`useImportType`/unused-var rules for `.astro`/`.svelte`/`.vue` — match surrounding code in those files rather than applying strict TS conventions.
- **Strict TypeScript**: `astro/tsconfigs/strict` + `strictNullChecks` + `allowJs: false`. `pnpm check` and `pnpm type-check` must stay clean.
- When adding a post frontmatter field, update **all** of: the Zod schema (`src/content/config.ts`), the CMS field list (`public/admin/config.yml`), and `BlogPostData` (`src/types/config.ts`).
