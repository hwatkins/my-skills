---
name: component-design-system
description: Practical architecture for building a consistent component library with Phoenix function components and Svelte components sharing a visual language. Covers variant APIs, slot patterns, design tokens in Tailwind, and cross-boundary consistency. Use when building reusable UI components, designing component APIs, or structuring a shared design system.
---

# Component Design System

Expert guidance for building a unified component library where Phoenix function components and LiveSvelte components share a consistent visual language, API conventions, and design tokens.

## Core Principles

- **One source of truth for tokens** — Tailwind config defines colors, spacing, typography, radii, shadows. Both Phoenix and Svelte components consume the same tokens.
- **Phoenix components for server-rendered UI** — buttons, badges, form inputs, layout primitives, flash messages, modals, tables.
- **Svelte components for interactive UI** — rich editors, data grids, charts, drag-and-drop, anything needing local reactive state.
- **Consistent API conventions** — same variant names, same size scale, same naming patterns across both worlds.
- **Composition over configuration** — prefer slots/snippets for flexible content over deeply nested option maps.

## Design Token System

### Tailwind as the Token Layer

Define your design tokens in `tailwind.config.js`. Both Phoenix templates and Svelte components consume them.

```javascript
// tailwind.config.js
export default {
  theme: {
    extend: {
      colors: {
        brand: {
          50: "#f0f5ff",
          100: "#e0ebff",
          500: "#3b82f6",
          600: "#2563eb",
          700: "#1d4ed8",
          900: "#1e3a5f",
        },
        surface: {
          DEFAULT: "var(--color-surface)",
          raised: "var(--color-surface-raised)",
          sunken: "var(--color-surface-sunken)",
        },
        destructive: {
          DEFAULT: "#ef4444",
          hover: "#dc2626",
        },
      },
      fontSize: {
        "2xs": ["0.625rem", { lineHeight: "0.875rem" }],
      },
      borderRadius: {
        DEFAULT: "0.5rem",
        pill: "9999px",
      },
      spacing: {
        18: "4.5rem",
        88: "22rem",
      },
      boxShadow: {
        "elevation-1": "0 1px 2px 0 rgb(0 0 0 / 0.05)",
        "elevation-2": "0 4px 6px -1px rgb(0 0 0 / 0.1)",
        "elevation-3": "0 10px 15px -3px rgb(0 0 0 / 0.1)",
      },
    },
  },
};
```

### CSS Custom Properties for Runtime Theming

For values that change at runtime (dark mode, tenant themes):

```css
/* assets/css/tokens.css */
:root {
  --color-surface: #ffffff;
  --color-surface-raised: #f9fafb;
  --color-surface-sunken: #f3f4f6;
  --color-text-primary: #111827;
  --color-text-secondary: #6b7280;
  --color-border: #e5e7eb;
  --radius-default: 0.5rem;
  --transition-fast: 150ms ease;
  --transition-normal: 200ms ease;
}

[data-theme="dark"] {
  --color-surface: #111827;
  --color-surface-raised: #1f2937;
  --color-surface-sunken: #0f172a;
  --color-text-primary: #f9fafb;
  --color-text-secondary: #9ca3af;
  --color-border: #374151;
}
```

### Token Consumption

```elixir
# Phoenix — tokens via Tailwind classes
~H"""
<div class="bg-surface text-text-primary rounded shadow-elevation-1">
  ...
</div>
"""
```

```svelte
<!-- Svelte — same Tailwind classes, same tokens -->
<div class="bg-surface text-text-primary rounded shadow-elevation-1">
  ...
</div>

<!-- Or CSS custom properties for computed styles -->
<div style="border-radius: var(--radius-default); transition: var(--transition-fast)">
  ...
</div>
```

## Phoenix Function Component Patterns

### The Variant + Size Pattern

The standard API for configurable components: `:variant` for visual style, `:size` for scale.

