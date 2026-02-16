---
name: frontend-tailwind
description: Expert Tailwind CSS development with utility-first patterns, responsive design, component extraction, and performance. Use when styling frontend applications with Tailwind.
---

# Tailwind CSS Development

Expert guidance for building modern, responsive UIs with Tailwind CSS.

## Core Principles

- Utility-first: compose styles from utility classes directly in markup
- Extract components only when patterns repeat 3+ times
- Use design tokens (theme config) for consistency
- Mobile-first responsive design
- Prefer Tailwind utilities over custom CSS

## Responsive Design

Mobile-first breakpoints: `sm` (640px), `md` (768px), `lg` (1024px), `xl` (1280px), `2xl` (1536px).

```html
<!-- ✅ Good: Mobile-first, progressive enhancement -->
<div class="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4">
  <div class="p-4 text-sm sm:text-base lg:text-lg">
    Responsive content
  </div>
</div>

<!-- ✅ Good: Responsive spacing and layout -->
<section class="px-4 py-8 sm:px-6 lg:px-8 lg:py-12">
  <h1 class="text-2xl font-bold sm:text-3xl lg:text-4xl">Title</h1>
</section>
```

## Layout Patterns

### Flexbox

```html
<!-- Centered content -->
<div class="flex min-h-screen items-center justify-center">
  <div>Centered</div>
</div>

<!-- Navbar pattern -->
<nav class="flex items-center justify-between px-4 py-3">
  <div class="text-lg font-bold">Logo</div>
  <div class="flex items-center gap-4">
    <a href="#">Link</a>
    <button class="rounded-lg bg-blue-600 px-4 py-2 text-white">CTA</button>
  </div>
</nav>

<!-- Space between with gap -->
<div class="flex flex-col gap-4 sm:flex-row sm:gap-6">
  <div class="flex-1">Column 1</div>
  <div class="flex-1">Column 2</div>
</div>
```

### Grid

```html
<!-- Dashboard layout -->
<div class="grid grid-cols-1 gap-6 md:grid-cols-2 lg:grid-cols-3">
  <div class="rounded-lg border p-6 shadow-sm">Card 1</div>
  <div class="rounded-lg border p-6 shadow-sm">Card 2</div>
  <div class="rounded-lg border p-6 shadow-sm">Card 3</div>
</div>

<!-- Sidebar layout -->
<div class="grid min-h-screen grid-cols-1 lg:grid-cols-[280px_1fr]">
  <aside class="border-r bg-gray-50 p-4">Sidebar</aside>
  <main class="p-6">Content</main>
</div>
```

## Component Patterns

### Buttons

```html
<!-- Primary -->
<button class="rounded-lg bg-blue-600 px-4 py-2 text-sm font-medium text-white transition-colors hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-offset-2 disabled:cursor-not-allowed disabled:opacity-50">
  Primary
</button>

<!-- Secondary -->
<button class="rounded-lg border border-gray-300 bg-white px-4 py-2 text-sm font-medium text-gray-700 transition-colors hover:bg-gray-50 focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-offset-2">
  Secondary
</button>

<!-- Destructive -->
<button class="rounded-lg bg-red-600 px-4 py-2 text-sm font-medium text-white transition-colors hover:bg-red-700 focus:outline-none focus:ring-2 focus:ring-red-500 focus:ring-offset-2">
  Delete
</button>
```

### Cards

```html
<div class="overflow-hidden rounded-xl border border-gray-200 bg-white shadow-sm transition-shadow hover:shadow-md">
  <div class="p-6">
    <h3 class="text-lg font-semibold text-gray-900">Card Title</h3>
    <p class="mt-2 text-sm text-gray-600">Card description goes here.</p>
  </div>
  <div class="border-t bg-gray-50 px-6 py-3">
    <button class="text-sm font-medium text-blue-600 hover:text-blue-700">Action</button>
  </div>
</div>
```

### Forms

