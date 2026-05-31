# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

This is **kuboard-press**: the source for the Kuboard documentation/marketing website (https://kuboard.cn), built with **VuePress 1.x**. It is a content site, not the Kuboard application itself. Content is Chinese-language Kubernetes documentation and tutorials. Output is a static site served by nginx in a Docker image.

## Commands

- `pnpm install` — install dependencies (pnpm is the canonical package manager; `pnpm-lock.yaml` is authoritative, though a `package-lock.json` also exists)
- `pnpm docs:dev` — local dev server on **port 8000** (`vuepress dev .`)
- `pnpm docs:build` — build the static site into `docs/` (`vuepress build .`)
- `./build.sh` — full release: installs, builds, then `docker buildx build` + push to Huawei SWR registry (requires `$SWR_AK`/`$SWR_PW`). Not for routine work.

There is **no test suite, no linter, and no CI config** in this repo. "Verifying" a change means running `pnpm docs:dev` and viewing the page.

### Build gotchas (hit while building on this Windows + node 24 host)

These are environment traps, not code bugs — work around them, don't "fix" the repo:

- **pnpm fails on Windows** with `EPERM: operation not permitted, symlink ...` (needs admin/developer mode for symlinks). Fall back to `npm install --legacy-peer-deps`.
- **Stale `package-lock.json` pins a dead mirror.** The committed lockfile's `resolved` URLs point at `r.cnpmjs.org`/`r2.cnpmjs.org`, whose TLS certs are expired (`CERT_HAS_EXPIRED`) → npm crashes with `Exit handler never called!`. Remove the lockfile (a `package-lock.json.bak` backup exists) and reinstall against `--registry=https://registry.npmmirror.com/` to regenerate it.
- **npmmirror serves a corrupted `lodash.template`** (reports nonexistent version 4.18.0, missing the `assignWith` definition) → build dies at render with `ReferenceError: assignWith is not defined`. Fix: `npm install lodash.template@4.5.0 "lodash.templatesettings@^4.0.0" "lodash._reinterpolate@^3.0.0" --registry=https://registry.npmjs.org/ --no-save`.
- **node 24 + VuePress 1.x (webpack 4)** needs `NODE_OPTIONS=--openssl-legacy-provider` set before `docs:build`, or the build fails on OpenSSL. (The devcontainer uses node 22 to avoid this.)
- The SSR warning `document is not defined` / `Failed to resolve async component` during "Rendering static HTML" is harmless — the 389 pages still generate.

### Running the image locally

The image is built for **Kubernetes**: `docker/nginx.80.conf` proxies `/uc-api/` to `http://svc-user-center-v3:8080`. A plain `docker run` exits immediately with `nginx: [emerg] host not found in upstream "svc-user-center-v3"` because that DNS name only resolves inside the cluster. To smoke-test the static docs locally, give nginx a dummy resolution: `docker run -p 8088:80 --add-host svc-user-center-v3:127.0.0.1 <image>` (the `/uc-api/` proxy won't work, but all static pages serve fine).

## Key conventions

- **Build output `docs/` is the publish directory** (`dest: "docs"` in config) and is git-ignored. Never hand-edit files under `docs/` — they are generated. Edit the Markdown source in the top-level content folders instead.
- The Docker image (`Dockerfile`) copies `./docs` into nginx. So a deploy requires running `pnpm docs:build` first to regenerate `docs/`.
- Content is authored as Markdown with VuePress frontmatter. The first line of many pages is `layout: SpecialHomePage` or a `description:` used for SEO (see `intro.md`, `README.md`).
- Custom Vue components are used directly inside Markdown (e.g. `<StarCount></StarCount>`, `<LearningPlan>`). See registration below before adding new ones.
- Pages live in versioned/topic folders at the repo root, each routed by path. Main areas: `v4/` (current install docs), `install/` (v3), `learning/` (K8S tutorials), `guide/` & `guide-v2/`, `overview/`, `support/`, `glossary/`, `t/`.

## Architecture

VuePress config and all customization live under `.vuepress/`:

- **`.vuepress/config.js`** — site config: title, port (8000), `dest: "docs"`, plugins (vssue comments, code-copy, code-switcher, sitemap, seo, google-analytics), and a dev proxy mapping `/uc-api/` → `http://kb:8080`. `themeConfig.nav` and `themeConfig.sidebar` drive navigation.
- **Sidebar is data-driven**: `themeConfig.sidebar = require("./config-sidebar")`. To change the sidebar/page ordering for a section, edit **`.vuepress/config-sidebar.js`** (large map keyed by route prefix, e.g. `/v4/`, `/learning/`). `config-sidebar-guide.js` is a separate map for `/guide-v2/`. Sidebar `children` entries are file paths relative to the section prefix (no extension).
- **Custom theme**: `.vuepress/theme/index.js` simply `extend`s `@vuepress/theme-default`, then overrides specific components in `.vuepress/theme/components/` (e.g. `Navbar.vue`, `Sidebar.vue`, `Page.vue`, `OnlineChat.vue`) and the 404 layout in `.vuepress/theme/layouts/`.
- **Global components, two registries** — when adding a component used in Markdown, register it in the right place:
  - `.vuepress/components/` is auto-registered by VuePress convention (filename = tag name). Mostly ads, install widgets, GitHub-star counters.
  - `.vuepress/comp/` is registered manually — you must add an entry to **`.vuepress/comp/index.js`** for the component to be usable (e.g. `LearningPlan`, `Course`, `KuboardDemo`).
- **`.vuepress/enhanceApp.js`** — app-level setup: registers BootstrapVue, a custom fraction-grid system (`.vuepress/grid/`), element-ui (mini size), vue-clipboard, AOS animations, and a `$sendGaEvent` helper. Add global Vue plugins/prototype methods here.
- **`.babelrc`** uses `babel-plugin-component` for on-demand element-ui imports (theme-chalk).

## Docker / dev container

- `Dockerfile` — production image: nginx serving `docs/`, with configs from `docker/nginx.conf` and `docker/nginx.80.conf`.
- `.devcontainer/` provides an Ubuntu 24.04 dev container with Node 22, pnpm, JDK 17, Maven, and docker CLI (the JDK/Maven are unused by this site's build).
- `docker-compose.yaml` is mostly for running the actual Kuboard app locally against these docs; not needed to edit content.