```elixir
attr :variant, :atom,
  values: [:primary, :secondary, :ghost, :destructive],
  default: :primary

attr :size, :atom,
  values: [:xs, :sm, :md, :lg],
  default: :md

attr :disabled, :boolean, default: false
attr :type, :string, default: "button"
attr :rest, :global, include: ~w(form)

slot :inner_block, required: true

def button(assigns) do
  ~H"""
  <button
    type={@type}
    disabled={@disabled}
    class={[
      "inline-flex items-center justify-center font-medium rounded transition-colors",
      "focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-brand-500 focus-visible:ring-offset-2",
      "disabled:pointer-events-none disabled:opacity-50",
      variant_classes(@variant),
      size_classes(@size)
    ]}
    {@rest}
  >
    {render_slot(@inner_block)}
  </button>
  """
end

defp variant_classes(:primary), do: "bg-brand-600 text-white hover:bg-brand-700"
defp variant_classes(:secondary), do: "bg-surface-raised text-text-primary border border-border hover:bg-surface-sunken"
defp variant_classes(:ghost), do: "text-text-secondary hover:bg-surface-raised hover:text-text-primary"
defp variant_classes(:destructive), do: "bg-destructive text-white hover:bg-destructive-hover"

defp size_classes(:xs), do: "h-7 px-2 text-xs gap-1"
defp size_classes(:sm), do: "h-8 px-3 text-sm gap-1.5"
defp size_classes(:md), do: "h-9 px-4 text-sm gap-2"
defp size_classes(:lg), do: "h-10 px-5 text-base gap-2"
```

### Usage

```heex
<.button>Save</.button>
<.button variant={:secondary} size={:sm}>Cancel</.button>
<.button variant={:destructive} phx-click="delete" data-confirm="Are you sure?">
  Delete
</.button>
<.button variant={:ghost} size={:xs} form="search-form">Search</.button>
```

### Icon Buttons with Slots

```elixir
attr :variant, :atom, values: [:primary, :secondary, :ghost, :destructive], default: :primary
attr :size, :atom, values: [:xs, :sm, :md, :lg], default: :md
attr :rest, :global

slot :icon_left
slot :icon_right
slot :inner_block, required: true

def button(assigns) do
  ~H"""
  <button
    class={[
      "inline-flex items-center justify-center font-medium rounded transition-colors",
      "focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-brand-500 focus-visible:ring-offset-2",
      "disabled:pointer-events-none disabled:opacity-50",
      variant_classes(@variant),
      size_classes(@size)
    ]}
    {@rest}
  >
    <span :if={@icon_left != []} class={icon_size(@size)}>
      {render_slot(@icon_left)}
    </span>
    {render_slot(@inner_block)}
    <span :if={@icon_right != []} class={icon_size(@size)}>
      {render_slot(@icon_right)}
    </span>
  </button>
  """
end

defp icon_size(:xs), do: "h-3 w-3"
defp icon_size(:sm), do: "h-3.5 w-3.5"
defp icon_size(:md), do: "h-4 w-4"
defp icon_size(:lg), do: "h-5 w-5"
```

```heex
<.button>
  <:icon_left><.icon name="hero-plus" /></:icon_left>
  Add Item
</.button>
```

### Named Slots for Complex Layouts

```elixir
slot :header
slot :inner_block, required: true
slot :footer

attr :padding, :atom, values: [:none, :sm, :md, :lg], default: :md
attr :rest, :global

def card(assigns) do
  ~H"""
  <div class={["bg-surface-raised rounded-lg border border-border shadow-elevation-1", @rest]}>
    <div :if={@header != []} class="border-b border-border px-4 py-3">
      {render_slot(@header)}
    </div>
    <div class={padding_classes(@padding)}>
      {render_slot(@inner_block)}
    </div>
    <div :if={@footer != []} class="border-t border-border px-4 py-3">
      {render_slot(@footer)}
    </div>
  </div>
  """
end

defp padding_classes(:none), do: ""
defp padding_classes(:sm), do: "p-3"
defp padding_classes(:md), do: "p-4"
defp padding_classes(:lg), do: "p-6"
```

```heex
<.card>
  <:header>
    <h3 class="text-lg font-semibold">Settings</h3>
  </:header>
  <p>Card content here.</p>
  <:footer>
    <.button size={:sm}>Save</.button>
  </:footer>
</.card>
```

### The Table Component with Slot Attributes

The canonical example of named slots with `:let`:

