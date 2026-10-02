# arik dev - blog personal

Blog personal en **Astro** + **Tailwind CSS v4**. Un solo idioma (español), sin sección de proyectos — el portfolio vive en un proyecto aparte.

## Dirección de diseño

Se mantiene la paleta oscura navy-violeta original, pero con la temática de terminal atenuada:

- Ya no hay tarjetas con barra de título y "los tres puntitos" — el contenido va en `.card`, un contenedor simple con borde y hover sutil (`src/styles/global.css`).
- El buscador (`Ctrl/Cmd+K`) sigue existiendo, pero sin el marco tipo rofi: es un cuadro de búsqueda limpio.
- Lo único que queda del acento "shell" es tipográfico: JetBrains Mono en etiquetas, tags y metadatos, IBM Plex Sans en el cuerpo — y el separador punteado (`.status-rule`) entre secciones.

## Estructura

```
src/
  content.config.ts      colección "blog" (sin carpetas de idioma)
  content/blog/*.md       un archivo por post
  components/             TopBar, Footer, Hero, PostCard, CommandPalette
  layouts/BaseLayout.astro
  pages/
    index.astro            home: hero + últimos posts
    blog/index.astro        listado con filtro
    blog/[slug].astro       post
    sobre-mi.astro
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