```html
<form class="space-y-4">
  <div>
    <label class="block text-sm font-medium text-gray-700" for="email">Email</label>
    <input
      id="email"
      type="email"
      class="mt-1 block w-full rounded-lg border border-gray-300 px-3 py-2 text-sm shadow-sm transition-colors focus:border-blue-500 focus:outline-none focus:ring-1 focus:ring-blue-500"
      placeholder="you@example.com"
    />
  </div>
  <div>
    <label class="block text-sm font-medium text-gray-700" for="message">Message</label>
    <textarea
      id="message"
      rows="4"
      class="mt-1 block w-full rounded-lg border border-gray-300 px-3 py-2 text-sm shadow-sm transition-colors focus:border-blue-500 focus:outline-none focus:ring-1 focus:ring-blue-500"
    ></textarea>
  </div>
  <button type="submit" class="w-full rounded-lg bg-blue-600 px-4 py-2 text-sm font-medium text-white hover:bg-blue-700">
    Submit
  </button>
</form>
```

## Dark Mode

Use the `dark:` variant. Configure with `darkMode: 'class'` or `darkMode: 'media'`.

```html
<div class="bg-white text-gray-900 dark:bg-gray-900 dark:text-gray-100">
  <h1 class="text-gray-900 dark:text-white">Title</h1>
  <p class="text-gray-600 dark:text-gray-400">Description</p>
  <div class="rounded-lg border border-gray-200 bg-gray-50 dark:border-gray-700 dark:bg-gray-800">
    Card content
  </div>
</div>
```

## Animations & Transitions

```html
<!-- Smooth hover transition -->
<button class="transform transition-all duration-200 hover:scale-105 hover:shadow-lg">
  Hover me
</button>

<!-- Fade in -->
<div class="animate-fade-in">Content</div>

<!-- Spin (loading) -->
<svg class="h-5 w-5 animate-spin text-blue-600">...</svg>

<!-- Pulse (skeleton) -->
<div class="h-4 w-3/4 animate-pulse rounded bg-gray-200"></div>
```

## Custom Theme Configuration

```javascript
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      colors: {
        brand: {
          50: "#eff6ff",
          500: "#3b82f6",
          600: "#2563eb",
          700: "#1d4ed8",
        },
      },
      fontFamily: {
        sans: ["Inter", "system-ui", "sans-serif"],
      },
      spacing: {
        18: "4.5rem",
      },
      animation: {
        "fade-in": "fadeIn 0.3s ease-in-out",
      },
      keyframes: {
        fadeIn: {
          "0%": { opacity: "0" },
          "100%": { opacity: "1" },
        },
      },
    },
  },
};
```

## Class Organization

Order classes consistently: layout → sizing → spacing → typography → colors → effects → states.

```html
<!-- ✅ Good: Consistent ordering -->
<div class="flex items-center gap-4 w-full max-w-md p-4 text-sm font-medium text-gray-900 bg-white rounded-lg shadow-sm hover:shadow-md transition-shadow">

<!-- Use clsx/cn for conditional classes -->
```

```typescript
import { clsx } from "clsx";
import { twMerge } from "tailwind-merge";

function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}

// Usage
<button className={cn(
  "rounded-lg px-4 py-2 text-sm font-medium transition-colors",
  variant === "primary" && "bg-blue-600 text-white hover:bg-blue-700",
  variant === "secondary" && "border border-gray-300 bg-white text-gray-700 hover:bg-gray-50",
  disabled && "cursor-not-allowed opacity-50"
)}>
```

## Common Mistakes

```html
<!-- ❌ Don't use arbitrary values when a utility exists -->
<div class="w-[100%]">  <!-- Use w-full -->
<div class="mt-[16px]"> <!-- Use mt-4 -->

<!-- ❌ Don't fight Tailwind with custom CSS -->
<style>.custom { margin-top: 1rem; }</style>

<!-- ✅ Use utilities -->
<div class="mt-4">

<!-- ❌ Don't forget focus states for accessibility -->
<button class="bg-blue-600 hover:bg-blue-700">

<!-- ✅ Include focus states -->
<button class="bg-blue-600 hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-offset-2">

<!-- ❌ Don't use too many breakpoints on one element -->
<div class="text-xs sm:text-sm md:text-base lg:text-lg xl:text-xl 2xl:text-2xl">

<!-- ✅ Keep it simple — 2-3 breakpoints max -->
<div class="text-sm md:text-base lg:text-lg">
```

## Performance

- Use `@apply` sparingly — it defeats the purpose of utility-first
- Purge unused styles in production (automatic with Tailwind 3+)
- Prefer `gap` over margin for spacing between flex/grid children
- Use `will-change-transform` only when needed for animation performance
