---
title: Full feature end-to-end (Projects)
tags: example, end-to-end, walkthrough
---

## Full feature, end-to-end

A feature that touches every layer: list **Projects** (TanStack Query) and create one (shadcn
dialog + RHF/Zod form + mutation with cache update + toast). Placeholder domain — replace
`Project`/`project` with yours. Build order: api → page/query → form/mutation → route → test.

### 1. API function — `src/api/get-projects.ts` / `create-project.ts`

```ts
import { api } from '@/lib/axios'

export interface Project { id: string; name: string; description: string | null }
export interface GetProjectsResponse { projects: Project[] }

export async function getProjects() {
  const response = await api.get<GetProjectsResponse>('/projects')
  return response.data
}
```

```ts
export interface CreateProjectBody { name: string; description: string | null }

export async function createProject(body: CreateProjectBody) {
  const response = await api.post<{ project: Project }>('/projects', body)
  return response.data
}
```

> If the backend exposes an OpenAPI spec, these come from Orval instead (`@/http/api`) — see
> `api-codegen-orval`.

### 2. List page with query + states — `src/pages/app/projects/projects.tsx`

```tsx
import { useQuery } from '@tanstack/react-query'
import { Dialog, DialogTrigger } from '@/components/ui/dialog'
import { Button } from '@/components/ui/button'
import { getProjects } from '@/api/get-projects'
import { CreateProjectDialog } from './create-project-dialog'
import { ProjectListSkeleton } from './project-list-skeleton'

export function Projects() {
  const { data, isLoading } = useQuery({ queryKey: ['projects'], queryFn: getProjects })

  return (
    <div className="flex flex-col gap-4">
      <div className="flex items-center justify-between">
        <h1 className="text-3xl font-bold tracking-tight">Projetos</h1>
        <Dialog>
          <DialogTrigger asChild>
            <Button>Novo projeto</Button>
          </DialogTrigger>
          <CreateProjectDialog />
        </Dialog>
      </div>

      {isLoading && <ProjectListSkeleton />}

      {data && data.projects.length === 0 && (
        <p className="text-muted-foreground">Nenhum projeto ainda. Crie o primeiro.</p>
      )}

      {data && data.projects.length > 0 && (
        <ul className="divide-y rounded-md border">
          {data.projects.map((project) => (
            <li key={project.id} className="p-4">
              <p className="font-medium">{project.name}</p>
              {project.description && (
                <p className="text-muted-foreground text-sm">{project.description}</p>
              )}
            </li>
          ))}
        </ul>
      )}
    </div>
  )
}
```

### 3. Dialog form with mutation + cache update — `create-project-dialog.tsx`

```tsx
import { zodResolver } from '@hookform/resolvers/zod'
import { useMutation, useQueryClient } from '@tanstack/react-query'
import { useForm } from 'react-hook-form'
import { toast } from 'sonner'
import { z } from 'zod'
import { createProject } from '@/api/create-project'
import type { GetProjectsResponse } from '@/api/get-projects'
import { Button } from '@/components/ui/button'
import { DialogContent, DialogHeader, DialogTitle } from '@/components/ui/dialog'
import { Input } from '@/components/ui/input'
import { Label } from '@/components/ui/label'

const createProjectForm = z.object({
  name: z.string().min(1, 'Informe um nome'),
  description: z.string().nullable(),
})
type CreateProjectForm = z.infer<typeof createProjectForm>

export function CreateProjectDialog() {
  const queryClient = useQueryClient()
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<CreateProjectForm>({ resolver: zodResolver(createProjectForm) })

  const { mutateAsync: create } = useMutation({
    mutationFn: createProject,
    onSuccess: ({ project }) => {
      queryClient.setQueryData<GetProjectsResponse>(['projects'], (cached) =>
        cached ? { projects: [project, ...cached.projects] } : cached,
      )
    },
  })

  async function handleCreate(data: CreateProjectForm) {
    try {
      await create(data)
      toast.success('Projeto criado!')
    } catch {
      toast.error('Não foi possível criar o projeto.')
    }
  }

  return (
    <DialogContent>
      <DialogHeader>
        <DialogTitle>Novo projeto</DialogTitle>
      </DialogHeader>
      <form onSubmit={handleSubmit(handleCreate)} className="space-y-4">
        <div className="space-y-2">
          <Label htmlFor="name">Nome</Label>
          <Input id="name" aria-invalid={!!errors.name} {...register('name')} />
          {errors.name && (
            <p role="alert" className="text-destructive text-sm">
              {errors.name.message}
            </p>
          )}
        </div>
        <Button type="submit" disabled={isSubmitting} className="w-full">Criar</Button>
      </form>
    </DialogContent>
  )
}
```

### 4. Register the route — `routes.tsx`

```tsx
{ path: '/projects', element: <Projects /> }, // inside the AppLayout children
```

### 5. Component test — `projects.spec.tsx`

```tsx
import { render, screen } from '@testing-library/react'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { MemoryRouter } from 'react-router-dom'
import { describe, expect, it } from 'vitest'
import { Projects } from './projects'

function renderWithProviders(ui: React.ReactNode) {
  const queryClient = new QueryClient()
  return render(
    <QueryClientProvider client={queryClient}>
      <MemoryRouter>{ui}</MemoryRouter>
    </QueryClientProvider>,
  )
}

describe('Projects', () => {
  it('renders the heading', () => {
    renderWithProviders(<Projects />)
    expect(screen.getByRole('heading', { name: 'Projetos' })).toBeInTheDocument()
  })
})
```

(With an MSW handler for `GET /projects`, assert the empty state and the rendered list — see
`testing`.)

### Checklist for any new feature

1. `src/api/*.ts` typed function(s) — or Orval-generated (`api-codegen-orval`).
2. Page with `useQuery`; handle loading (skeleton), empty, and data (`ui-states`).
3. Form with RHF + `zodResolver` + `z.infer`; shadcn inputs; field errors (`forms`).
4. Mutation that updates/invalidates the query cache; `sonner` toast feedback (`data-fetching`).
5. Register the route (+ layout/`errorElement`/404 if new area) (`routing`).
6. Component test (+ MSW handler) proportional to the risk (`testing`).
