# arik dev - blog personal

Blog personal en **Astro** + **Tailwind CSS v4**. Un solo idioma (español).

## Estructura

```
src/
  content.config.ts      colección "blog"
  content/blog/*.md       un archivo por post
  components/             TopBar, Footer, Hero, PostCard, CommandPalette
  layouts/BaseLayout.astro
  pages/
    index.astro            home: hero + últimos posts
    blog/index.astro        listado con filtro
    blog/[slug].astro       post
    404.astro
```

## Agregar un post

Un archivo nuevo en `src/content/blog/`, con este frontmatter:

```md
---
title: "Mi nuevo post"
description: "Resumen en una o dos líneas."
date: 2026-07-23
tags: ["linux", "lo-que-sea"]
---
```

Nada más — el post aparece solo en `/blog`, en la home, y en el buscador.

## Getting started

```sh
pnpm install
pnpm dev       # http://localhost:4321
pnpm build     # astro check + astro build
pnpm preview
```