```elixir
slot :column, doc: "Table columns" do
  attr :label, :string, required: true
  attr :class, :string
end

attr :rows, :list, required: true
attr :row_click, :any, default: nil
attr :row_id, :any, default: nil

def data_table(assigns) do
  assigns = assign_new(assigns, :row_id, fn -> fn {_dom_id, row} -> row.id end end)

  ~H"""
  <table class="w-full text-sm">
    <thead class="border-b border-border">
      <tr>
        <th :for={col <- @column} class={["text-left font-medium text-text-secondary px-3 py-2", col[:class]]}>
          {col.label}
        </th>
      </tr>
    </thead>
    <tbody>
      <tr
        :for={row <- @rows}
        class={["border-b border-border last:border-0 hover:bg-surface-sunken", @row_click && "cursor-pointer"]}
        phx-click={@row_click && @row_click.(row)}
      >
        <td :for={col <- @column} class={["px-3 py-2", col[:class]]}>
          {render_slot(col, row)}
        </td>
      </tr>
    </tbody>
  </table>
  """
end
```

```heex
<.data_table rows={@users}>
  <:column :let={user} label="Name">{user.name}</:column>
  <:column :let={user} label="Email" class="text-text-secondary">{user.email}</:column>
  <:column :let={user} label="Role">
    <.badge variant={role_variant(user.role)}>{user.role}</.badge>
  </:column>
</.data_table>
```

### Badge Component

```elixir
attr :variant, :atom,
  values: [:default, :success, :warning, :error, :info],
  default: :default

attr :size, :atom, values: [:sm, :md], default: :sm
attr :rest, :global

slot :inner_block, required: true

def badge(assigns) do
  ~H"""
  <span
    class={[
      "inline-flex items-center font-medium rounded-full",
      badge_variant(@variant),
      badge_size(@size)
    ]}
    {@rest}
  >
    {render_slot(@inner_block)}
  </span>
  """
end

defp badge_variant(:default), do: "bg-gray-100 text-gray-700"
defp badge_variant(:success), do: "bg-green-100 text-green-700"
defp badge_variant(:warning), do: "bg-yellow-100 text-yellow-800"
defp badge_variant(:error), do: "bg-red-100 text-red-700"
defp badge_variant(:info), do: "bg-blue-100 text-blue-700"

defp badge_size(:sm), do: "px-2 py-0.5 text-xs"
defp badge_size(:md), do: "px-2.5 py-1 text-sm"
```

### Form Input with Error States

```elixir
attr :field, Phoenix.HTML.FormField, required: true
attr :type, :string, default: "text"
attr :label, :string, default: nil
attr :placeholder, :string, default: nil
attr :rest, :global

def input(assigns) do
  ~H"""
  <div>
    <label :if={@label} for={@field.id} class="block text-sm font-medium text-text-primary mb-1">
      {@label}
    </label>
    <input
      type={@type}
      id={@field.id}
      name={@field.name}
      value={@field.value}
      placeholder={@placeholder}
      class={[
        "w-full rounded border px-3 py-2 text-sm transition-colors",
        "focus:outline-none focus:ring-2 focus:ring-brand-500 focus:border-brand-500",
        @field.errors == [] && "border-border",
        @field.errors != [] && "border-red-500 focus:ring-red-500"
      ]}
      {@rest}
    />
    <p :for={error <- @field.errors} class="mt-1 text-xs text-red-600">
      {translate_error(error)}
    </p>
  </div>
  """
end
```

## Svelte Component Patterns

### Mirroring the Variant API

Svelte components should use the **same variant and size vocabulary** as Phoenix components.

```svelte
<!-- assets/svelte/Button.svelte -->
<script lang="ts">
  import type { Snippet } from "svelte";

  type Variant = "primary" | "secondary" | "ghost" | "destructive";
  type Size = "xs" | "sm" | "md" | "lg";

  let {
    variant = "primary",
    size = "md",
    disabled = false,
    onclick,
    children,
  }: {
    variant?: Variant;
    size?: Size;
    disabled?: boolean;
    onclick?: (e: MouseEvent) => void;
    children: Snippet;
  } = $props();

  const variantClasses: Record<Variant, string> = {
    primary: "bg-brand-600 text-white hover:bg-brand-700",
    secondary: "bg-surface-raised text-text-primary border border-border hover:bg-surface-sunken",
    ghost: "text-text-secondary hover:bg-surface-raised hover:text-text-primary",
    destructive: "bg-destructive text-white hover:bg-destructive-hover",
  };

  const sizeClasses: Record<Size, string> = {
    xs: "h-7 px-2 text-xs gap-1",
    sm: "h-8 px-3 text-sm gap-1.5",
    md: "h-9 px-4 text-sm gap-2",
    lg: "h-10 px-5 text-base gap-2",
  };
</script>

<button
  class="inline-flex items-center justify-center font-medium rounded transition-colors
    focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-brand-500 focus-visible:ring-offset-2
    disabled:pointer-events-none disabled:opacity-50
    {variantClasses[variant]} {sizeClasses[size]}"
  {disabled}
  {onclick}
>
  {@render children()}
</button>
```

