---
title: Styling with Tailwind CSS
tags: tailwind, cn, responsive, dark-mode, tokens
---

## Styling

Style with Tailwind utility classes against the **design tokens** (`bg-background`,
`text-muted-foreground`, `border-input`, `rounded-lg`) rather than raw colors, so theming and dark
mode come for free (see `setup-vite-tailwind`).

### Composing classes with `cn`

Use `cn` (clsx + tailwind-merge) whenever classes are conditional or merged with an incoming
`className` — tailwind-merge resolves conflicts so the last wins:

```ts
// lib/utils.ts
import { type ClassValue, clsx } from 'clsx'
import { twMerge } from 'tailwind-merge'

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

```tsx
<div className={cn('flex items-center gap-2', isActive && 'text-foreground', className)} />
```

Expose state as `data-*` attributes and style them with variant selectors — cleaner than toggling
class strings:

```tsx
// nav-link.tsx
<Link
  data-current={pathname === props.to}
  className="text-muted-foreground data-[current=true]:text-foreground flex items-center gap-1.5 text-sm font-medium"
  {...props}
/>
```

### Responsive, state, dark

- Responsive and state variants inline: `md:flex-row`, `hover:bg-accent`, `focus-visible:ring-1`,
  `disabled:opacity-50`.
- Dark mode is driven by the `.dark` class + the `dark:` variant (`dark:bg-emerald-600`); prefer
  tokens that already flip in `.dark` over per-element `dark:` overrides.
- Keep class ordering consistent with **prettier-plugin-tailwindcss** (auto-sorts classes).

**Don't** extract a `@apply` component class or a wrapper component just to avoid repeating a few
utilities — repetition in markup is fine until a real, reused pattern emerges (then prefer a shadcn
component or a `cva` variant over `@apply`). **Don't** mix a second styling system (CSS-in-JS)
into a Tailwind project.
