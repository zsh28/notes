# Framer Motion — SSR-Safe Animation Reference

A complete reference for the animation system used in this project. Covers setup, patterns, and rules that can be carried into any Next.js App Router project.

---

## Table of Contents

1. [Provider Setup](#provider-setup)
2. [SSR Safety](#ssr-safety)
3. [Reusable Motion Components](#reusable-motion-components)
4. [Animation Patterns](#animation-patterns)
5. [Firework Burst Pattern](#firework-burst-pattern)
6. [Easing Reference](#easing-reference)
7. [CSS Fallbacks](#css-fallbacks)
8. [The Golden Rules](#the-golden-rules)

---

## Provider Setup

**File:** `src/app/providers.tsx`

```tsx
"use client"

import { LazyMotion, domAnimation, MotionConfig } from "framer-motion"

export function Providers({ children }) {
  return (
    <LazyMotion features={domAnimation}>
      <MotionConfig reducedMotion="user">
        {children}
      </MotionConfig>
    </LazyMotion>
  )
}
```

**File:** `src/app/layout.tsx`

```tsx
import { Providers } from "./providers"

export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  )
}
```

### Why LazyMotion

The default `motion` import bundles all Framer Motion features (~30KB gzipped). `LazyMotion` with `domAnimation` loads only what's needed for scroll, hover, tap, and exit animations (~18KB). That's ~12KB saved with no feature loss for typical marketing sites.

| Import | Size | Use when |
| --- | --- | --- |
| `motion` (default) | ~30KB | You need drag, layout animations |
| `domAnimation` | ~18KB | Scroll reveals, hover, tap, exit |
| `domMax` | ~30KB | Same as default — drag + layout |

### Why MotionConfig reducedMotion="user"

Without this, users who have enabled "Reduce Motion" in their OS settings still see full animations. `reducedMotion="user"` reads the `prefers-reduced-motion` media query and automatically disables all Framer Motion animations globally — no per-component handling needed.

Options:

```tsx
<MotionConfig reducedMotion="user">    // Respects OS setting (recommended)
<MotionConfig reducedMotion="always">  // Disables all animations (useful for testing)
<MotionConfig reducedMotion="never">   // Ignores OS setting (not recommended)
```

---

## SSR Safety

### The problem

Next.js App Router renders components on the server by default. Framer Motion relies on browser APIs (`requestAnimationFrame`, `IntersectionObserver`, `window`, etc.) that don't exist in Node.js. Naively importing `motion` in a Server Component will throw a runtime error.

On top of that, if a component sets `initial={{ opacity: 0 }}`, the server might render it invisible — meaning search crawlers see blank content.

### The solution: `framer-motion/client`

Import from `framer-motion/client` instead of `framer-motion` in any component that does **not** have `"use client"` at the top.

```tsx
// In a Server Component (no "use client") — safe
import * as motion from "framer-motion/client"

export function MySection() {
  return (
    <motion.div
      initial={{ opacity: 0, y: 52 }}
      whileInView={{ opacity: 1, y: 0 }}
      viewport={{ once: true }}
      transition={{ duration: 0.75 }}
    >
      Content here
    </motion.div>
  )
}
```

**What this does:**

- Acts as a Client Component boundary — Next.js treats this file as the split point between server and client
- On the **server**: renders in the `animate` state (fully visible, full opacity) — crawlers see all content
- On the **client**: hydrates and plays the animation from `initial` → `animate`

### When to use `"use client"` instead

Use `"use client"` (and import from `framer-motion` directly) when the component needs other client-only features: `useState`, `useEffect`, event handlers, etc.

```tsx
"use client"

import { useState } from "react"
import { motion } from "framer-motion"

export function InteractiveCard() {
  const [hovered, setHovered] = useState(false)
  // ...
}
```

### The two failure modes to avoid

| Failure | Cause | Fix |
| --- | --- | --- |
| Invisible content on page load | `initial={{ opacity: 0 }}` rendered server-side | Use `framer-motion/client` — server renders `animate` state |
| Content missing from DOM | `{isLoaded && <Content />}` conditional render | Never gate critical content on JS — always render in DOM |

---

## Reusable Motion Components

### FadeUp

**File:** `src/components/motion/fade-up.tsx`

Content flows upward as it enters the viewport. Use for headings, body text, cards, and CTA sections.

```tsx
import type { ReactNode } from "react"
import * as motion from "framer-motion/client"

type FadeUpProps = {
  children: ReactNode
  delay?: number
  className?: string
}

export function FadeUp({ children, delay = 0, className }: FadeUpProps) {
  return (
    <motion.div
      initial={{ opacity: 0, y: 52 }}
      whileInView={{ opacity: 1, y: 0 }}
      viewport={{ once: true, margin: "-80px" }}
      transition={{ duration: 0.75, delay, ease: [0.16, 1, 0.3, 1] }}
      className={className}
    >
      {children}
    </motion.div>
  )
}
```

**Usage:**

```tsx
// Simple reveal
<FadeUp>
  <h2>Section heading</h2>
</FadeUp>

// With delay (for staggered grids)
<FadeUp delay={0.1}><Card /></FadeUp>
<FadeUp delay={0.2}><Card /></FadeUp>
<FadeUp delay={0.3}><Card /></FadeUp>

// With className to pass layout styles
<FadeUp className="flex flex-col justify-center">
  <p>Content</p>
</FadeUp>
```

---

### SlideIn

**File:** `src/components/motion/slide-in.tsx`

Panels, images, and large cards slide in from the left or right. Creates a strong horizontal reveal — good for alternating image/text layouts.

```tsx
import type { ReactNode } from "react"
import * as motion from "framer-motion/client"

type SlideInProps = {
  children: ReactNode
  direction?: "left" | "right"
  delay?: number
  className?: string
}

export function SlideIn({ children, direction = "left", delay = 0, className }: SlideInProps) {
  const x = direction === "left" ? -80 : 80

  return (
    <motion.div
      initial={{ opacity: 0, x }}
      whileInView={{ opacity: 1, x: 0 }}
      viewport={{ once: true, margin: "-80px" }}
      transition={{ duration: 0.9, delay, ease: [0.16, 1, 0.3, 1] }}
      className={className}
    >
      {children}
    </motion.div>
  )
}
```

**Usage:**

```tsx
// Image slides in from the right
<SlideIn direction="right" delay={0.15}>
  <Image src="..." alt="..." />
</SlideIn>

// Alternating layout — image left, text right
<SlideIn direction="left"><Image /></SlideIn>
<FadeUp delay={0.1}><article>Text content</article></FadeUp>
```

---

## Animation Patterns

### 1. Basic scroll reveal

Every section gets a `FadeUp` on its heading. This is the baseline — even if nothing else on the page animates, section titles always flow up.

```tsx
<FadeUp>
  <h2>Section Title</h2>
  <p>Subtitle or description</p>
</FadeUp>
```

### 2. Staggered grid

Apply increasing `delay` values to grid items. The stagger makes cards feel like they cascade in rather than all appearing at once.

```tsx
// Cards
{items.map((item, i) => (
  <FadeUp key={item.id} delay={i * 0.1}>
    <Card>{item}</Card>
  </FadeUp>
))}

// Typical delays for 3-column grids: 0, 0.1, 0.2
// Typical delays for 6-item grids:   0, 0.08, 0.16, 0.24, 0.32, 0.4
```

### 3. Alternating image / text layout

In two-column sections, images slide in from the outer edge and text fades up. This creates directional tension that guides the eye across the layout.

```tsx
// Row 1: text left, image right
<FadeUp delay={0.1}><article>Text</article></FadeUp>
<SlideIn direction="right" delay={0.15}><Image /></SlideIn>

// Row 2: image left, text right
<SlideIn direction="left" delay={0.15}><Image /></SlideIn>
<FadeUp delay={0.1}><article>Text</article></FadeUp>
```

### 4. Staggered list items inside a section

When a section has a list of items inside a larger card, animate each row individually rather than the whole card. This is more interesting than a single block animation.

```tsx
import * as motion from "framer-motion/client"

{items.map((item, i) => (
  <motion.div
    key={item.id}
    initial={{ opacity: 0, y: 24 }}
    whileInView={{ opacity: 1, y: 0 }}
    viewport={{ once: true, margin: "-60px" }}
    transition={{ duration: 0.6, delay: i * 0.1, ease: [0.16, 1, 0.3, 1] }}
  >
    {item.content}
  </motion.div>
))}
```

### 5. Viewport margin tuning

The `margin` property on `viewport` controls when the animation fires relative to the viewport edge. Negative values trigger the animation before the element fully enters the screen — important for smooth reveals.

```tsx
viewport={{ once: true, margin: "-80px" }}   // Fires 80px before element enters — standard
viewport={{ once: true, margin: "-40px" }}   // Fires closer to viewport edge — for small items
viewport={{ once: true, margin: "-120px" }}  // Fires earlier — for large hero sections
```

---

## Firework Burst Pattern

A radial burst where all items appear to originate from a single central point and explode outward to their grid positions. Used for grid sections where you want maximum visual impact on scroll.

### How it works

1. The grid container uses Framer Motion `variants` with `staggerChildren` to orchestrate timing
2. Each card has a directional offset (`x`, `y`) pointing FROM its natural position TOWARD the grid centre
3. `scale: 0.25` makes every card start as a near-invisible dot — creating the illusion they all come from one point
4. Spring physics with slight overshoot gives the "explosive pop" feel

### Code

```tsx
import * as motion from "framer-motion/client"

// Container — controls stagger timing
const gridVariants = {
  hidden: {},
  show: {
    transition: { staggerChildren: 0.06 },
  },
}

// Directional offsets for a 6-card 3×2 grid
// Positive x = right, positive y = down
// Each card starts near centre (offset toward centre) and flies outward to x:0, y:0
const BURST_ORIGINS = [
  { x:  55, y:  35 },  // top-left     → flies left  + up
  { x:   0, y:  55 },  // top-centre   → flies straight up
  { x: -55, y:  35 },  // top-right    → flies right + up
  { x:  55, y: -35 },  // bottom-left  → flies left  + down
  { x:   0, y: -55 },  // bottom-centre → flies straight down
  { x: -55, y: -35 },  // bottom-right → flies right + down
]

export function BurstGrid({ items }) {
  return (
    <motion.div
      variants={gridVariants}
      initial="hidden"
      whileInView="show"
      viewport={{ once: true, margin: "-80px" }}
      className="grid gap-4 md:grid-cols-2 lg:grid-cols-3"
    >
      {items.map((item, i) => {
        const { x, y } = BURST_ORIGINS[i]

        const itemVariants = {
          hidden: { opacity: 0, scale: 0.25, x, y },
          show: {
            opacity: 1,
            scale: 1,
            x: 0,
            y: 0,
            transition: {
              type: "spring" as const,
              stiffness: 220,
              damping: 18,
            },
          },
        }

        return (
          <motion.div key={i} variants={itemVariants}>
            {item.content}
          </motion.div>
        )
      })}
    </motion.div>
  )
}
```

### Adapting for different grid sizes

For a **4-item 2×2 grid:**

```tsx
const BURST_ORIGINS = [
  { x:  50, y:  40 },  // top-left
  { x: -50, y:  40 },  // top-right
  { x:  50, y: -40 },  // bottom-left
  { x: -50, y: -40 },  // bottom-right
]
```

For a **3-item row:**

```tsx
const BURST_ORIGINS = [
  { x:  60, y:  0 },  // left
  { x:   0, y: 40 },  // centre (flies up)
  { x: -60, y:  0 },  // right
]
```

### Tuning the spring

```tsx
// Snappy with slight overshoot (default — good for cards)
{ type: "spring", stiffness: 220, damping: 18 }

// More dramatic overshoot (bouncier)
{ type: "spring", stiffness: 300, damping: 14 }

// Tight, no overshoot (professional feel)
{ type: "spring", stiffness: 200, damping: 25 }
```

---

## Easing Reference

All scroll reveal animations in this project use a single easing curve:

```
[0.16, 1, 0.3, 1]
```

This is an **expo-out** curve: fast initial movement that decelerates sharply to a smooth stop. It gives animations a "premium" feel — content arrives decisively then settles.

### Comparison

```
[0.25, 0.46, 0.45, 0.94]  — ease-out (softer, original default used before upgrade)
[0.16, 1, 0.3, 1]         — expo-out (faster start, sharper stop — current)
[0.4, 0, 0.2, 1]          — material design standard
"easeOut"                  — Framer Motion built-in (gentle, good default)
"easeInOut"                — symmetric, good for page transitions
```

### Duration guidelines

| Animation type | Duration |
| --- | --- |
| Text / headings | 0.75s |
| Images / large panels | 0.9s |
| Small cards in a grid | 0.6s |
| List items (stagger) | 0.6s |
| UI micro-interactions | 0.2–0.3s |

---

## CSS Fallbacks

Added to `src/app/globals.css`. These run independently of JavaScript and Framer Motion.

```css
/* Show all content when JS is disabled (crawlers, noscript environments) */
@media (scripting: none) {
  * {
    opacity: 1 !important;
    transform: none !important;
  }
}

/* CSS-level reduce-motion fallback (belt + suspenders alongside MotionConfig) */
@media (prefers-reduced-motion: reduce) {
  * {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
```

**Why both?** `MotionConfig reducedMotion="user"` handles Framer Motion animations. The CSS rule handles any CSS transitions or animations you add outside of Framer Motion.

---

## The Golden Rules

### 1. Content must always be in the DOM

Never use conditional rendering to hide content that should be indexed:

```tsx
// BAD — content absent from server HTML
{isLoaded && <h1>Important heading</h1>}

// GOOD — content always in DOM, only visual properties animate
<motion.div initial={{ opacity: 0 }} whileInView={{ opacity: 1 }}>
  <h1>Important heading</h1>
</motion.div>
```

### 2. Always use viewport={{ once: true }}

Without `once: true`, elements re-hide when they scroll out of view. Content disappearing when you scroll back is jarring and can confuse crawlers.

```tsx
// Always
viewport={{ once: true, margin: "-80px" }}
```

### 3. Only animate safe CSS properties

These properties affect visuals only — the HTML content is fully present regardless of their values:

- `opacity`
- `x` / `y` (transform: translate)
- `scale`
- `rotate`
- `filter` (blur)

Never use `display: none`, `visibility: hidden`, or `height: 0; overflow: hidden` to hide important content.

### 4. Use `framer-motion/client` in Server Components

```tsx
// Server Component — no "use client" at top
import * as motion from "framer-motion/client"   // correct
import { motion } from "framer-motion"            // wrong — will error
```

### 5. Test with JS disabled

Open DevTools → Settings → Debugger → Disable JavaScript, then reload the page. Every heading, paragraph, and link should be fully readable. If anything is missing or invisible, the animation setup is wrong.
