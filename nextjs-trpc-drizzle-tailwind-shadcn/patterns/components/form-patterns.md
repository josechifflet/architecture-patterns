# Form Patterns

## When to Use
- Validated user input with live feedback
- Submit mutations to tRPC

## Structure
```
components/
  forms/
    PostForm.tsx        # client component, validated
lib/
  zod.ts                # shared zod schemas (optional)
```

## Complete Example
File: `src/components/forms/PostForm.tsx`
```tsx
"use client"
import { z } from 'zod'
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { trpc } from '@/lib/trpcClient'
import { Button, Input, Label } from '@/components/ui'

const schema = z.object({
  title: z.string().min(3),
})

export function PostForm() {
  const { register, handleSubmit, formState: { errors, isSubmitting }, reset } = useForm<z.infer<typeof schema>>({
    resolver: zodResolver(schema),
    defaultValues: { title: '' },
  })

  const onSubmit = handleSubmit(async (values) => {
    await trpc.posts.create.mutate(values)
    reset()
  })

  return (
    <form onSubmit={onSubmit} className="space-y-3">
      <div className="space-y-1">
        <Label htmlFor="title">Title</Label>
        <Input id="title" {...register('title')} />
        {errors.title && <p className="text-sm text-red-600">{errors.title.message}</p>}
      </div>
      <Button disabled={isSubmitting} type="submit">Create</Button>
    </form>
  )
}
```

## Anti-Patterns
❌ Duplicate validation rules in UI and server.
✅ Reuse the exact Zod schema on both sides.

❌ Fire-and-forget mutations with no error handling.
✅ Surface errors to the user and reset on success.

## Checklist
- [ ] Client component with `"use client"`
- [ ] Zod schema shared with server procedure
- [ ] Proper disabled/loading states
- [ ] Error rendering that matches brand tone

## Related Patterns
- See: `../data/mutations.md`
- See: `shadcn-usage.md`

