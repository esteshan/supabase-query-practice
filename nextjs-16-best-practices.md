# Next.js 16 Best Practices

> **Version**: Next.js 16.0.5 | **Last Updated**: November 2025  
> **Sources**: [Next.js Documentation](https://nextjs.org/docs), [Scott Moss - Next.js Fundamentals](https://github.com/Hendrixer/next.js-fundamentals)

This document serves as a reference for AI assistants and developers to ensure Next.js 16 best practices are followed. Review this document periodically when working on Next.js projects.

---

## Table of Contents

1. [Core Architecture Principles](#1-core-architecture-principles)
2. [Server and Client Components](#2-server-and-client-components)
3. [Cache Components](#3-cache-components)
4. [Data Fetching Patterns](#4-data-fetching-patterns)
5. [Server Actions (Mutations)](#5-server-actions-mutations)
6. [Error Handling](#6-error-handling)
7. [Project Structure](#7-project-structure)
8. [Data Security](#8-data-security)
9. [Quick Reference Checklist](#9-quick-reference-checklist)

---

## 1. Core Architecture Principles

These principles are inspired by Scott Moss's course and updated for Next.js 16.

### RSC-First Development

**Always start with Server Components.** Only add `'use client'` when you need:

- State (`useState`, `useReducer`)
- Event handlers (`onClick`, `onChange`)
- Lifecycle effects (`useEffect`)
- Browser APIs (`localStorage`, `window`, `navigator`)

```tsx
// GOOD: Server Component by default - fetches data on server
export default async function Page() {
  const data = await fetchData()
  return <ProductList products={data} />
}

// BAD: Unnecessary client component
'use client'
export default function Page() {
  const [data, setData] = useState([])
  useEffect(() => { fetchData().then(setData) }, [])
  return <ProductList products={data} />
}
```

### Colocation Philosophy

Keep related code together. Place server actions, schemas, and types next to the route/component using them.

```
app/
├── dashboard/
│   ├── page.tsx           # Route component
│   ├── _lib/
│   │   ├── actions.ts     # Server actions for this route
│   │   └── data.ts        # Data fetching functions
│   ├── _components/
│   │   └── chart.tsx      # Route-specific components
│   └── types.ts           # Types/schemas for this route
```

### Small Component Surfaces

Design components with narrow props. Avoid "god" components that accept everything.

```tsx
// GOOD: Narrow, specific props
interface ProductCardProps {
  name: string
  price: number
  imageUrl: string
}

// BAD: Overly broad props
interface ProductCardProps {
  product: Product // Exposes entire object including sensitive fields
}
```

### Typed Boundaries with Validation

Use Zod at input boundaries; infer TypeScript types from Zod schemas.

```tsx
import { z } from 'zod'

// Define schema once
const CreatePostSchema = z.object({
  title: z.string().min(1).max(100),
  content: z.string().min(1),
})

// Infer type from schema
type CreatePostInput = z.infer<typeof CreatePostSchema>

// Use in server action
export async function createPost(formData: FormData) {
  'use server'
  const result = CreatePostSchema.safeParse({
    title: formData.get('title'),
    content: formData.get('content'),
  })
  
  if (!result.success) {
    return { ok: false, issues: result.error.flatten() }
  }
  
  // Proceed with validated data
  const { title, content } = result.data
}
```

### Guard Clauses and Happy Path

Use early returns for errors. Never bury error branches deep in code.

```tsx
// GOOD: Early returns, happy path flows naturally
export async function getPost(id: string) {
  const session = await getSession()
  if (!session) {
    return { ok: false, error: 'Unauthorized' }
  }

  const post = await db.post.findUnique({ where: { id } })
  if (!post) {
    return { ok: false, error: 'Not found' }
  }

  if (post.authorId !== session.userId) {
    return { ok: false, error: 'Forbidden' }
  }

  return { ok: true, data: post }
}

// BAD: Deeply nested conditionals
export async function getPost(id: string) {
  const session = await getSession()
  if (session) {
    const post = await db.post.findUnique({ where: { id } })
    if (post) {
      if (post.authorId === session.userId) {
        return { ok: true, data: post }
      } else {
        return { ok: false, error: 'Forbidden' }
      }
    } else {
      return { ok: false, error: 'Not found' }
    }
  } else {
    return { ok: false, error: 'Unauthorized' }
  }
}
```

### Caching Strategy

**Static by default.** Opt into dynamic rendering only when needed.

- Use `'use cache'` for data that changes infrequently
- Use `cacheLife` to control cache duration
- Use `revalidatePath`/`revalidateTag` intentionally
- Avoid excessive `router.refresh()` calls

### Streaming UX

Pair `loading.tsx` and `<Suspense>` with server streams for fast Time to Interactive (TTI).

```tsx
// app/dashboard/loading.tsx - Shows while page streams
export default function Loading() {
  return <DashboardSkeleton />
}

// app/dashboard/page.tsx - Granular streaming with Suspense
import { Suspense } from 'react'

export default function DashboardPage() {
  return (
    <div>
      <h1>Dashboard</h1>
      <Suspense fallback={<ChartSkeleton />}>
        <RevenueChart />
      </Suspense>
      <Suspense fallback={<TableSkeleton />}>
        <RecentOrders />
      </Suspense>
    </div>
  )
}
```

---

## 2. Server and Client Components

### When to Use Each

| Use Server Components When | Use Client Components When |
|---------------------------|---------------------------|
| Fetching data from databases/APIs | Managing state (`useState`) |
| Accessing secrets/API keys | Using event handlers (`onClick`) |
| Reducing client JS bundle | Using browser APIs (`localStorage`) |
| Heavy computations | Using lifecycle effects (`useEffect`) |
| Reading from filesystem | Using custom hooks with state |

### The `'use client'` Directive

Place `'use client'` at the top of files that need client-side features. Everything imported below becomes part of the client bundle.

```tsx
// app/ui/search.tsx
'use client'

import { useState } from 'react'

export default function Search() {
  const [query, setQuery] = useState('')
  return (
    <input
      value={query}
      onChange={(e) => setQuery(e.target.value)}
      placeholder="Search..."
    />
  )
}
```

### Reducing JS Bundle Size

Add `'use client'` only to specific interactive components, not entire pages.

```tsx
// GOOD: Only Search is a client component
import Search from './search'    // Client Component
import Logo from './logo'        // Server Component
import Nav from './nav'          // Server Component

export default function Layout({ children }: { children: React.ReactNode }) {
  return (
    <>
      <nav>
        <Logo />
        <Search />
        <Nav />
      </nav>
      <main>{children}</main>
    </>
  )
}
```

### Passing Data from Server to Client

Props passed to Client Components must be serializable.

```tsx
// app/[id]/page.tsx - Server Component
import LikeButton from '@/app/ui/like-button'
import { getPost } from '@/lib/data'

export default async function Page({
  params,
}: {
  params: Promise<{ id: string }>
}) {
  const { id } = await params
  const post = await getPost(id)

  // Only pass what the client needs
  return <LikeButton initialLikes={post.likes} postId={post.id} />
}

// app/ui/like-button.tsx - Client Component
'use client'

import { useState } from 'react'

export default function LikeButton({ 
  initialLikes, 
  postId 
}: { 
  initialLikes: number
  postId: string 
}) {
  const [likes, setLikes] = useState(initialLikes)
  // ...
}
```

### Interleaving Pattern

Pass Server Components as children to Client Components using the slot pattern.

```tsx
// app/ui/modal.tsx - Client Component
'use client'

import { useState } from 'react'

export default function Modal({ children }: { children: React.ReactNode }) {
  const [isOpen, setIsOpen] = useState(false)
  
  if (!isOpen) return <button onClick={() => setIsOpen(true)}>Open</button>
  
  return (
    <div className="modal">
      {children}
      <button onClick={() => setIsOpen(false)}>Close</button>
    </div>
  )
}

// app/page.tsx - Server Component
import Modal from './ui/modal'
import Cart from './ui/cart' // Server Component that fetches data

export default function Page() {
  return (
    <Modal>
      <Cart /> {/* Rendered on server, passed as children */}
    </Modal>
  )
}
```

### Context Providers

Wrap context providers in Client Components and use them in layouts.

```tsx
// app/providers/theme-provider.tsx
'use client'

import { createContext, useContext } from 'react'

const ThemeContext = createContext('light')

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  return (
    <ThemeContext.Provider value="dark">
      {children}
    </ThemeContext.Provider>
  )
}

export const useTheme = () => useContext(ThemeContext)

// app/layout.tsx
import { ThemeProvider } from './providers/theme-provider'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        <ThemeProvider>{children}</ThemeProvider>
      </body>
    </html>
  )
}
```

**Best Practice**: Render providers as deep as possible in the tree to optimize static rendering.

### Third-Party Components

Wrap third-party components that use client features in your own Client Component.

```tsx
// app/ui/carousel.tsx
'use client'

import { Carousel } from 'acme-carousel'

export default Carousel

// Now use it in Server Components
import Carousel from './ui/carousel'

export default function Page() {
  return <Carousel />
}
```

### Preventing Environment Poisoning

Use the `server-only` package to prevent server code from being imported on the client.

```tsx
// lib/data.ts
import 'server-only'

export async function getSecretData() {
  const res = await fetch('https://api.example.com/data', {
    headers: {
      authorization: process.env.API_KEY, // Safe - server only
    },
  })
  return res.json()
}
```

Install with: `npm install server-only`

---

## 3. Cache Components

Cache Components is a **Next.js 16 feature** that enables Partial Prerendering (PPR). It lets you mix static, cached, and dynamic content in a single route.

### Enabling Cache Components

```ts
// next.config.ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  cacheComponents: true,
}

export default nextConfig
```

### How It Works

1. At build time, Next.js prerenders your route into a **static HTML shell**
2. Components that can't complete during prerendering must be:
   - Wrapped in `<Suspense>` (deferred to request time), OR
   - Marked with `'use cache'` (cached in static shell)

### The `'use cache'` Directive

Cache async functions and components. Arguments become part of the cache key.

```tsx
import { cacheLife } from 'next/cache'

// Cache at component level
export default async function BlogPosts() {
  'use cache'
  cacheLife('hours') // Cache for 1 hour
  
  const posts = await fetch('https://api.example.com/posts')
  return (
    <ul>
      {posts.map(post => <li key={post.id}>{post.title}</li>)}
    </ul>
  )
}

// Cache at function level
async function getProducts(category: string) {
  'use cache'
  cacheLife('days')
  
  return db.products.findMany({ where: { category } })
}
```

### Cache Life Profiles

Use built-in profiles or custom configurations:

```tsx
import { cacheLife } from 'next/cache'

// Built-in profiles
cacheLife('seconds')  // Short-lived cache
cacheLife('minutes')  // Minutes
cacheLife('hours')    // 1 hour
cacheLife('days')     // 1 day
cacheLife('weeks')    // 1 week
cacheLife('max')      // Maximum duration

// Custom configuration
cacheLife({
  stale: 3600,      // 1 hour until considered stale
  revalidate: 7200, // 2 hours until revalidated
  expire: 86400,    // 1 day until expired
})
```

### Suspense for Dynamic Content

Wrap dynamic content that needs request-time data in `<Suspense>`.

```tsx
import { Suspense } from 'react'
import { cookies } from 'next/headers'

export default function Page() {
  return (
    <>
      <h1>Part of static shell</h1>
      <Suspense fallback={<p>Loading preferences...</p>}>
        <UserPreferences />
      </Suspense>
    </>
  )
}

// This component accesses runtime data - must be in Suspense
async function UserPreferences() {
  const theme = (await cookies()).get('theme')?.value || 'light'
  return <div>Your theme: {theme}</div>
}
```

### Runtime Data That Requires Suspense

These APIs require `<Suspense>` wrapping:

- `cookies()` - User's cookie data
- `headers()` - Request headers
- `searchParams` - URL query parameters
- `params` - Dynamic route parameters (without `generateStaticParams`)

### Tagging and Revalidating

Use `cacheTag` for on-demand revalidation:

```tsx
import { cacheTag, revalidateTag, updateTag } from 'next/cache'

// In cached function
async function getCart() {
  'use cache'
  cacheTag('cart')
  return db.cart.findFirst({ where: { userId: getCurrentUserId() } })
}

// In server action - use updateTag for immediate refresh
export async function updateCart(itemId: string) {
  'use server'
  await db.cart.update({ /* ... */ })
  updateTag('cart') // Expires and refreshes in same request
}

// In server action - use revalidateTag for stale-while-revalidate
export async function createPost(data: FormData) {
  'use server'
  await db.posts.create({ /* ... */ })
  revalidateTag('posts', 'max') // Marks as stale, revalidates in background
}
```

### Migration from Route Segment Configs

| Old Config | New Approach |
|-----------|-------------|
| `dynamic = 'force-dynamic'` | Not needed (all pages dynamic by default) |
| `dynamic = 'force-static'` | Use `'use cache'` with `cacheLife('max')` |
| `revalidate = 3600` | Use `cacheLife('hours')` |
| `fetchCache` | Not needed with `'use cache'` |
| `runtime = 'edge'` | Not supported with Cache Components |

```tsx
// Before (Next.js 15)
export const revalidate = 3600

export default async function Page() {
  const data = await fetch('...')
  return <div>...</div>
}

// After (Next.js 16)
import { cacheLife } from 'next/cache'

export default async function Page() {
  'use cache'
  cacheLife('hours')
  
  const data = await fetch('...')
  return <div>...</div>
}
```

### Complete Cache Components Example

```tsx
import { Suspense } from 'react'
import { cookies } from 'next/headers'
import { cacheLife } from 'next/cache'

export default function BlogPage() {
  return (
    <>
      {/* Static - prerendered automatically */}
      <header>
        <h1>Our Blog</h1>
        <nav>Home | About</nav>
      </header>

      {/* Cached - included in static shell */}
      <BlogPosts />

      {/* Dynamic - streams at request time */}
      <Suspense fallback={<p>Loading preferences...</p>}>
        <UserPreferences />
      </Suspense>
    </>
  )
}

async function BlogPosts() {
  'use cache'
  cacheLife('hours')
  
  const posts = await fetch('https://api.example.com/posts')
  return <ul>{/* render posts */}</ul>
}

async function UserPreferences() {
  const theme = (await cookies()).get('theme')?.value || 'light'
  return <aside>Your theme: {theme}</aside>
}
```

---

## 4. Data Fetching Patterns

### Server Component Data Fetching

**Preferred approach**: Fetch data in Server Components.

```tsx
// With fetch API
export default async function Page() {
  const res = await fetch('https://api.example.com/posts')
  const posts = await res.json()
  
  return (
    <ul>
      {posts.map(post => <li key={post.id}>{post.title}</li>)}
    </ul>
  )
}

// With ORM/Database
import { db } from '@/lib/db'

export default async function Page() {
  const posts = await db.post.findMany()
  
  return (
    <ul>
      {posts.map(post => <li key={post.id}>{post.title}</li>)}
    </ul>
  )
}
```

### Client Component Data Fetching

Use React's `use` hook for streaming data to Client Components:

```tsx
// app/page.tsx - Server Component
import Posts from '@/app/ui/posts'
import { Suspense } from 'react'
import { getPosts } from '@/lib/data'

export default function Page() {
  const posts = getPosts() // Don't await - pass promise
  
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <Posts posts={posts} />
    </Suspense>
  )
}

// app/ui/posts.tsx - Client Component
'use client'

import { use } from 'react'

export default function Posts({ 
  posts 
}: { 
  posts: Promise<Post[]> 
}) {
  const allPosts = use(posts) // Unwrap promise
  
  return (
    <ul>
      {allPosts.map(post => <li key={post.id}>{post.title}</li>)}
    </ul>
  )
}
```

### Streaming with `loading.js`

Create a `loading.js` file to stream the entire page:

```tsx
// app/dashboard/loading.tsx
export default function Loading() {
  return (
    <div className="animate-pulse">
      <div className="h-8 bg-gray-200 rounded w-1/4 mb-4" />
      <div className="h-64 bg-gray-200 rounded" />
    </div>
  )
}
```

### Granular Streaming with Suspense

```tsx
import { Suspense } from 'react'

export default function Page() {
  return (
    <div>
      {/* Sent immediately */}
      <header>
        <h1>Welcome to the Blog</h1>
      </header>
      
      <main>
        {/* Streams when ready */}
        <Suspense fallback={<PostsSkeleton />}>
          <BlogPosts />
        </Suspense>
      </main>
    </div>
  )
}
```

### Sequential vs Parallel Data Fetching

**Sequential** (when data depends on previous request):

```tsx
export default async function Page({ params }: { params: Promise<{ username: string }> }) {
  const { username } = await params
  
  // Sequential - playlists depend on artist ID
  const artist = await getArtist(username)
  
  return (
    <>
      <h1>{artist.name}</h1>
      <Suspense fallback={<div>Loading playlists...</div>}>
        <Playlists artistId={artist.id} />
      </Suspense>
    </>
  )
}
```

**Parallel** (independent requests):

```tsx
export default async function Page({ params }: { params: Promise<{ username: string }> }) {
  const { username } = await params
  
  // Parallel - start both requests immediately
  const artistPromise = getArtist(username)
  const albumsPromise = getAlbums(username)
  
  const [artist, albums] = await Promise.all([artistPromise, albumsPromise])
  
  return (
    <>
      <h1>{artist.name}</h1>
      <Albums list={albums} />
    </>
  )
}
```

### Request Deduplication

Use React's `cache` for ORM/database queries:

```tsx
import { cache } from 'react'
import { db } from '@/lib/db'

export const getPost = cache(async (id: string) => {
  return db.post.findUnique({ where: { id } })
})

// Called multiple times in render tree - only executes once
// Component A
const post = await getPost('123')
// Component B
const samePost = await getPost('123') // Deduped!
```

### Preloading Data

Start loading data before it's needed:

```tsx
import { getItem, checkIsAvailable } from '@/lib/data'

export default async function Page({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params
  
  // Start loading item data immediately
  preload(id)
  
  // Do other async work
  const isAvailable = await checkIsAvailable()
  
  return isAvailable ? <Item id={id} /> : null
}

const preload = (id: string) => {
  void getItem(id) // void prevents awaiting
}
```

---

## 5. Server Actions (Mutations)

### Creating Server Functions

Use `'use server'` directive at function or file level:

```tsx
// File-level - all exports are server functions
// app/actions.ts
'use server'

export async function createPost(formData: FormData) {
  const title = formData.get('title')
  // ...
}

export async function deletePost(id: string) {
  // ...
}

// Function-level - inline in Server Components
// app/page.tsx
export default function Page() {
  async function submitForm(formData: FormData) {
    'use server'
    // ...
  }
  
  return <form action={submitForm}>...</form>
}
```

### Form Handling

```tsx
// app/ui/form.tsx
import { createPost } from '@/app/actions'

export function PostForm() {
  return (
    <form action={createPost}>
      <input type="text" name="title" required />
      <textarea name="content" required />
      <button type="submit">Create</button>
    </form>
  )
}

// app/actions.ts
'use server'

import { revalidatePath } from 'next/cache'
import { redirect } from 'next/navigation'

export async function createPost(formData: FormData) {
  const title = formData.get('title') as string
  const content = formData.get('content') as string
  
  await db.post.create({ data: { title, content } })
  
  revalidatePath('/posts')
  redirect('/posts')
}
```

### Pending States with useActionState

```tsx
'use client'

import { useActionState, startTransition } from 'react'
import { createPost } from '@/app/actions'

const initialState = { message: '', errors: {} }

export function PostForm() {
  const [state, action, pending] = useActionState(createPost, initialState)
  
  return (
    <form action={action}>
      <input type="text" name="title" />
      {state.errors?.title && <p className="error">{state.errors.title}</p>}
      
      <textarea name="content" />
      {state.errors?.content && <p className="error">{state.errors.content}</p>}
      
      <button disabled={pending}>
        {pending ? 'Creating...' : 'Create Post'}
      </button>
      
      {state.message && <p aria-live="polite">{state.message}</p>}
    </form>
  )
}
```

### Event Handler Invocation

```tsx
'use client'

import { useState } from 'react'
import { incrementLike } from '@/app/actions'

export function LikeButton({ initialLikes }: { initialLikes: number }) {
  const [likes, setLikes] = useState(initialLikes)
  
  return (
    <button
      onClick={async () => {
        const newLikes = await incrementLike()
        setLikes(newLikes)
      }}
    >
      {likes} Likes
    </button>
  )
}
```

### Revalidation Strategies

```tsx
'use server'

import { revalidatePath, revalidateTag } from 'next/cache'
import { redirect } from 'next/navigation'

export async function updatePost(id: string, formData: FormData) {
  await db.post.update({ where: { id }, data: { /* ... */ } })
  
  // Option 1: Revalidate specific path
  revalidatePath('/posts')
  
  // Option 2: Revalidate by tag
  revalidateTag('posts')
  
  // Option 3: Revalidate and redirect
  revalidatePath('/posts')
  redirect('/posts')
}
```

### Refresh Current Page

```tsx
'use server'

import { refresh } from 'next/cache'

export async function updateUserPreferences(formData: FormData) {
  await db.preferences.update({ /* ... */ })
  
  // Refresh current page to reflect changes
  refresh()
}
```

### Cookie Handling

```tsx
'use server'

import { cookies } from 'next/headers'

export async function setTheme(theme: string) {
  const cookieStore = await cookies()
  
  // Set cookie
  cookieStore.set('theme', theme, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    maxAge: 60 * 60 * 24 * 365, // 1 year
  })
}

export async function getTheme() {
  const cookieStore = await cookies()
  return cookieStore.get('theme')?.value || 'light'
}
```

---

## 6. Error Handling

### Expected Errors (Return Values)

Model expected errors as return values, not thrown exceptions.

```tsx
// app/actions.ts
'use server'

type ActionResult = 
  | { ok: true; data: Post }
  | { ok: false; error: string }

export async function createPost(formData: FormData): Promise<ActionResult> {
  const session = await getSession()
  if (!session) {
    return { ok: false, error: 'Please sign in to create a post' }
  }
  
  const result = PostSchema.safeParse({
    title: formData.get('title'),
    content: formData.get('content'),
  })
  
  if (!result.success) {
    return { ok: false, error: 'Invalid input' }
  }
  
  const post = await db.post.create({ data: result.data })
  return { ok: true, data: post }
}
```

### Using useActionState for Expected Errors

```tsx
'use client'

import { useActionState } from 'react'
import { createPost } from '@/app/actions'

const initialState = { message: '' }

export function Form() {
  const [state, action, pending] = useActionState(createPost, initialState)
  
  return (
    <form action={action}>
      <input type="text" name="title" />
      <textarea name="content" />
      {state?.message && <p aria-live="polite">{state.message}</p>}
      <button disabled={pending}>Create</button>
    </form>
  )
}
```

### Server Component Error Handling

```tsx
export default async function Page() {
  const res = await fetch('https://api.example.com/data')
  
  if (!res.ok) {
    return <div>There was an error loading the data.</div>
  }
  
  const data = await res.json()
  return <DataDisplay data={data} />
}
```

### Not Found Handling

```tsx
import { notFound } from 'next/navigation'

export default async function Page({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params
  const post = await getPost(slug)
  
  if (!post) {
    notFound()
  }
  
  return <article>{post.content}</article>
}

// app/blog/[slug]/not-found.tsx
export default function NotFound() {
  return (
    <div>
      <h2>Post Not Found</h2>
      <p>The requested post could not be found.</p>
    </div>
  )
}
```

### Error Boundaries with error.js

```tsx
// app/dashboard/error.tsx
'use client' // Error boundaries must be Client Components

import { useEffect } from 'react'

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  useEffect(() => {
    // Log to error reporting service
    console.error(error)
  }, [error])
  
  return (
    <div>
      <h2>Something went wrong!</h2>
      <button onClick={() => reset()}>Try again</button>
    </div>
  )
}
```

### Global Error Boundary

```tsx
// app/global-error.tsx
'use client'

export default function GlobalError({
  error,
  reset,
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  return (
    <html>
      <body>
        <h2>Something went wrong!</h2>
        <button onClick={() => reset()}>Try again</button>
      </body>
    </html>
  )
}
```

### Event Handler Error Handling

Error boundaries don't catch errors in event handlers. Handle manually:

```tsx
'use client'

import { useState } from 'react'

export function Button() {
  const [error, setError] = useState<Error | null>(null)
  
  const handleClick = async () => {
    try {
      await riskyOperation()
    } catch (err) {
      setError(err as Error)
    }
  }
  
  if (error) {
    return <div>Error: {error.message}</div>
  }
  
  return <button onClick={handleClick}>Click me</button>
}
```

---

## 7. Project Structure

### File Conventions

| File | Purpose |
|------|---------|
| `layout.tsx` | Shared UI for segment and children |
| `page.tsx` | Unique UI for route, makes route accessible |
| `loading.tsx` | Loading UI (wraps page in Suspense) |
| `error.tsx` | Error UI (React error boundary) |
| `not-found.tsx` | Not found UI |
| `route.ts` | API endpoint |
| `template.tsx` | Re-rendered layout |
| `default.tsx` | Parallel route fallback |

### Component Hierarchy

Components render in this order:

1. `layout.tsx`
2. `template.tsx`
3. `error.tsx` (error boundary)
4. `loading.tsx` (suspense boundary)
5. `not-found.tsx` (error boundary)
6. `page.tsx` or nested `layout.tsx`

### Private Folders

Prefix with underscore to opt out of routing:

```
app/
├── dashboard/
│   ├── page.tsx
│   ├── _components/     # Private - not routable
│   │   ├── chart.tsx
│   │   └── table.tsx
│   └── _lib/            # Private - not routable
│       └── utils.ts
```

### Route Groups

Use parentheses to organize without affecting URL:

```
app/
├── (marketing)/         # URL: /
│   ├── page.tsx
│   ├── about/
│   │   └── page.tsx    # URL: /about
│   └── layout.tsx      # Marketing layout
├── (shop)/              # URL: /
│   ├── cart/
│   │   └── page.tsx    # URL: /cart
│   └── layout.tsx      # Shop layout
```

### Dynamic Routes

```
app/
├── blog/
│   └── [slug]/          # /blog/my-post
│       └── page.tsx
├── shop/
│   └── [...slug]/       # /shop/a, /shop/a/b, /shop/a/b/c
│       └── page.tsx
├── docs/
│   └── [[...slug]]/     # /docs, /docs/a, /docs/a/b
│       └── page.tsx
```

### Accessing Dynamic Params

```tsx
// Next.js 16: params is a Promise
export default async function Page({
  params,
}: {
  params: Promise<{ slug: string }>
}) {
  const { slug } = await params
  return <div>Post: {slug}</div>
}
```

### Recommended Project Organization

**Option 1: Colocate in app directory**

```
app/
├── (routes)/
│   ├── dashboard/
│   │   ├── page.tsx
│   │   ├── _components/
│   │   └── _lib/
│   └── settings/
│       ├── page.tsx
│       └── _lib/
├── components/          # Shared components
├── lib/                 # Shared utilities
└── types/              # Shared types
```

**Option 2: Keep outside app directory**

```
src/
├── app/                 # Only routing files
│   ├── dashboard/
│   │   └── page.tsx
│   └── settings/
│       └── page.tsx
├── components/          # All components
├── lib/                 # Utilities
└── types/              # Types
```

---

## 8. Data Security

### Data Access Layer (DAL)

Create a centralized layer for all data access:

```tsx
// lib/dal/index.ts
import 'server-only'

export * from './users'
export * from './posts'
export * from './auth'

// lib/dal/auth.ts
import 'server-only'
import { cache } from 'react'
import { cookies } from 'next/headers'

export const getCurrentUser = cache(async () => {
  const token = (await cookies()).get('AUTH_TOKEN')?.value
  if (!token) return null
  
  const decoded = await verifyToken(token)
  // Return only safe, public data
  return {
    id: decoded.id,
    name: decoded.name,
    role: decoded.role,
  }
})
```

### Data Transfer Objects (DTOs)

Only return necessary, safe data:

```tsx
// lib/dal/users.ts
import 'server-only'
import { getCurrentUser } from './auth'

function canSeeEmail(viewer: User, target: User) {
  return viewer.id === target.id || viewer.role === 'admin'
}

export async function getUserProfile(userId: string) {
  const viewer = await getCurrentUser()
  if (!viewer) throw new Error('Unauthorized')
  
  const user = await db.user.findUnique({ where: { id: userId } })
  if (!user) return null
  
  // Return only what viewer is allowed to see
  return {
    id: user.id,
    name: user.name,
    email: canSeeEmail(viewer, user) ? user.email : null,
    // Never include: passwordHash, internalNotes, etc.
  }
}
```

### Server Action Security

Always validate input and verify authorization:

```tsx
'use server'

import { z } from 'zod'
import { getCurrentUser } from '@/lib/dal'

const UpdatePostSchema = z.object({
  id: z.string().uuid(),
  title: z.string().min(1).max(100),
  content: z.string().min(1),
})

export async function updatePost(formData: FormData) {
  // 1. Verify authentication
  const user = await getCurrentUser()
  if (!user) {
    return { ok: false, error: 'Unauthorized' }
  }
  
  // 2. Validate input with Zod
  const result = UpdatePostSchema.safeParse({
    id: formData.get('id'),
    title: formData.get('title'),
    content: formData.get('content'),
  })
  
  if (!result.success) {
    return { ok: false, errors: result.error.flatten() }
  }
  
  // 3. Verify authorization
  const post = await db.post.findUnique({ where: { id: result.data.id } })
  if (post?.authorId !== user.id) {
    return { ok: false, error: 'Forbidden' }
  }
  
  // 4. Perform mutation
  await db.post.update({
    where: { id: result.data.id },
    data: { title: result.data.title, content: result.data.content },
  })
  
  return { ok: true }
}
```

### Never Trust Client Input

```tsx
// BAD: Trusting searchParams
export default async function Page({ searchParams }) {
  const params = await searchParams
  if (params.isAdmin === 'true') {
    return <AdminPanel /> // Vulnerable!
  }
}

// GOOD: Verify server-side
import { getCurrentUser } from '@/lib/dal'

export default async function Page() {
  const user = await getCurrentUser()
  
  if (user?.role === 'admin') {
    return <AdminPanel />
  }
  
  return <UserDashboard />
}
```

### Environment Variables

- Server-only variables: `API_KEY`, `DATABASE_URL`
- Public variables: `NEXT_PUBLIC_API_URL` (exposed to client)

```tsx
// Server Component - safe
const apiKey = process.env.API_KEY

// Client Component - only NEXT_PUBLIC_ vars available
const publicUrl = process.env.NEXT_PUBLIC_API_URL
```

### Preventing Side Effects During Rendering

Never mutate data during rendering:

```tsx
// BAD: Mutation during render
export default async function Page({ searchParams }) {
  const params = await searchParams
  if (params.logout) {
    (await cookies()).delete('AUTH_TOKEN') // Don't do this!
  }
  return <UserProfile />
}

// GOOD: Use Server Actions
import { logout } from './actions'

export default function Page() {
  return (
    <>
      <UserProfile />
      <form action={logout}>
        <button type="submit">Logout</button>
      </form>
    </>
  )
}
```

### CSRF Protection

Next.js provides built-in protection:

1. Server Actions only accept `POST` requests
2. Origin header is compared to Host header
3. Configure allowed origins for reverse proxies:

```ts
// next.config.ts
const nextConfig = {
  serverActions: {
    allowedOrigins: ['my-proxy.com', '*.my-proxy.com'],
  },
}
```

---

## 9. Quick Reference Checklist

Use this checklist when reviewing Next.js 16 code:

### Components

- [ ] Server Components are the default (no unnecessary `'use client'`)
- [ ] `'use client'` only on components that need state/effects/browser APIs
- [ ] Props passed to Client Components are serializable
- [ ] Third-party client components are wrapped
- [ ] `server-only` package used for sensitive server code

### Data Fetching

- [ ] Data fetched in Server Components when possible
- [ ] `<Suspense>` used for streaming
- [ ] `loading.tsx` provided for route-level loading states
- [ ] React `cache()` used for ORM/database query deduplication
- [ ] Parallel fetching with `Promise.all` when possible

### Cache Components (if enabled)

- [ ] `cacheComponents: true` in next.config
- [ ] `'use cache'` directive used for cacheable data
- [ ] `cacheLife` specified for all cached functions
- [ ] `<Suspense>` wraps runtime data access (cookies, headers, searchParams)
- [ ] `cacheTag`/`revalidateTag` used for cache invalidation

### Server Actions

- [ ] All actions validate input with Zod
- [ ] Authentication checked in every action
- [ ] Authorization verified before mutations
- [ ] `revalidatePath`/`revalidateTag` called after mutations
- [ ] Expected errors returned as values, not thrown

### Error Handling

- [ ] `error.tsx` provided for route error boundaries
- [ ] `not-found.tsx` provided where needed
- [ ] Expected errors use return values
- [ ] Event handler errors caught manually
- [ ] Error messages don't leak sensitive information

### Project Structure

- [ ] Private folders prefixed with `_`
- [ ] Route groups `()` used for organization
- [ ] Related code colocated with routes
- [ ] Shared code in `lib/` or `components/`
- [ ] Types defined with Zod schemas

### Security

- [ ] Data Access Layer pattern implemented
- [ ] DTOs used (no raw database objects to client)
- [ ] No mutations during rendering
- [ ] Client input never trusted
- [ ] Environment variables properly scoped
- [ ] `params` and `searchParams` awaited (Next.js 16)

---

## Version History

| Version | Next.js | Date | Changes |
|---------|---------|------|---------|
| 1.0 | 16.0.5 | Nov 2025 | Initial release |

---

*This document should be reviewed whenever Next.js releases major updates. Always verify against the [official documentation](https://nextjs.org/docs).*