### Svelte Snippets (Svelte 5 Slot Equivalent)

Svelte 5 uses `{#snippet}` and `{@render}` instead of `<slot>`:

```svelte
<!-- assets/svelte/Card.svelte -->
<script lang="ts">
  import type { Snippet } from "svelte";

  let {
    header,
    children,
    footer,
    padding = "md",
  }: {
    header?: Snippet;
    children: Snippet;
    footer?: Snippet;
    padding?: "none" | "sm" | "md" | "lg";
  } = $props();

  const paddingClasses = {
    none: "",
    sm: "p-3",
    md: "p-4",
    lg: "p-6",
  };
</script>

<div class="bg-surface-raised rounded-lg border border-border shadow-elevation-1">
  {#if header}
    <div class="border-b border-border px-4 py-3">
      {@render header()}
    </div>
  {/if}

  <div class={paddingClasses[padding]}>
    {@render children()}
  </div>

  {#if footer}
    <div class="border-t border-border px-4 py-3">
      {@render footer()}
    </div>
  {/if}
</div>
```

```svelte
<!-- Usage -->
<Card>
  {#snippet header()}
    <h3 class="text-lg font-semibold">Settings</h3>
  {/snippet}

  <p>Card content here.</p>

  {#snippet footer()}
    <Button size="sm">Save</Button>
  {/snippet}
</Card>
```

### Svelte Data Table with Snippet Props

The Svelte equivalent of Phoenix's table with slot attributes:

```svelte
<!-- assets/svelte/DataTable.svelte -->
<script lang="ts" generics="T">
  import type { Snippet } from "svelte";

  type Column<T> = {
    label: string;
    cell: Snippet<[T]>;
    class?: string;
  };

  let {
    rows,
    columns,
    onRowClick,
  }: {
    rows: T[];
    columns: Column<T>[];
    onRowClick?: (row: T) => void;
  } = $props();
</script>

<table class="w-full text-sm">
  <thead class="border-b border-border">
    <tr>
      {#each columns as col}
        <th class="text-left font-medium text-text-secondary px-3 py-2 {col.class ?? ''}">
          {col.label}
        </th>
      {/each}
    </tr>
  </thead>
  <tbody>
    {#each rows as row}
      <tr
        class="border-b border-border last:border-0 hover:bg-surface-sunken"
        class:cursor-pointer={!!onRowClick}
        onclick={() => onRowClick?.(row)}
      >
        {#each columns as col}
          <td class="px-3 py-2 {col.class ?? ''}">
            {@render col.cell(row)}
          </td>
        {/each}
      </tr>
    {/each}
  </tbody>
</table>
```

## Cross-Boundary Consistency

### Shared Vocabulary

| Concept | Phoenix Convention | Svelte Convention |
|---------|-------------------|-------------------|
| Visual style | `attr :variant, :atom` | `variant: Variant` prop |
| Scale | `attr :size, :atom` | `size: Size` prop |
| Disabled state | `attr :disabled, :boolean` | `disabled: boolean` prop |
| Content projection | Named slots (`<:header>`) | Snippets (`{#snippet header()}`) |
| Default content | `@inner_block` | `children` snippet |
| Pass-through attrs | `attr :rest, :global` | `{...restProps}` |
| Conditional render | `:if={@header != []}` | `{#if header}` |

### Variant Values Must Match

Define variant values once and use them everywhere:

```elixir
# lib/my_app_web/components/variants.ex
defmodule MyAppWeb.Variants do
  @button_variants ~w(primary secondary ghost destructive)a
  @sizes ~w(xs sm md lg)a
  @badge_variants ~w(default success warning error info)a

  def button_variants, do: @button_variants
  def sizes, do: @sizes
  def badge_variants, do: @badge_variants
end
```

```typescript
// assets/js/types/variants.ts
export type ButtonVariant = "primary" | "secondary" | "ghost" | "destructive";
export type Size = "xs" | "sm" | "md" | "lg";
export type BadgeVariant = "default" | "success" | "warning" | "error" | "info";
```

