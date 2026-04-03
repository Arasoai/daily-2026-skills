```markdown
---
name: boneyard-skeleton-loading
description: Auto-generated pixel-perfect skeleton loading screens extracted from real DOM using boneyard-js
triggers:
  - add skeleton loading to my component
  - generate skeleton screens automatically
  - show loading placeholder while data fetches
  - use boneyard for skeleton UI
  - pixel perfect skeleton loading
  - auto generate loading skeletons
  - wrap component with skeleton loader
  - boneyard skeleton setup
---

# Boneyard Skeleton Loading

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Boneyard (`boneyard-js`) generates pixel-perfect skeleton loading screens by snapshotting your real DOM — no manual measurement or hand-tuned placeholders needed.

## How It Works

1. Wrap a component with `<Skeleton name="...">` and a `loading` prop
2. Run `npx boneyard-js build` — it launches your dev server, snapshots the DOM, and writes bones files
3. Import the generated registry once in your app root
4. Every `<Skeleton name="...">` auto-resolves its layout from the registry

## Installation

```bash
npm install boneyard-js
```

## Quick Setup

### 1. Wrap your component

```tsx
import { Skeleton } from 'boneyard-js/react'

function BlogPage() {
  const { data, isLoading } = useFetch('/api/post')

  return (
    <Skeleton name="blog-card" loading={isLoading}>
      {data && <BlogCard data={data} />}
    </Skeleton>
  )
}
```

### 2. Build the bones

```bash
npx boneyard-js build
# or specify your dev server URL
npx boneyard-js build http://localhost:3000
```

### 3. Import the registry once (e.g. in `app/layout.tsx`)

```tsx
import './bones/registry'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  )
}
```

## Component API

### `<Skeleton>` Props

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `name` | `string` | required | Unique identifier matching a built bones file |
| `loading` | `boolean` | required | When `true`, renders skeleton; when `false`, renders children |
| `color` | `string` | `#e0e0e0` | Fill color for bones (hex, rgb, etc.) |
| `animate` | `boolean` | `true` | Enable pulse shimmer animation |
| `snapshotConfig` | `object` | — | Control which elements are captured |

## CLI Commands

```bash
# Auto-detect dev server and build all skeletons
npx boneyard-js build

# Target a specific dev server URL
npx boneyard-js build http://localhost:3000

# Custom breakpoints and output directory
npx boneyard-js build --breakpoints 390,820,1440 --out ./public/bones

# Default breakpoints: 375px, 768px, 1280px
```

## Code Examples

### Basic data-fetching skeleton

```tsx
import { Skeleton } from 'boneyard-js/react'

function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState(null)
  const [loading, setLoading] = useState(true)

  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(data => {
        setUser(data)
        setLoading(false)
      })
  }, [userId])

  return (
    <Skeleton name="user-profile" loading={loading}>
      {user && (
        <div className="profile">
          <img src={user.avatar} alt={user.name} />
          <h2>{user.name}</h2>
          <p>{user.bio}</p>
        </div>
      )}
    </Skeleton>
  )
}
```

### With React Query

```tsx
import { Skeleton } from 'boneyard-js/react'
import { useQuery } from '@tanstack/react-query'

function ProductCard({ productId }: { productId: string }) {
  const { data, isLoading } = useQuery({
    queryKey: ['product', productId],
    queryFn: () => fetch(`/api/products/${productId}`).then(r => r.json()),
  })

  return (
    <Skeleton name="product-card" loading={isLoading} color="#f0f0f0">
      {data && (
        <div className="product">
          <img src={data.image} alt={data.title} />
          <h3>{data.title}</h3>
          <span>${data.price}</span>
        </div>
      )}
    </Skeleton>
  )
}
```

### List of skeletons

```tsx
import { Skeleton } from 'boneyard-js/react'

function PostList() {
  const { data: posts, isLoading } = usePosts()

  // Show multiple skeleton placeholders while loading
  if (isLoading) {
    return (
      <div className="post-list">
        {Array.from({ length: 5 }).map((_, i) => (
          <Skeleton key={i} name="post-list-item" loading={true}>
            {null}
          </Skeleton>
        ))}
      </div>
    )
  }

  return (
    <div className="post-list">
      {posts.map(post => (
        <Skeleton key={post.id} name="post-list-item" loading={false}>
          <PostItem post={post} />
        </Skeleton>
      ))}
    </div>
  )
}
```

