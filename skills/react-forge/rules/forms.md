---
title: Forms — React Hook Form + Zod
tags: forms, react-hook-form, zod, validation, shadcn
---

## Forms

For validated or multi-field forms, use **React Hook Form** with a **Zod** schema via `zodResolver`,
and derive the form type with `z.infer`. Wire shadcn inputs with `register`, and submit through
`handleSubmit`.

```tsx
import { zodResolver } from '@hookform/resolvers/zod'
import { useForm } from 'react-hook-form'
import { z } from 'zod'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Label } from '@/components/ui/label'

const signUpForm = z.object({
  name: z.string().min(1, 'Informe o nome'),
  email: z.email('E-mail inválido'), // Zod 4 top-level; on Zod 3 use z.string().email()
})

type SignUpForm = z.infer<typeof signUpForm>

export function SignUpForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<SignUpForm>({ resolver: zodResolver(signUpForm) })

  async function onSubmit(data: SignUpForm) {
    // call a mutation / api fn here (see data-fetching)
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-4">
      <div className="space-y-2">
        <Label htmlFor="email">E-mail</Label>
        <Input id="email" type="email" aria-invalid={!!errors.email} {...register('email')} />
        {errors.email && (
          <p role="alert" className="text-destructive text-sm">
            {errors.email.message}
          </p>
        )}
      </div>

      <Button type="submit" disabled={isSubmitting} className="w-full">
        Criar conta
      </Button>
    </form>
  )
}
```

Conventions:
- The Zod schema is the single source of truth: validation messages live in the schema; the TS type
  is `z.infer`. Don't hand-write a parallel interface.
- Show field errors from `formState.errors` (`errors.<field>?.message`); disable the submit with
  `isSubmitting`. For shadcn's `Field`/`FieldError` primitives, pass `errors={[errors.<field>]}`.
- **Accessibility:** set `aria-invalid` on the input and `role="alert"` on the message so the error
  is announced and tied to the field (shadcn's `Field`/`FieldError` wire this for you).
- **Zod version:** on **Zod 4** use the top-level format validators (`z.email()`, `z.url()`,
  `z.uuid()`) — the string-method forms (`z.string().email()`) are deprecated. On **Zod 3** only the
  method forms exist. Match the project's installed Zod version.
- Submitting usually calls a TanStack Query **mutation** or an `src/api` function (see
  `data-fetching`); give user feedback with a `sonner` toast (see `ui-states`).
- **Default values** from server data: use RHF `values` (not `defaultValues`) so the form syncs when
  a query resolves — e.g. `values: { name: data?.name ?? '' }`.

**When not to use it:** a single uncontrolled field (e.g. a search box that just reads
`searchParams`) doesn't need RHF — a plain input or `register` without a resolver is enough. Add the
resolver when there's real validation.
