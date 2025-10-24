# Example: Complex Form (shadcn + RHF + Zod)

## Overview
Multi-field validation with field-level errors and submit states.

## Files
```
components/forms/ProfileForm.tsx
```

## Component
```tsx
"use client"
import { z } from 'zod'
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { Button, Input, Label, Select } from '@/components/ui'

const schema = z.object({
  name: z.string().min(2),
  email: z.string().email(),
  role: z.enum(['admin','editor','viewer']),
})

export function ProfileForm() {
  const { register, handleSubmit, formState: { errors, isSubmitting } } = useForm<z.infer<typeof schema>>({ resolver: zodResolver(schema) })

  const onSubmit = handleSubmit(async (values) => {
    // call a tRPC mutation here
  })

  return (
    <form onSubmit={onSubmit} className="space-y-4">
      <div>
        <Label htmlFor="name">Name</Label>
        <Input id="name" {...register('name')} />
        {errors.name && <p className="text-sm text-red-600">{errors.name.message}</p>}
      </div>
      <div>
        <Label htmlFor="email">Email</Label>
        <Input id="email" type="email" {...register('email')} />
        {errors.email && <p className="text-sm text-red-600">{errors.email.message}</p>}
      </div>
      <div>
        <Label htmlFor="role">Role</Label>
        <Select id="role" {...register('role')} />
        {errors.role && <p className="text-sm text-red-600">{errors.role.message}</p>}
      </div>
      <Button disabled={isSubmitting} type="submit">Save</Button>
    </form>
  )
}
```

## Notes
- Extract the validation schema and reuse on the server.
- Disable button while submitting; show success/error toasts as needed.