### Custom color and no animation

```tsx
<Skeleton
  name="dashboard-chart"
  loading={isLoading}
  color="#d4d4d4"
  animate={false}
>
  {data && <Chart data={data} />}
</Skeleton>
```

### Snapshot config — exclude certain elements

```tsx
<Skeleton
  name="nav-header"
  loading={isLoading}
  snapshotConfig={{
    exclude: ['button', '[data-no-skeleton]'],
    minWidth: 8,
    minHeight: 8,
  }}
>
  {data && <Header data={data} />}
</Skeleton>
```

## Project Structure After Build

```
bones/
  registry.ts          ← import this once in app root
  blog-card.bones.json
  user-profile.bones.json
  product-card.bones.json
```

Each `.bones.json` stores a flat array of `{ x, y, w, h, r }` rectangles captured at each breakpoint. The `<Skeleton>` component reads these at runtime and renders matching gray rectangles.

## Next.js App Router Integration

```tsx
// app/layout.tsx
import '../bones/registry'
import type { Metadata } from 'next'

export const metadata: Metadata = {
  title: 'My App',
}

export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  )
}
```

## Next.js Pages Router Integration

```tsx
// pages/_app.tsx
import '../bones/registry'
import type { AppProps } from 'next/app'

export default function App({ Component, pageProps }: AppProps) {
  return <Component {...pageProps} />
}
```

## Workflow for New Skeletons

1. Add `<Skeleton name="my-new-component" loading={...}>` around your component
2. Make sure your dev server is running (`npm run dev`)
3. Run `npx boneyard-js build`
4. Commit the updated `bones/` directory

## Common Patterns

### Conditional loading state with SWR

```tsx
import useSWR from 'swr'
import { Skeleton } from 'boneyard-js/react'

const fetcher = (url: string) => fetch(url).then(r => r.json())

function ArticleView({ slug }: { slug: string }) {
  const { data, isLoading } = useSWR(`/api/articles/${slug}`, fetcher)

  return (
    <Skeleton name="article-view" loading={isLoading}>
      {data && (
        <article>
          <h1>{data.title}</h1>
          <p className="byline">By {data.author}</p>
          <div dangerouslySetInnerHTML={{ __html: data.content }} />
        </article>
      )}
    </Skeleton>
  )
}
```

### Dashboard with multiple skeleton zones

```tsx
import { Skeleton } from 'boneyard-js/react'

function Dashboard() {
  const { stats, statsLoading } = useStats()
  const { chart, chartLoading } = useChartData()
  const { activity, activityLoading } = useActivity()

  return (
    <div className="dashboard">
      <Skeleton name="stats-row" loading={statsLoading}>
        {stats && <StatsRow stats={stats} />}
      </Skeleton>

      <Skeleton name="main-chart" loading={chartLoading}>
        {chart && <MainChart data={chart} />}
      </Skeleton>

      <Skeleton name="activity-feed" loading={activityLoading}>
        {activity && <ActivityFeed items={activity} />}
      </Skeleton>
    </div>
  )
}
```

## Troubleshooting

### Skeleton name not found / blank skeleton shows
- Run `npx boneyard-js build` — the bones file for that name hasn't been generated yet
- Ensure the `name` prop exactly matches the one used during build
- Confirm `bones/registry` is imported in your app root

### Build command can't find dev server
- Start your dev server first (`npm run dev`)
- Pass the URL explicitly: `npx boneyard-js build http://localhost:3000`

### Skeleton layout doesn't match real component
- Re-run `npx boneyard-js build` after making layout changes
- Bones are static snapshots — they must be regenerated after UI changes

### Skeleton shows at wrong breakpoint size
- Use `--breakpoints` to match your app's actual responsive breakpoints
  ```bash
  npx boneyard-js build --breakpoints 390,768,1280,1920
  ```

### Output directory issues
- Default output is `./bones` relative to project root
- Change with `--out`: `npx boneyard-js build --out ./src/bones`
- Update your registry import path accordingly

## Links

- [Homepage](https://boneyard.vercel.app/overview)
- [npm](https://www.npmjs.com/package/boneyard-js)
- [GitHub](https://github.com/0xGF/boneyard)
```
