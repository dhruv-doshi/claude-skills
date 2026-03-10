## App Router & File Conventions

```
app/
├── layout.tsx          # Root layout — must include <html> and <body>
├── page.tsx            # Route page
├── loading.tsx         # Suspense boundary (auto-wraps page)
├── error.tsx           # Error boundary (must be 'use client')
├── not-found.tsx       # 404 page
├── route.ts            # API route handler
├── (group)/            # Route group — organizes without affecting URL
├── [slug]/             # Dynamic segment
├── [...slug]/          # Catch-all segment
└── @slot/              # Parallel route slot
```

## Server vs Client Components

**Server Components (default)** — run on the server, zero JS sent to client:
- Can `async/await` directly, access DB, secrets, file system
- Cannot use hooks, event handlers, or browser APIs

**Client Components** — add `'use client'` directive:
- Required for interactivity, `useState`, `useEffect`, browser APIs
- Still SSR-rendered, then hydrated on client

```tsx
// Server Component — fetch directly, no boilerplate
async function ProductPage({ params }: { params: { id: string } }) {
  const product = await db.product.findUnique({ where: { id: params.id } })
  return <ProductDetails product={product} />
}

// Client Component
'use client'
function Counter() {
  const [count, setCount] = useState(0)
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>
}
```

**Key rule**: Push Client Components to the leaves. Pass Server Component output as `children` to Client Components — don't import Server Components inside Client ones.

## Layouts & Routing

```tsx
// Persistent layout — does not re-render on navigation
export default function DashboardLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="flex">
      <Sidebar />
      <main>{children}</main>
    </div>
  )
}

// Dynamic route with static params for SSG
export async function generateStaticParams() {
  const posts = await getPosts()
  return posts.map(post => ({ slug: post.slug }))
}

// Metadata
export const metadata: Metadata = { title: 'My App' }

// Dynamic metadata
export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const product = await getProduct(params.id)
  return { title: product.name }
}
```

## Data Fetching

### In Server Components

```tsx
// Fetch in parallel to avoid waterfalls
const [user, posts] = await Promise.all([getUser(id), getPosts(id)])

// fetch() with caching options
fetch(url, { cache: 'force-cache' })        // Cache indefinitely (SSG)
fetch(url, { cache: 'no-store' })           // Never cache (SSR, default in Next 15)
fetch(url, { next: { revalidate: 60 } })    // ISR — revalidate every 60s
fetch(url, { next: { tags: ['products'] } }) // Tag for on-demand revalidation

// Route-level config
export const revalidate = 60               // Revalidate entire route every 60s
export const dynamic = 'force-dynamic'    // Always SSR
```

### Server Actions (Mutations)

```tsx
// app/actions.ts
'use server'
import { revalidatePath, revalidateTag } from 'next/cache'

export async function createPost(prevState: unknown, formData: FormData) {
  const title = formData.get('title') as string
  // Validate, then write to DB
  await db.post.create({ data: { title } })
  revalidatePath('/posts')
  redirect('/posts')
}
```

```tsx
// Client Component using the Server Action
'use client'
import { useActionState } from 'react'
import { createPost } from '@/app/actions'

function CreatePostForm() {
  const [state, action, isPending] = useActionState(createPost, null)
  return (
    <form action={action}>
      <input name="title" />
      <button disabled={isPending}>{isPending ? 'Saving...' : 'Create'}</button>
      {state?.errors && <p>{state.errors.title}</p>}
    </form>
  )
}
```

### API Routes

```tsx
// app/api/posts/route.ts
export async function GET(request: NextRequest) {
  const posts = await db.post.findMany()
  return NextResponse.json(posts)
}

export async function POST(request: NextRequest) {
  const body = await request.json()
  const post = await db.post.create({ data: body })
  return NextResponse.json(post, { status: 201 })
}
```

## Performance

### Images

```tsx
import Image from 'next/image'

// Always use next/image — auto-optimizes, lazy loads, serves WebP/AVIF
<Image src="/hero.jpg" alt="Hero" width={1200} height={600} priority />
//                                                            ^ Add to LCP image

// Responsive fill
<div className="relative aspect-video">
  <Image src="/cover.jpg" alt="Cover" fill sizes="(max-width: 768px) 100vw, 50vw" className="object-cover" />
</div>
```

### Fonts

```tsx
// next/font — self-hosted, eliminates layout shift, no network request
import { Inter } from 'next/font/google'

const inter = Inter({ subsets: ['latin'], variable: '--font-inter' })

export default function RootLayout({ children }) {
  return <html className={inter.variable}><body className={inter.className}>{children}</body></html>
}
```

### Lazy Loading

```tsx
import dynamic from 'next/dynamic'

// Load heavy components on demand
const HeavyChart = dynamic(() => import('@/components/HeavyChart'), {
  loading: () => <Skeleton />,
  ssr: false,  // Disable for browser-only components
})
```

### Core Web Vitals Tips

- **LCP**: Add `priority` to the largest above-the-fold image
- **CLS**: Always set `width`/`height` on images; use `next/font`
- **INP**: Use `useTransition` for non-urgent updates; avoid large client bundles
- **Prefer SSG/ISR** over SSR where possible

## Styling

### Tailwind CSS (Recommended)

```bash
npm install tailwindcss @tailwindcss/postcss postcss
```

```css
/* app/globals.css */
@import "tailwindcss";

@theme {
  --color-brand: #3b82f6;
  --radius-card: 0.75rem;
}
```

### CSS Modules

```css
/* Button.module.css */
.button { padding: 0.5rem 1rem; border-radius: var(--radius); }
.primary { background-color: var(--color-brand); color: white; }
```

```tsx
import styles from './Button.module.css'
import { cn } from '@/lib/utils'

export function Button({ variant = 'primary', className, ...props }) {
  return <button className={cn(styles.button, styles[variant], className)} {...props} />
}
```

### Utility Setup

```tsx
// lib/utils.ts — merge Tailwind classes safely
import { clsx, type ClassValue } from 'clsx'
import { twMerge } from 'tailwind-merge'

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

### Dark Mode (next-themes)

```tsx
// app/layout.tsx
import { ThemeProvider } from 'next-themes'

export default function RootLayout({ children }) {
  return (
    <html suppressHydrationWarning>
      <body>
        <ThemeProvider attribute="class" defaultTheme="system" enableSystem>
          {children}
        </ThemeProvider>
      </body>
    </html>
  )
}
```