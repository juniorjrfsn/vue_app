```
junior@junior-MS-7C09:~/Documentos/projetos/vue_app$ npm create vue@latest
┌  Vue.js - The Progressive JavaScript Framework
│
◇  Project name (target directory):
│  cnncheckin
│
◇  Select features to include in your project: (↑/↓ to navigate, space to select, a to toggle all, enter to confirm)
│  TypeScript, Vitest (unit testing)
│
◇  Select experimental features to include in your project: (↑/↓ to navigate, space to select, a to toggle all, enter to confirm)
│  Oxlint (experimental), rolldown-vite (experimental)
│
◇  Skip all example code and start with a blank Vue project?
│  No

Scaffolding project in /home/junior/Documentos/projetos/vue_app/cnncheckin...
│
└  Done. Now run:

   cd cnncheckin
   npm install
   npm run dev

| Optional: Initialize Git in your project directory with:
  
   git init && git add -A && git commit -m "initial commit"
```

# cnncheckin

This template should help get you started developing with Vue 3 in Vite.

## Recommended IDE Setup

[VSCode](https://code.visualstudio.com/) + [Volar](https://marketplace.visualstudio.com/items?itemName=Vue.volar) (and disable Vetur).

## Type Support for `.vue` Imports in TS

TypeScript cannot handle type information for `.vue` imports by default, so we replace the `tsc` CLI with `vue-tsc` for type checking. In editors, we need [Volar](https://marketplace.visualstudio.com/items?itemName=Vue.volar) to make the TypeScript language service aware of `.vue` types.

## Customize configuration

See [Vite Configuration Reference](https://vite.dev/config/).

## Project Setup

```sh
cd cnncheckin
npm install
```

### Compile and Hot-Reload for Development

```sh
npm run dev
```

### Type-Check, Compile and Minify for Production

```sh
npm run build
```

### Run Unit Tests with [Vitest](https://vitest.dev/)

```sh
npm run test:unit
```