### Class Maps Must Match

The Tailwind classes for each variant must be identical in both Phoenix and Svelte. Extract them if needed:

```typescript
// assets/js/lib/component-classes.ts
export const buttonVariants = {
  primary: "bg-brand-600 text-white hover:bg-brand-700",
  secondary: "bg-surface-raised text-text-primary border border-border hover:bg-surface-sunken",
  ghost: "text-text-secondary hover:bg-surface-raised hover:text-text-primary",
  destructive: "bg-destructive text-white hover:bg-destructive-hover",
} as const;

export const buttonSizes = {
  xs: "h-7 px-2 text-xs gap-1",
  sm: "h-8 px-3 text-sm gap-1.5",
  md: "h-9 px-4 text-sm gap-2",
  lg: "h-10 px-5 text-base gap-2",
} as const;
```

Import in Svelte components. For Phoenix, keep the `defp` helpers in sync manually (or generate from a shared source).

## File Organization

```
lib/my_app_web/
├── components/
│   ├── core_components.ex    # Button, badge, input, card, table, modal, flash
│   ├── layouts.ex            # App shell, sidebar, nav
│   └── variants.ex           # Shared variant/size constants
assets/
├── svelte/
│   ├── Button.svelte         # Only if you need a Svelte button (rare)
│   ├── DataGrid.svelte       # Complex interactive table
│   ├── RichEditor.svelte     # TipTap/ProseMirror wrapper
│   ├── Chart.svelte          # Chart.js/D3 wrapper
│   └── DragDrop.svelte       # Sortable list
├── js/
│   ├── lib/
│   │   └── component-classes.ts  # Shared class maps
│   └── types/
│       └── variants.ts           # Shared type definitions
├── css/
│   ├── app.css               # Imports tokens.css + Tailwind
│   └── tokens.css            # CSS custom properties
└── tailwind.config.js        # Design tokens
```

## When Phoenix vs. When Svelte

| Use Phoenix Function Component | Use Svelte Component |
|-------------------------------|---------------------|
| Static or server-driven content | Complex client-side interactivity |
| Buttons, badges, inputs, cards | Rich text editors, data grids |
| Modals (with JS commands) | Drag-and-drop with state |
| Tables with simple data | Charts with animations |
| Flash messages, alerts | Multi-step wizards with local state |
| Navigation, breadcrumbs | Real-time collaborative features |
| Forms (LiveView handles state) | File upload with preview/crop |

**Rule of thumb:** If the component's behavior can be expressed with `phx-*` bindings and JS commands, use Phoenix. If it needs reactive local state or complex DOM manipulation, use Svelte.

## Anti-Patterns

### Duplicating components across both systems

```
❌ Button.svelte AND button/1 in core_components.ex doing the same thing
✅ Use Phoenix button/1 for server-rendered buttons (99% of cases)
   Use Svelte Button only inside other Svelte components that need it
```

### Inconsistent variant names

```
❌ Phoenix: :primary, :secondary  |  Svelte: "main", "alt"
✅ Same names everywhere: primary, secondary, ghost, destructive
```

### Hardcoding colors instead of using tokens

```
❌ class="bg-blue-600"  (what blue? whose blue?)
✅ class="bg-brand-600" (the brand blue, defined once in tailwind.config.js)
```

### Skipping :global on Phoenix components

```elixir
# ❌ No way to add phx-click, class, or data-* attributes
attr :variant, :atom, default: :primary
def button(assigns) do ...

# ✅ Accept global attributes
attr :rest, :global
def button(assigns) do
  ~H"""<button {@rest}>...</button>"""
end
```

### Over-engineering the shared layer

```
❌ Building a code generator that outputs both Phoenix and Svelte from a DSL
✅ Keep it simple: shared Tailwind config + matching class maps + same naming conventions
```

## Related Skills

- **frontend-tailwind**: Tailwind utility patterns, responsive design, dark mode
- **frontend-design**: Aesthetic direction, typography, color theory, motion principles
- **svelte-core**: LiveSvelte integration, `live.pushEvent`, SSR, reactivity
- **elixir-liveview**: LiveView lifecycle, streams, forms
- **liveview-js-interop**: JS commands for component behavior (show/hide, transitions)
