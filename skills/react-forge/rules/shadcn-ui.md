---
title: shadcn/ui — the preferred component approach
tags: shadcn, components, radix, cva, icons, accessibility
---

## shadcn/ui

shadcn/ui is the **preferred** way to build UI: you own the component code (copied into your repo,
not a dependency), composed from Radix primitives + `class-variance-authority` (`cva`) + `cn`. It's
accessible by default (Radix) and themed through the CSS-variable tokens (see `setup-vite-tailwind`).

### Setup & adding components

```bash
npx shadcn@latest init        # writes components.json, lib/utils.ts (cn), token CSS
npx shadcn@latest add button  # adds src/components/ui/button.tsx
```

`components.json` records the choices (style, base color, icon library, aliases):

```jsonc
{
  "style": "new-york",
  "tailwind": { "css": "src/global.css", "baseColor": "zinc", "cssVariables": true },
  "iconLibrary": "lucide",
  "aliases": { "components": "@/components", "ui": "@/components/ui", "utils": "@/lib/utils" }
}
```

Import from `@/components/ui/*`. Added components live in `components/ui/`; your own shared
components live directly in `components/`.

### Composition patterns

Components use `cva` for variants and `cn` to merge classes; `asChild` (Radix `Slot`) lets a
component render as another element without extra DOM:

```tsx
// usage — variant + size from cva; asChild renders the Link as the button
<Button variant="ghost" size="sm" asChild>
  <Link to="/sign-up">Novo cadastro</Link>
</Button>
```

```tsx
// components/ui/button.tsx (shape added by the CLI)
const buttonVariants = cva('inline-flex items-center justify-center rounded-md …', {
  variants: {
    variant: { default: 'bg-primary text-primary-foreground hover:bg-primary/90', ghost: '…' },
    size: { default: 'h-9 px-4 py-2', sm: 'h-8 px-3 text-xs', icon: 'h-9 w-9' },
  },
  defaultVariants: { variant: 'default', size: 'default' },
})
```

### Customization, icons, accessibility

- **Customize** by editing the component in `components/ui/` and by changing the CSS-variable tokens
  — not by wrapping every component in another abstraction. Add a new `cva` variant when you need a
  new visual style (e.g. a `success` button variant).
- **Icons:** use **lucide-react** (shadcn's default). Size via the `[&_svg]:size-4` pattern already
  in the variants, or a class. Decorative icons get `aria-hidden`; icon-only buttons get an
  `sr-only` label.
- **Accessibility** comes from Radix — keep the provided roles/labels; don't strip them.

**Don't** create a generic `<MyButton>`/`<MyInput>` wrapper when the shadcn component already
fits — extend it in place. **Don't** force shadcn into a project that explicitly uses another
component library (precedence rule #1–2), and don't add a component you won't use.
