---
title: Vite + Tailwind CSS v4 setup
tags: vite, tailwind, setup, alias, theme
---

## Vite + Tailwind v4

Tailwind v4 (current stable) is **CSS-first**: install the Vite plugin, import Tailwind in one CSS
file, and configure the design system in that CSS with `@theme` — **no `tailwind.config.js` is
required**. Verify versions against the official docs before pinning.

```bash
npm install tailwindcss @tailwindcss/vite
npm install -D @types/node
```

```ts
// vite.config.ts
import path from 'node:path'
import tailwindcss from '@tailwindcss/vite'
import react from '@vitejs/plugin-react'
import { defineConfig } from 'vite'

export default defineConfig({
  plugins: [react(), tailwindcss()],
  resolve: {
    alias: { '@': path.resolve(__dirname, './src') },
  },
})
```

Set the `@/*` alias in TypeScript too, so the editor and `tsc` resolve it (mirror it in
`tsconfig.json` **and** `tsconfig.app.json`):

```jsonc
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": { "@/*": ["./src/*"] }
  }
}
```

The CSS entry imports Tailwind and declares tokens with `@theme` (v4). shadcn adds CSS variables in
`:root`/`.dark` and maps them into Tailwind color tokens:

```css
/* src/global.css */
@import "tailwindcss";
@plugin "tailwindcss-animate";
@custom-variant dark (&:is(.dark *));

@theme inline {
  --color-background: var(--background);
  --color-foreground: var(--foreground);
  --color-primary: var(--primary);
  --color-primary-foreground: var(--primary-foreground);
  --color-muted-foreground: var(--muted-foreground);
  --radius-lg: var(--radius);
}

:root {
  --radius: 0.65rem;
  --background: oklch(1 0 0);
  --foreground: oklch(0.141 0.005 285.823);
  --primary: oklch(0.577 0.245 27.325);
  /* …full token set is written by `shadcn init` */
}

.dark {
  --background: oklch(0.141 0.005 285.823);
  --foreground: oklch(0.985 0 0);
  /* … */
}

@layer base {
  * { @apply border-border outline-ring/50; }
  body { @apply bg-background text-foreground; }
}
```

- Define colors/radii/fonts as **CSS variables** and expose them through `@theme`, so classes like
  `bg-background`, `text-muted-foreground`, `rounded-lg` resolve to your tokens and dark mode is a
  single `.dark` class flip.
- A minimal `tailwind.config.js` may still exist (shadcn references it), but v4 config belongs in
  CSS. Don't port v3 `theme.extend` JS config into a v4 project.

**Don't** hand-write PostCSS/Tailwind v3 config in a v4 project, and don't read colors as hex in
components — use the token classes.
