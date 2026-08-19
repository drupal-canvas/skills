---
name: canvas-headless
description:
  Use when working in a Canvas Headless codebase — any project with
  `@drupal-canvas/headless` or `@drupal-canvas/headless-*` in `package.json`,
  i.e. a Next.js, Nuxt, Astro, or TanStack Start app rendering Canvas
  components. Establishes what differs from Canvas-rendered React projects:
  per-framework entry files and slot consumption, the unsupported
  `drupal-canvas` package, SDK-based data fetching, and changed
  push/pull/validate semantics.
---

# Canvas Headless

Canvas Headless lets an external frontend application (Next.js, Nuxt, Astro, or
TanStack Start) render Canvas components instead of Drupal. Components are
authored in the frontend codebase in that framework's own language, and only
their **metadata** is registered with Drupal — no component code is pushed.

This skill is the source of truth for how Canvas skills apply in a headless
codebase. Other `canvas-*` skills link here instead of restating these rules.

## Detection

A project is a Canvas Headless codebase when `package.json` lists a dependency
named `@drupal-canvas/headless` or starting with `@drupal-canvas/headless-` (in
`dependencies` or `devDependencies`). This is the same rule the Canvas CLI uses
internally.

When detected, the guidance in this skill **overrides** React-specific guidance
in other Canvas skills. When not detected, the project is a Canvas-rendered
React project and the other skills apply as written. These are the only two
project types: no headless dependency always means a React codebase.

## What stays the same

The Canvas component contract is framework-neutral and carries over unchanged:

- One folder per component named by `machineName`, containing the metadata and
  the implementation entry file
- The full `component.yml` schema: `name`, `machineName`, `status`, `required`,
  `props`, `slots`, enums with `meta:enum`, image and content entity reference
  prop shapes
- Props/slots modeling rules, repeatable-content patterns, and granularity
  checks (`canvas-component-composability`, `canvas-design-decomposition`)
- Workbench mock files (`mocks.json`) beside the component — in React projects
  only, since only Workbench reads them; do not create mocks in non-React
  projects
- Tailwind CSS 4 with `@theme` design tokens in the global CSS file configured
  by `globalCssPath` in `canvas.config.json`

## Entry files and slots per framework

The implementation entry file sits beside `component.yml` and must match the
project's framework:

| Framework                 | Entry file           | Slot consumption              |
| ------------------------- | -------------------- | ----------------------------- |
| React (Next.js, TanStack) | `index.jsx` / `.tsx` | Slots received as named props |
| Vue (Nuxt)                | `index.vue`          | `<slot name="slotKey" />`     |
| Astro                     | `index.astro`        | `<slot name="slotKey" />`     |

Discovery also accepts `.svelte` entries, but no official Svelte adapter
currently exists.

Component registration is automatic: the headless SDK generates a registry
module from the discovered components (for example under `.canvas/`). Never
write manual component-to-machine-name mappings and never edit generated
`.canvas/` files.

## The `drupal-canvas` package is not supported

Do not import from the `drupal-canvas` package in a headless codebase. None of
its exports — `cn`, `FormattedText`, `Image`, `JsonApiClient`, `Region`,
`sortMenu`, `getPageData`, and the rest — are supported in headless components.
Use framework-native alternatives instead:

- **Rich text / HTML props:** render with the framework's HTML-injection
  primitive (`v-html` in Vue, `set:html` in Astro, `dangerouslySetInnerHTML` in
  React)
- **Images:** the framework's image-optimization component where one exists
  (`next/image` in Next.js, `<Image>` from `astro:assets` in Astro, `<NuxtImg>`
  from `@nuxt/image` in Nuxt); otherwise a plain `<img>` element with `src`,
  `alt`, `width`, and `height` from the image prop object
- **Class composition:** the framework's own idiom (array/object class bindings
  in Vue, `class:list` in Astro, template literals or an existing utility in
  React) — follow the conventions already present in the project.
  class-variance-authority (CVA) works in any framework when the project
  installs it.

## Data fetching

Fetch Drupal data through the headless SDK, not through `drupal-canvas`:

- **Page trees:** `fetchPage()` from the framework's adapter package. It runs
  server-side only (server components/functions in Next.js and TanStack Start,
  Nitro server routes in Nuxt, the Astro context in Astro). Pass the result to
  `CanvasComponentTree`.
- **Content queries (lists, entities, menus):** the JSON:API client from the
  headless SDK — `getClient()` (public) or the draft-aware variant — instead of
  `JsonApiClient` from `drupal-canvas`.
- Prefer the framework's idiomatic data-loading path (server components, route
  loaders, `useFetch`, Astro frontmatter) over client-side fetching libraries.

The SWR + `JsonApiClient` patterns in `canvas-data-fetching` describe
Canvas-rendered React projects; do not copy them into a headless codebase.

## CLI semantics in a headless codebase

The CLI detects the headless SDK automatically (same `package.json` rule as
above) and changes behavior without any flag:

- **`canvas push`** pushes every component as `type: external` — metadata only,
  no source or compiled code. External components on the Drupal side are created
  and updated but **never deleted** by push. A single component can also opt in
  explicitly with `type: external` in its `component.yml`.
- **`canvas pull`** excludes external components. Pull is useful once, to
  migrate previously Drupal-hosted components into the headless codebase.
- **`canvas scaffold`** is React-only. In a non-React headless project, create
  component files by hand, following an existing component in the repository.
- **`canvas build` and `canvas validate` are headless-unaware.** Expect
  `canvas build` to report "No components found. Nothing to build." in a
  headless project; this is normal and not something to fix.

## Pages, regions, and content templates

- **Global regions are not supported** in headless projects. Site chrome and
  layout belong to the framework app (its own routes and layout files, with a
  catch-all route rendering `fetchPage()` results). Do not create region specs
  or a `layout.jsx`.
- **Page specs and content templates work in headless codebases**, controlled by
  the `sync.pages` and `sync.contentTemplates` options in `canvas.config.json`.
  Only create them when the matching option is enabled. Projects generated with
  Canvas Create (`@drupal-canvas/create`) disable both by default and ship
  without `pages/` or `content-templates/` directories.

## Verification

Canvas Workbench currently supports React projects only. In React headless
projects (Next.js, TanStack Start) use Workbench as usual. In Nuxt and Astro
projects, verify components by running the framework dev server and viewing
pages rendered through `CanvasComponentTree`, including the Canvas editor
preview when connected to a Drupal site.
