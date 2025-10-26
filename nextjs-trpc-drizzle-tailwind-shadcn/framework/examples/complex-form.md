# Complex Form

A multi‑step form using shadcn/ui, Zod validation, tRPC mutation, and conditional fields.

## Schema

```ts
// src/lib/api/commands/mutations/issues/create-issue.mutation.ts
import { z } from 'zod'

export const CreateIssueInputSchema = z
  .object({
    title: z.string().min(1),
    description: z.string().min(1),
    type: z.enum(['bug', 'feature']),
    severity: z.enum(['low', 'medium', 'high']).optional(),
  })
  .refine((v) => (v.type === 'bug' ? !!v.severity : true), { message: 'Severity required for bugs' })

export const CreateIssueOutputSchema = z.object({ id: z.number() })

export async function createIssue(_ctx: { userId: string }, input: z.infer<typeof CreateIssueInputSchema>) {
  // persist and return id
  return { id: 1 }
}
```

## Router

```ts
// src/lib/api/routers/issues.router.ts
import { createTRPCRouter, rbacProcedure } from '../trpc'
import { CreateIssueInputSchema, CreateIssueOutputSchema, createIssue } from '../commands/mutations/issues/create-issue.mutation'

export const issuesRouter = createTRPCRouter({
  create: rbacProcedure.input(CreateIssueInputSchema).output(CreateIssueOutputSchema).mutation(({ ctx, input }) => createIssue(ctx, input)),
})
```

## UI

```tsx
// src/components/features/issues/CreateIssueClient.tsx
'use client'

import { z } from 'zod'
import { useState } from 'react'
import { useMutation } from '@tanstack/react-query'
import { trpc } from '@/lib/api/react'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Select } from '@/components/ui/select'

const step1 = z.object({ title: z.string().min(1) })
const step2 = z.object({ description: z.string().min(1) })
const step3 = z.object({ type: z.enum(['bug', 'feature']), severity: z.enum(['low', 'medium', 'high']).optional() })

type IssueType = z.infer<typeof step3>['type']
type Severity = NonNullable<z.infer<typeof step3>['severity']>
type State = {
  title?: string
  description?: string
  type?: IssueType
  severity?: Severity | ''
}

const isSeverity = (v: string): v is Severity =>
  (['low', 'medium', 'high'] as const).includes(v as Severity)

export default function CreateIssueClient() {
  const [step, setStep] = useState(1)
  const [state, setState] = useState<State>({})
  const create = useMutation(trpc.issues.create.mutationOptions())

  function next<TSchema extends z.ZodTypeAny>(partial: Partial<z.infer<TSchema>>, schema: TSchema) {
    const candidate = { ...state, ...partial }
    const res = schema.safeParse(candidate)
    if (!res.success) return
    setState(candidate)
    setStep(step + 1)
  }

  async function submit() {
    const payload = {
      title: state.title!,
      description: state.description!,
      type: (state.type ?? 'bug') as IssueType,
      ...(state.type === 'bug' && isSeverity(String(state.severity)) ? { severity: state.severity } : {}),
    }
    await create.mutateAsync(payload)
  }

  return (
    <div className="space-y-4">
      {step === 1 && (
        <form
          onSubmit={(e) => {
            e.preventDefault()
            next({ title: new FormData(e.currentTarget).get('title') }, step1)
          }}
          className="flex gap-2"
        >
          <Input name="title" placeholder="Title" />
          <Button type="submit">Next</Button>
        </form>
      )}

      {step === 2 && (
        <form
          onSubmit={(e) => {
            e.preventDefault()
            next({ description: new FormData(e.currentTarget).get('description') }, step2)
          }}
          className="flex gap-2"
        >
          <Input name="description" placeholder="Description" />
          <Button type="submit">Next</Button>
        </form>
      )}

      {step === 3 && (
        <form
          onSubmit={(e) => {
            e.preventDefault()
            next({ type: state.type ?? 'bug', severity: state.severity || undefined }, step3)
          }}
          className="flex items-end gap-2"
        >
          <Select value={state.type ?? 'bug'} onValueChange={(v) => setState((s) => ({ ...s, type: (v as IssueType) }))}>
            <option value="bug">Bug</option>
            <option value="feature">Feature</option>
          </Select>
          <Select value={state.severity ?? ''} onValueChange={(v) => setState((s) => ({ ...s, severity: isSeverity(v) ? v : '' }))}>
            <option value="">Severity</option>
            <option value="low">Low</option>
            <option value="medium">Medium</option>
            <option value="high">High</option>
          </Select>
          {/* If you need FormData compatibility, mirror values to hidden inputs */}
          <input type="hidden" name="type" value={state.type ?? 'bug'} />
          <input type="hidden" name="severity" value={state.severity ?? ''} />
          <Button type="submit">Review</Button>
        </form>
      )}

      {step === 4 && (
        <div className="space-y-2">
          <div>Title: {String(state.title)}</div>
          <div>Description: {String(state.description)}</div>
          <div>Type: {String(state.type)}</div>
          {state.type === 'bug' && <div>Severity: {String(state.severity)}</div>}
          <Button onClick={submit} disabled={create.isPending}>Create</Button>
        </div>
      )}
    </div>
  )
}
```

## Verification

- Severity is required when type is bug
- Successful submit returns an id
- Steps block on invalid inputs

## Related

- ../patterns/forms.md
- ../patterns/mutations.md
