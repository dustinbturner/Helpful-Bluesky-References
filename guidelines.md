# Next.js App Router & TypeScript Standards Guide

## Table of Contents
1. [Project Structure](#project-structure)
2. [TypeScript Standards](#typescript-standards)
3. [Component Guidelines](#component-guidelines)
4. [Next.js App Router Guidelines](#nextjs-app-router-guidelines)
5. [State Management](#state-management)
6. [Styling Guidelines](#styling-guidelines)
7. [Performance & Error Handling](#performance-and-error-handling)

## Project Structure

### Directory Organization
```
src/
├── app/                     # App router pages
│   ├── (auth)/             # Auth group route
│   ├── (dashboard)/        # Dashboard group route
│   ├── api/                # API routes
│   ├── error.tsx           # Error handling
│   ├── layout.tsx          # Root layout
│   └── page.tsx            # Home page
├── components/             
│   ├── ui/                 # Shadcn UI components
│   └── features/           # Feature-specific components
├── lib/                    # Utility functions
│   ├── utils.ts           
│   └── validations.ts    
├── hooks/                  # Custom React hooks
├── types/                  # TypeScript types
│   ├── common.ts
│   └── api.ts
└── styles/                 # Global styles
```

### File Naming Conventions
- Use kebab-case for directories and files: `user-profile/`
- Use PascalCase for components: `UserProfile.tsx`
- Use camelCase for utilities: `formatDate.ts`
- Prefix types/interfaces with 'I' or 'T': `IUser` or `TApiResponse`

## TypeScript Standards

### Base Types and Interfaces
```typescript
// Common type patterns
type TRoute = {
  params: { [key: string]: string }
  searchParams: { [key: string]: string | string[] | undefined }
}

// Shared interfaces
interface IBaseProps {
  className?: string
  children?: React.ReactNode
}

// Utility types
type TStatus = 'idle' | 'loading' | 'success' | 'error'
type TNullable<T> = T | null
type TOptional<T> = T | undefined
```

### Component Props
```typescript
// Extend HTML elements
interface IButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'default' | 'outline' | 'ghost'
  isLoading?: boolean
}

// Page props
interface IPageProps {
  params: { slug: string }
  searchParams: { [key: string]: string | string[] | undefined }
}
```

### Type Safety Patterns
```typescript
// API response typing
interface IApiResponse<T> {
  data: T
  error: string | null
  status: number
}

// Form data typing with Zod
import { z } from 'zod'

const userSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8)
})

type TUserFormData = z.infer<typeof userSchema>
```

## Component Guidelines

### Base Component Structure
```typescript
import { type FC } from 'react'
import { cn } from '@/lib/utils'

interface IComponentProps extends IBaseProps {
  title: string
  onAction: () => void
}

/**
 * @component ComponentName
 * @description Component purpose
 */
export const ComponentName: FC<IComponentProps> = ({
  title,
  onAction,
  className,
  ...props
}) => {
  // Early returns
  if (!title) return null

  return (
    <div 
      className={cn("base-classes", className)}
      {...props}
    >
      {title}
    </div>
  )
}
```

### Client Components
```typescript
'use client'

interface IClientComponentProps {
  onInteraction: () => void
}

/**
 * Client Component
 * @requires client-side - Uses browser APIs: localStorage
 */
export const ClientComponent: FC<IClientComponentProps> = ({ onInteraction }) => {
  // Implementation
}
```

## Next.js App Router Guidelines

### Page Components
```typescript
// src/app/[slug]/page.tsx
import { type Metadata } from 'next'

interface IPageProps {
  params: { slug: string }
  searchParams: { [key: string]: string | undefined }
}

export const metadata: Metadata = {
  title: 'Page Title'
}

export default async function Page({ params, searchParams }: IPageProps) {
  // Implementation
}
```

### Route Handlers
```typescript
// src/app/api/users/route.ts
import { type NextRequest } from 'next/server'

export async function GET(request: NextRequest) {
  const searchParams = request.nextUrl.searchParams
  const query = searchParams.get('query')
  
  // Implement with type safety
}
```

### Server Actions
```typescript
'use server'

import { z } from 'zod'

const inputSchema = z.object({
  title: z.string()
})

export async function serverAction(input: z.infer<typeof inputSchema>) {
  const validated = inputSchema.parse(input)
  // Implementation
}
```

## State Management

### Hooks Usage
```typescript
interface IUseDataResult<T> {
  data: T | null
  isLoading: boolean
  error: Error | null
}

function useData<T>(url: string): IUseDataResult<T> {
  const [state, setState] = useState<IUseDataResult<T>>({
    data: null,
    isLoading: true,
    error: null
  })

  // Implementation
  
  return state
}
```

## Styling Guidelines

### Tailwind Usage
```typescript
// Organized className structure
<div
  className={cn(
    // Layout
    "grid grid-cols-1 md:grid-cols-2",
    // Spacing
    "gap-4 p-6",
    // Colors & States
    "bg-background hover:bg-muted",
    // Conditionals
    isActive && "border-primary",
    className
  )}
>
```

### Shadcn/UI Components
```typescript
// Custom variants
import { cva, type VariantProps } from 'class-variance-authority'

const buttonVariants = cva(
  "rounded-md transition-colors",
  {
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground",
        outline: "border border-input bg-background"
      },
      size: {
        default: "h-10 px-4 py-2",
        sm: "h-9 px-3",
        lg: "h-11 px-8"
      }
    },
    defaultVariants: {
      variant: "default",
      size: "default"
    }
  }
)

interface IButtonProps extends VariantProps<typeof buttonVariants> {
  // Additional props
}
```

## Performance and Error Handling

### Error Boundaries
```typescript
interface IErrorBoundaryProps {
  children: React.ReactNode
  fallback: React.ReactNode
}

export default function ErrorBoundary({ children, fallback }: IErrorBoundaryProps) {
  // Implementation
}
```

### API Error Handling
```typescript
class ApiError extends Error {
  constructor(
    message: string,
    public statusCode: number,
    public code: string
  ) {
    super(message)
    this.name = 'ApiError'
  }
}

async function fetchData<T>(url: string): Promise<T> {
  try {
    const response = await fetch(url)
    if (!response.ok) {
      throw new ApiError(
        'Failed to fetch',
        response.status,
        'FETCH_ERROR'
      )
    }
    return response.json()
  } catch (error) {
    if (error instanceof ApiError) {
      // Handle API error
    }
    throw error
  }
}
```

Remember:
- Always use TypeScript's strict mode
- Keep components as server components by default
- Use 'use client' only when necessary
- Implement proper TypeScript interfaces for all props
- Use Zod for runtime type validation
- Keep styling organized with Tailwind
- Handle errors with proper typing
- Use early returns for better code clarity
- Document complex logic with clear comments