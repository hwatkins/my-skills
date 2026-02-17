---
name: component-design-system
description: >
  Build consistent, cross-framework component libraries where Phoenix function components
  and Svelte 5 components share a visual language through Tailwind design tokens. Use this
  skill whenever building UI components for a Phoenix LiveView + Svelte stack, setting up
  a component library or design system, creating variant-based component APIs (like button/1
  with :variant and :size attrs), working with slots/snippets patterns, or establishing
  shared Tailwind token systems. Also trigger when the user mentions "component library",
  "design system", "shared components", "variant API", "token system", or is building
  Phoenix function components alongside Svelte components and needs them to look and
  behave consistently.
---

# Component Design System

Architecture patterns for building a consistent component library where Phoenix function
components and Svelte 5 components live side by side, sharing visual language through
a unified Tailwind design token system.

## When You Need This Skill

You're working in a stack where Phoenix LiveView renders most of the UI via HEEx templates
and function components, but islands of interactivity are handled by Svelte 5 components
(mounted via LiveSvelte). The challenge: both sides need to produce visually identical
output from a single source of truth for colors, spacing, typography, and component
structure.

## Core Architecture

The system has three layers:

1. **Token Layer** — Tailwind CSS `@theme` variables that define the design language
2. **Component API Layer** — Variant/size/slot patterns implemented per-framework
3. **Contract Layer** — Shared conventions that keep both frameworks aligned

## Core Principles

- **One source of truth for tokens** — Tailwind config defines colors, spacing, typography, radii, shadows. Both Phoenix and Svelte components consume the same tokens.
- **Phoenix components for server-rendered UI** — buttons, badges, form inputs, layout primitives, flash messages, modals, tables.
- **Svelte components for interactive UI** — rich editors, data grids, charts, drag-and-drop, anything needing local reactive state.
- **Consistent API conventions** — same variant names, same size scale, same naming patterns across both worlds.
- **Composition over configuration** — prefer slots/snippets for flexible content over deeply nested option maps.

## Design Token System

Design tokens follow a three-tier hierarchy:

1. **Primitive tokens** — Raw color, spacing, and type values
2. **Semantic tokens** — Map primitives to intent (what components reference)
3. **Component tokens** — Scoped to a specific component when the semantic layer isn't specific enough

### Token Layers and Naming

#### Primitive Tokens (Raw Values)

```css
@theme {
  --color-blue-50: oklch(97% 0.02 250);
  --color-blue-500: oklch(60% 0.22 260);
  --color-blue-600: oklch(55% 0.22 260);
  --color-blue-900: oklch(25% 0.12 260);
  --color-gray-50: oklch(98% 0.005 260);
  --color-gray-200: oklch(90% 0.01 260);
  --color-gray-500: oklch(55% 0.02 260);
  --color-gray-900: oklch(20% 0.02 260);
  --color-red-500: oklch(60% 0.25 25);
  --color-red-600: oklch(53% 0.25 25);
}
```

#### Semantic Tokens (Intent-Based)

Map primitives to semantic meaning. These are what components reference:

```css
@theme {
  --color-primary: var(--color-blue-500);
  --color-primary-hover: var(--color-blue-600);
  --color-destructive: var(--color-red-500);
  --color-destructive-hover: var(--color-red-600);
  --color-foreground: var(--color-gray-900);
  --color-muted-foreground: var(--color-gray-500);
  --color-muted: var(--color-gray-50);
  --color-surface: oklch(99% 0.005 260);
  --color-surface-raised: oklch(100% 0 0);
  --color-border: var(--color-gray-200);
  --color-ring: var(--color-blue-500);
}
```

#### Component Tokens (Scoped to a Component)

For complex components where the semantic layer isn't specific enough.
**These go in `:root`, NOT in `@theme`** — you don't want Tailwind to generate
utility classes like `bg-btn-primary-bg`:

```css
:root {
  --btn-primary-bg: var(--color-primary);
  --btn-primary-fg: white;
  --btn-primary-hover-bg: var(--color-primary-hover);
  --btn-size-sm-px: var(--spacing-sm);
  --btn-size-sm-py: var(--spacing-xs);
  --btn-size-md-px: var(--spacing-lg);
  --btn-size-md-py: var(--spacing-sm);
  --btn-size-lg-px: var(--spacing-xl);
  --btn-size-lg-py: var(--spacing-md);
  --input-border: var(--color-border);
  --input-focus-ring: var(--color-ring);
  --input-bg: var(--color-surface);
}
```

### Tailwind v4 `@theme` Configuration

Tailwind v4 uses CSS-first configuration via the `@theme` directive. Each namespace
generates corresponding utility classes:

```css
/* assets/css/tokens.css — the single source of truth */
@import "tailwindcss";

@theme {
  /* Colors → bg-*, text-*, border-*, ring-* utilities */
  --color-primary: oklch(65% 0.25 260);
  --color-surface: oklch(99% 0.005 260);

  /* Spacing → p-*, m-*, gap-*, w-*, h-* utilities */
  --spacing-xs: 0.25rem;
  --spacing-sm: 0.5rem;
  --spacing-md: 0.75rem;
  --spacing-lg: 1rem;
  --spacing-xl: 1.5rem;
  --spacing-2xl: 2rem;
  --spacing-3xl: 3rem;

  /* Radii → rounded-* utilities */
  --radius-sm: 0.25rem;
  --radius-md: 0.375rem;
  --radius-lg: 0.5rem;
  --radius-xl: 0.75rem;
  --radius-full: 9999px;

  /* Fonts → font-* utilities */
  --font-display: "Instrument Sans", system-ui, sans-serif;
  --font-body: "Inter", system-ui, sans-serif;
  --font-mono: "JetBrains Mono", ui-monospace, monospace;

  /* Font sizes → text-* utilities */
  --text-xs: 0.75rem;
  --text-sm: 0.875rem;
  --text-base: 1rem;
  --text-lg: 1.125rem;
  --text-xl: 1.25rem;
  --text-2xl: 1.5rem;

  /* Shadows → shadow-* utilities */
  --shadow-sm: 0 1px 2px 0 rgb(0 0 0 / 0.05);
  --shadow-md: 0 4px 6px -1px rgb(0 0 0 / 0.1);
  --shadow-lg: 0 10px 15px -3px rgb(0 0 0 / 0.1);

  /* Breakpoints → responsive variants */
  --breakpoint-sm: 40rem;
  --breakpoint-md: 48rem;
  --breakpoint-lg: 64rem;
  --breakpoint-xl: 80rem;
}
```

All tokens are simultaneously available as CSS variables for use in Svelte `<style>`
blocks or arbitrary Tailwind values.

### Dark Mode and Theming

Use `@theme { initial }` to declare themeable token names, then set values per-theme
in `:root`. This keeps utility class names stable while values change:

```css
@theme {
  /* Declare names — values come from :root */
  --color-background: initial;
  --color-foreground: initial;
  --color-surface: initial;
  --color-muted: initial;
  --color-primary: initial;
}

/* Light theme (default) */
:root {
  --color-background: oklch(99% 0 0);
  --color-foreground: oklch(15% 0.02 260);
  --color-surface: oklch(100% 0 0);
  --color-muted: oklch(96% 0.005 260);
  --color-primary: oklch(55% 0.25 260);
}

/* System preference dark mode */
@media (prefers-color-scheme: dark) {
  :root {
    --color-background: oklch(15% 0.02 260);
    --color-foreground: oklch(95% 0.005 260);
    --color-surface: oklch(20% 0.02 260);
    --color-muted: oklch(25% 0.02 260);
    --color-primary: oklch(70% 0.22 260);
  }
}

/* Class-based dark mode (for manual toggle) */
.dark {
  --color-background: oklch(15% 0.02 260);
  --color-foreground: oklch(95% 0.005 260);
  --color-surface: oklch(20% 0.02 260);
  --color-muted: oklch(25% 0.02 260);
  --color-primary: oklch(70% 0.22 260);
}
```

This means `bg-background`, `text-foreground`, etc. automatically adapt to the current
theme in both Phoenix templates and Svelte components.

### Consuming Tokens

#### In Phoenix HEEx Templates

Use Tailwind utility classes directly:

```heex
<div class="bg-surface rounded-lg p-lg shadow-md">
  <h2 class="text-xl font-display text-foreground">Title</h2>
  <p class="text-sm text-muted-foreground">Description</p>
</div>
```

For computed/conditional styles, use arbitrary value syntax:

```heex
<div class="px-[--spacing-lg] py-[--spacing-md]">
  <!-- When you need a token value that doesn't have a perfect utility -->
</div>
```

#### In Svelte Components

Use the same Tailwind classes in markup. For Svelte `<style>` blocks, reference
the CSS variables directly:

```svelte
<div class="bg-surface rounded-lg p-lg shadow-md">
  <h2 class="text-xl font-display text-foreground">Title</h2>
</div>

<style>
  .custom-layout {
    padding: var(--spacing-lg);
    gap: var(--spacing-md);
    border-radius: var(--radius-lg);
  }
</style>
```

#### In JavaScript/TypeScript (Runtime Access)

For animations, canvas rendering, or dynamic styles:

```typescript
const primary = getComputedStyle(document.documentElement)
  .getPropertyValue('--color-primary');
```

## Phoenix Function Component Patterns

### The Variant + Size Pattern

The variant pattern uses `attr/3` declarations with constrained values and private helper
functions for class resolution. This gives compile-time validation and clean separation
of concerns.

```elixir
defmodule MyAppWeb.UI.Button do
  use Phoenix.Component

  @doc """
  Renders a button with variant and size support.

  ## Examples

      <.button>Click me</.button>
      <.button variant="outline" size="lg">Big outline</.button>
      <.button variant="destructive" phx-click="delete">Remove</.button>
  """
  attr :variant, :string,
    values: ~w(primary secondary outline ghost destructive),
    default: "primary",
    doc: "Visual style variant"
  attr :size, :string,
    values: ~w(sm md lg icon),
    default: "md",
    doc: "Size preset"
  attr :disabled, :boolean, default: false
  attr :class, :string, default: nil
  attr :rest, :global,
    include: ~w(type name form phx-click phx-disable-with navigate patch href)

  slot :inner_block, required: true
  slot :icon_left, doc: "Icon rendered before the label"
  slot :icon_right, doc: "Icon rendered after the label"

  def button(assigns) do
    ~H"""
    <button
      class={[
        base_classes(),
        variant_classes(@variant),
        size_classes(@size),
        @class
      ]}
      disabled={@disabled}
      {@rest}
    >
      <span :if={@icon_left != []} class="shrink-0">{render_slot(@icon_left)}</span>
      {render_slot(@inner_block)}
      <span :if={@icon_right != []} class="shrink-0">{render_slot(@icon_right)}</span>
    </button>
    """
  end

  defp base_classes do
    "inline-flex items-center justify-center gap-2 font-medium rounded-md " <>
    "transition-colors focus-visible:outline-2 focus-visible:outline-offset-2 " <>
    "focus-visible:outline-primary disabled:opacity-50 disabled:pointer-events-none"
  end

  defp variant_classes("primary"),     do: "bg-[--btn-primary-bg] text-[--btn-primary-fg] hover:bg-primary-hover active:bg-primary-active"
  defp variant_classes("secondary"),   do: "bg-muted text-foreground hover:bg-muted/80"
  defp variant_classes("outline"),     do: "border border-border text-foreground hover:bg-muted"
  defp variant_classes("ghost"),       do: "text-foreground hover:bg-muted"
  defp variant_classes("destructive"), do: "bg-destructive text-white hover:bg-destructive/90"

  defp size_classes("sm"),   do: "text-sm h-8 px-[--btn-size-sm-px] py-[--btn-size-sm-py]"
  defp size_classes("md"),   do: "text-sm h-10 px-[--btn-size-md-px] py-[--btn-size-md-py]"
  defp size_classes("lg"),   do: "text-base h-12 px-[--btn-size-lg-px] py-[--btn-size-lg-py]"
  defp size_classes("icon"), do: "size-10"
end
```

### Why Pattern-Matched Functions for Variants

Using `defp variant_classes("primary")` over a map lookup:
- Compiler catches typos in variant values
- Each clause is self-contained and easy to read
- Works naturally with Phoenix's class list merging (list of strings, nils filtered)
- Easy to extend without touching other variants

Size classes use Tailwind's arbitrary property syntax (`px-[--btn-size-sm-px]`)
to reference the component tokens from `:root`. This means changing button padding
requires editing only `tokens.css`.

### Usage

```heex
<.button>Save</.button>
<.button variant="secondary" size="sm">Cancel</.button>
<.button variant="destructive" phx-click="delete" data-confirm="Are you sure?">
  Delete
</.button>
<.button variant="outline" size="lg">
  <:icon_left><.icon name="hero-plus" /></:icon_left>
  Add Item
</.button>
<.button variant="ghost" class="w-full">Full Width Ghost</.button>
```

### Named Slots for Complex Layouts

```elixir
slot :header
slot :inner_block, required: true
slot :footer

attr :padding, :string, values: ~w(none sm md lg), default: "md"
attr :rest, :global

def card(assigns) do
  ~H"""
  <div class="rounded-lg border border-border bg-surface-raised shadow-sm">
    <div :if={@header != []} class="border-b border-border px-lg py-md">
      {render_slot(@header)}
    </div>
    <div class={padding_classes(@padding)}>
      {render_slot(@inner_block)}
    </div>
    <div :if={@footer != []} class="border-t border-border px-lg py-md bg-muted/50">
      {render_slot(@footer)}
    </div>
  </div>
  """
end

defp padding_classes("none"), do: ""
defp padding_classes("sm"),   do: "p-sm"
defp padding_classes("md"),   do: "p-lg"
defp padding_classes("lg"),   do: "p-xl"
```

```heex
<.card>
  <:header>
    <h3 class="font-semibold">Settings</h3>
  </:header>
  <p>Card content here.</p>
  <:footer>
    <.button size="sm">Save</.button>
  </:footer>
</.card>
```

### Render Slot with Fallback

```elixir
<div class="modal-title">
  {render_slot(@title) || "Untitled"}
</div>
```

### The Table Component with Slot Attributes

Named slots can declare their own attributes. This is the most powerful slot pattern —
it enables the caller to define structure (columns) while the component handles rendering
(iteration, layout).

```elixir
attr :rows, :list, required: true
attr :row_id, :fun, default: &Phoenix.Param.to_param/1
attr :row_click, :any, default: nil

slot :col, required: true do
  attr :label, :string, required: true
  attr :class, :string
end

def data_table(assigns) do
  ~H"""
  <div class="overflow-x-auto">
    <table class="w-full text-sm">
      <thead>
        <tr class="border-b-2 border-border">
          <th
            :for={col <- @col}
            class={["px-lg py-sm text-left font-semibold text-muted-foreground", col[:class]]}
          >
            {col.label}
          </th>
        </tr>
      </thead>
      <tbody>
        <tr
          :for={row <- @rows}
          id={"row-#{@row_id.(row)}"}
          class={["border-b border-border hover:bg-muted/50 transition-colors", @row_click && "cursor-pointer"]}
          phx-click={@row_click && @row_click.(row)}
        >
          <td :for={col <- @col} class={["px-lg py-md", col[:class]]}>
            {render_slot(col, row)}
          </td>
        </tr>
      </tbody>
    </table>
  </div>
  """
end
```

Usage with `:let` to pass row data back:

```heex
<.data_table rows={@users}>
  <:col :let={user} label="Name">
    <span class="font-medium">{user.name}</span>
  </:col>
  <:col :let={user} label="Email" class="text-muted-foreground">
    {user.email}
  </:col>
  <:col :let={user} label="Actions" class="text-right">
    <.button size="sm" variant="ghost" phx-click="edit" phx-value-id={user.id}>
      Edit
    </.button>
  </:col>
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

### Compound Components

For complex UI patterns (dropdown menus, accordions, tabs), compose multiple function
components that share a namespace:

```elixir
defmodule MyAppWeb.UI.Tabs do
  use Phoenix.Component

  attr :default, :string, required: true, doc: "ID of the initially active tab"
  slot :inner_block, required: true

  def tabs(assigns) do
    ~H"""
    <div id="tabs" phx-hook="Tabs" data-default={@default}>
      {render_slot(@inner_block)}
    </div>
    """
  end

  attr :id, :string, required: true
  slot :inner_block, required: true

  def tab_trigger(assigns) do
    ~H"""
    <button
      role="tab"
      data-tab-trigger={@id}
      class="px-lg py-sm text-sm font-medium text-muted-foreground
             data-[active]:text-foreground data-[active]:border-b-2
             data-[active]:border-primary transition-colors"
    >
      {render_slot(@inner_block)}
    </button>
    """
  end

  attr :id, :string, required: true
  slot :inner_block, required: true

  def tab_content(assigns) do
    ~H"""
    <div role="tabpanel" data-tab-content={@id} class="hidden data-[active]:block py-lg">
      {render_slot(@inner_block)}
    </div>
    """
  end
end
```

```heex
<.tabs default="overview">
  <div class="flex gap-xs border-b border-border">
    <.tab_trigger id="overview">Overview</.tab_trigger>
    <.tab_trigger id="analytics">Analytics</.tab_trigger>
  </div>
  <.tab_content id="overview">Overview content...</.tab_content>
  <.tab_content id="analytics">Analytics content...</.tab_content>
</.tabs>
```

### Global Attribute Forwarding

Use `attr :rest, :global` to forward HTML attributes and Phoenix-specific bindings.
Always specify `include:` to document which global attrs the component accepts:

```elixir
# For interactive elements
attr :rest, :global, include: ~w(phx-click phx-target phx-value-id)

# For navigation elements
attr :rest, :global, include: ~w(navigate patch href method)

# For form elements
attr :rest, :global, include: ~w(
  name form autocomplete placeholder required
  min max minlength maxlength pattern
  phx-change phx-blur phx-focus phx-debounce
)
```

### LiveComponent vs Function Component

Use **function components** for:
- Stateless rendering with variant/slot APIs
- All design system components (buttons, inputs, cards, modals)
- Any component that doesn't need its own event handling

Use **LiveComponent** for:
- Components that need their own `handle_event` callbacks
- Components with internal state that changes independently of the parent
- Reusable widgets that encapsulate both state and presentation

The design system should be composed entirely of function components. LiveComponents
are application-level, not design-system-level.

## Svelte Component Patterns

### Props with `$props()` and TypeScript

Svelte 5 uses the `$props()` rune for all prop declarations. For design system
components, always type your props explicitly:

```svelte
<script lang="ts">
  import type { Snippet } from "svelte";

  interface Props {
    variant?: "primary" | "secondary" | "outline" | "ghost" | "destructive";
    size?: "sm" | "md" | "lg";
    disabled?: boolean;
    class?: string;
    children: Snippet;
    onclick?: (e: MouseEvent) => void;
  }

  let {
    variant = "primary",
    size = "md",
    disabled = false,
    class: className = "",
    children,
    onclick,
  }: Props = $props();
</script>
```

Key patterns:
- Rename `class` to `className` via destructuring (`class` is reserved in JS)
- Use union types for variant values — mirrors Phoenix's `values: ~w(...)` constraint
- Default values in destructuring — mirrors Phoenix's `default:` option
- `Snippet` type from `"svelte"` for content slots
- Event handlers are plain callback props (no more `createEventDispatcher`)

### Variant Pattern with Type Safety

Mirror Phoenix's variant helper functions with TypeScript Record objects.
The class strings **must match** the Phoenix `defp variant_classes/1` exactly:

```svelte
<!-- assets/svelte/Button.svelte -->
<script lang="ts">
  import type { Snippet } from "svelte";

  type Variant = "primary" | "secondary" | "outline" | "ghost" | "destructive";
  type Size = "sm" | "md" | "lg" | "icon";

  interface Props {
    variant?: Variant;
    size?: Size;
    disabled?: boolean;
    class?: string;
    children: Snippet;
    iconLeft?: Snippet;
    iconRight?: Snippet;
    onclick?: (e: MouseEvent) => void;
  }

  let {
    variant = "primary",
    size = "md",
    disabled = false,
    class: className = "",
    children,
    iconLeft,
    iconRight,
    onclick,
  }: Props = $props();

  const VARIANT: Record<Variant, string> = {
    primary:     "bg-[--btn-primary-bg] text-[--btn-primary-fg] hover:bg-primary-hover active:bg-primary-active",
    secondary:   "bg-muted text-foreground hover:bg-muted/80",
    outline:     "border border-border text-foreground hover:bg-muted",
    ghost:       "text-foreground hover:bg-muted",
    destructive: "bg-destructive text-white hover:bg-destructive/90",
  };

  const SIZE: Record<Size, string> = {
    sm:   "text-sm h-8 px-[--btn-size-sm-px] py-[--btn-size-sm-py]",
    md:   "text-sm h-10 px-[--btn-size-md-px] py-[--btn-size-md-py]",
    lg:   "text-base h-12 px-[--btn-size-lg-px] py-[--btn-size-lg-py]",
    icon: "size-10",
  };

  const BASE = `inline-flex items-center justify-center gap-2 font-medium rounded-md
    transition-colors focus-visible:outline-2 focus-visible:outline-offset-2
    focus-visible:outline-primary disabled:opacity-50 disabled:pointer-events-none`;
</script>

<button
  class="{BASE} {VARIANT[variant]} {SIZE[size]} {className}"
  {disabled}
  {onclick}
>
  {#if iconLeft}
    <span class="shrink-0">{@render iconLeft()}</span>
  {/if}
  {@render children()}
  {#if iconRight}
    <span class="shrink-0">{@render iconRight()}</span>
  {/if}
</button>
```

### Snippet Composition (Replacing Slots)

Svelte 5 replaces slots with snippets. Snippets are more explicit and flexible.

#### Named Snippets (Card Example)

```svelte
<!-- assets/svelte/Card.svelte -->
<script lang="ts">
  import type { Snippet } from "svelte";

  interface Props {
    header?: Snippet;
    children: Snippet;
    footer?: Snippet;
  }

  let { header, children, footer }: Props = $props();
</script>

<div class="rounded-lg border border-border bg-surface-raised shadow-sm">
  {#if header}
    <div class="border-b border-border px-lg py-md">
      {@render header()}
    </div>
  {/if}
  <div class="px-lg py-lg">
    {@render children()}
  </div>
  {#if footer}
    <div class="border-t border-border px-lg py-md bg-muted/50">
      {@render footer()}
    </div>
  {/if}
</div>
```

```svelte
<!-- Usage -->
<Card>
  {#snippet header()}
    <h3 class="font-semibold">Settings</h3>
  {/snippet}

  <p>Card content here.</p>

  {#snippet footer()}
    <Button size="sm">Save</Button>
  {/snippet}
</Card>
```

#### Snippets with Parameters (Replaces `:let`)

This is the Svelte equivalent of Phoenix's `render_slot(@col, row)` with `:let={value}`:

```svelte
<script lang="ts" generics="T">
  import type { Snippet } from "svelte";

  interface Props<T> {
    items: T[];
    children: Snippet<[T, number]>;
    empty?: Snippet;
  }

  let { items, children, empty }: Props<T> = $props();
</script>

{#if items.length === 0 && empty}
  {@render empty()}
{:else}
  {#each items as item, index}
    {@render children(item, index)}
  {/each}
{/if}
```

```svelte
<!-- Usage -->
<List items={users}>
  {#snippet children(user, i)}
    <div class="flex items-center gap-md">
      <span class="text-muted-foreground">{i + 1}.</span>
      <span>{user.name}</span>
    </div>
  {/snippet}
  {#snippet empty()}
    <p class="text-muted-foreground">No users found.</p>
  {/snippet}
</List>
```

### Event Handling

Svelte 5 uses plain callback props instead of `on:` directives:

```svelte
<!-- Component definition -->
<script lang="ts">
  let { onclick, onkeydown, ...rest }: {
    onclick?: (e: MouseEvent) => void;
    onkeydown?: (e: KeyboardEvent) => void;
  } = $props();
</script>
<button {onclick} {onkeydown}>...</button>

<!-- Usage -->
<Button onclick={() => console.log("clicked")}>Click</Button>
```

For forwarding all event handlers, use rest props:

```svelte
<script lang="ts">
  let { children, class: className, ...rest } = $props();
</script>
<button class={className} {...rest}>{@render children()}</button>
```

### Generic Data Table

```svelte
<!-- assets/svelte/DataTable.svelte -->
<script lang="ts" generics="T">
  import type { Snippet } from "svelte";

  interface Column<T> {
    label: string;
    class?: string;
    cell: Snippet<[T]>;
  }

  interface Props<T> {
    rows: T[];
    columns: Column<T>[];
    rowId: (row: T) => string;
  }

  let { rows, columns, rowId }: Props<T> = $props();
</script>

<div class="overflow-x-auto">
  <table class="w-full text-sm">
    <thead>
      <tr class="border-b-2 border-border">
        {#each columns as col}
          <th class="px-lg py-sm text-left font-semibold text-muted-foreground {col.class ?? ''}">{col.label}</th>
        {/each}
      </tr>
    </thead>
    <tbody>
      {#each rows as row (rowId(row))}
        <tr class="border-b border-border hover:bg-muted/50 transition-colors">
          {#each columns as col}
            <td class="px-lg py-md {col.class ?? ''}">{@render col.cell(row)}</td>
          {/each}
        </tr>
      {/each}
    </tbody>
  </table>
</div>
```

## Cross-Framework Component Contract

### Alignment Checklist

When building a component that exists in both Phoenix and Svelte, verify:

- [ ] **Same variant names**: Both use identical string values (e.g., "primary", "outline")
- [ ] **Same size names**: Both use identical string values (e.g., "sm", "md", "lg")
- [ ] **Same HTML structure**: Same element hierarchy, same semantic elements
- [ ] **Same class strings**: Variant→class and size→class mappings are character-identical
- [ ] **Same ARIA attributes**: role, aria-label, aria-expanded, etc. are consistent
- [ ] **Same slot→snippet mapping**: Each Phoenix slot has a corresponding Svelte snippet prop
- [ ] **Same defaults**: Default variant, size, and boolean values match
- [ ] **Same token usage**: Neither framework hardcodes values that should come from tokens

### Mapping Phoenix to Svelte Concepts

| Phoenix | Svelte 5 | Notes |
|---------|----------|-------|
| `attr :variant, :string, values: ~w(a b)` | `variant?: "a" \| "b"` | Same values, different syntax |
| `attr :disabled, :boolean, default: false` | `disabled?: boolean` (default in destructuring) | |
| `attr :class, :string, default: nil` | `class?: string` (rename to `className`) | |
| `attr :rest, :global` | `...rest` spread | |
| `slot :inner_block` | `children: Snippet` | |
| `slot :header` | `header?: Snippet` | |
| `slot :col do attr :label, :string end` | `Column<T>` interface with `label` field | |
| `render_slot(@col, row)` | `{@render col.cell(row)}` | Data passed back to caller |
| `:if={@header != []}` | `{#if header}` | Optional slot check |
| `phx-click={JS.push("event")}` | `onclick` callback prop | Different event systems |
| `@class` (class list merge) | Template literal class string | |

### Variant Values Must Match

Define variant values once and use them everywhere:

```elixir
# lib/my_app_web/components/variants.ex
defmodule MyAppWeb.Variants do
  @button_variants ~w(primary secondary outline ghost destructive)
  @sizes ~w(sm md lg icon)
  @badge_variants ~w(default success warning error info)

  def button_variants, do: @button_variants
  def sizes, do: @sizes
  def badge_variants, do: @badge_variants
end
```

### Class Maps Must Match (Shared Variant Maps)

The Tailwind classes for each variant must be identical in both Phoenix and Svelte.
Extract them to a shared TypeScript module that serves as documentation and the
single reference when updating Phoenix `defp` functions:

```typescript
// assets/svelte/lib/variants.ts
export const buttonVariants = {
  primary:     "bg-[--btn-primary-bg] text-[--btn-primary-fg] hover:bg-primary-hover active:bg-primary-active",
  secondary:   "bg-muted text-foreground hover:bg-muted/80",
  outline:     "border border-border text-foreground hover:bg-muted",
  ghost:       "text-foreground hover:bg-muted",
  destructive: "bg-destructive text-white hover:bg-destructive/90",
} as const;

export const buttonSizes = {
  sm:   "text-sm h-8 px-[--btn-size-sm-px] py-[--btn-size-sm-py]",
  md:   "text-sm h-10 px-[--btn-size-md-px] py-[--btn-size-md-py]",
  lg:   "text-base h-12 px-[--btn-size-lg-px] py-[--btn-size-lg-py]",
  icon: "size-10",
} as const;

export type ButtonVariant = keyof typeof buttonVariants;
export type ButtonSize = keyof typeof buttonSizes;
export type BadgeVariant = "default" | "success" | "warning" | "error" | "info";
```

### Class String Synchronization Strategy

The class strings are the real contract. To keep them in sync:

**Option A: Copy-and-verify (Recommended for small systems)**

Maintain the class strings in both frameworks. When updating one, search for the same
strings in the other and update there too. Add a comment at the top of each variant map:

```elixir
# Synced with: assets/svelte/components/Button.svelte
defp variant_classes("primary"), do: "bg-[--btn-primary-bg] text-[--btn-primary-fg] hover:bg-primary-hover"
```

```typescript
// Synced with: lib/my_app_web/components/ui/button.ex
const VARIANT = {
  primary: "bg-[--btn-primary-bg] text-[--btn-primary-fg] hover:bg-primary-hover",
};
```

**Option B: Shared JSON (For larger systems)**

Define variants in a JSON file consumed by both:

```json
// assets/shared/button-variants.json
{
  "variants": {
    "primary": "bg-[--btn-primary-bg] text-[--btn-primary-fg] hover:bg-primary-hover",
    "secondary": "bg-muted text-foreground hover:bg-muted/80"
  },
  "sizes": {
    "sm": "text-sm h-8 px-[--btn-size-sm-px] py-[--btn-size-sm-py]",
    "md": "text-sm h-10 px-[--btn-size-md-px] py-[--btn-size-md-py]"
  }
}
```

Phoenix reads this at compile time via a module attribute. Svelte imports it directly.
This guarantees identical strings but adds build complexity.

**Option C: Generated from tokens (For design-system-at-scale)**

Write a code generator that produces both `button.ex` helper functions and
`button-variants.ts` from a single YAML/TOML component spec. Only worthwhile for
very large component libraries.

### Common Components to Build First

Priority order for a Phoenix + Svelte design system:

1. **Button** — The canonical variant/size/slot component. Template for all others.
2. **Input/TextField** — Form fields with label, error, helper text. Phoenix needs for forms; Svelte for interactive widgets.
3. **Card** — Header/body/footer slot pattern. Tests named slot alignment.
4. **Badge** — Simple variant-only component. Quick to build, validates token usage.
5. **Alert** — Variant + icon + dismiss slot. Tests interactive patterns.
6. **Modal/Dialog** — Complex overlay with backdrop, header, body, footer, close. Tests JS interop.
7. **DataTable** — Advanced slot-with-attributes pattern. Tests data rendering.
8. **Dropdown/Select** — Tests compound component patterns and state management.

### Handling Interactive Behavior Differences

Phoenix and Svelte handle interactivity differently. The design system defines the
visual contract (classes, structure) while each framework handles behavior its own way:

| Behavior | Phoenix | Svelte |
|----------|---------|--------|
| Show/hide | `JS.show()` / `JS.hide()` with transitions | `{#if}` with `transition:` directive |
| Toggle | `JS.toggle_class()` | `$state` boolean + conditional classes |
| Dropdown open | `JS.toggle()` + `phx-click-away` | `$state` + `onclick` outside handler |
| Form validation | `phx-change` → server → re-render | `$state` + local validation |
| Navigation | `<.link navigate={...}>` | `<a>` or framework router |

The visual output (what classes are applied, what HTML structure exists) should be
identical. The mechanism for arriving at that state differs per framework.

## File Organization

```
lib/my_app_web/
├── components/
│   ├── core_components.ex      # Phoenix function components (button, badge, input, card, table)
│   ├── layouts.ex              # Layout components (app shell, sidebar, nav)
│   └── ui/
│       ├── button.ex           # Complex components get own modules
│       ├── data_table.ex
│       └── form_fields.ex
assets/
├── css/
│   ├── tokens.css              # @theme design tokens (THE source of truth)
│   └── app.css                 # Imports tokens.css + tailwindcss
├── svelte/
│   ├── components/
│   │   ├── Button.svelte       # Mirrors button/1 (only for use inside other Svelte components)
│   │   ├── DataGrid.svelte     # Complex interactive table
│   │   ├── RichEditor.svelte   # TipTap/ProseMirror wrapper
│   │   ├── Chart.svelte        # Chart.js/D3 wrapper
│   │   └── DragDrop.svelte     # Sortable list
│   └── lib/
│       └── variants.ts         # Shared variant/size class maps (optional)
├── js/
│   ├── lib/
│   │   └── component-classes.ts  # Shared class maps
│   └── types/
│       └── variants.ts           # Shared type definitions
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
✅ class="bg-primary"   (the primary color, defined once in tokens.css @theme)
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
✅ Keep it simple: shared tokens.css + matching class maps + same naming conventions
```

## Building New Components: The Process

When creating a new component for the design system:

1. **Define the API contract first.** Write out variants, sizes, and slots on paper.
   Both Phoenix and Svelte versions must accept the same props/attrs (with framework
   idioms for naming).

2. **Extract Tailwind class maps.** Build variant and size class maps as shared
   constants. In Phoenix these are `defp` functions. In Svelte these are `Record`
   objects. The string values must be identical.

3. **Establish the DOM structure.** Both versions should render the same HTML element
   hierarchy with the same class patterns. Semantic HTML first, then ARIA attributes.

4. **Implement slots/snippets.** Map Phoenix named slots to Svelte snippet props.
   Optional slots in Phoenix (`@foo != []`) become optional snippet props in Svelte
   (`if foo`).

5. **Test visual parity.** Render both versions side by side. They should be pixel-
   identical because they share the same tokens and class strings.

## Key Principles

**Tokens are the contract.** Never hardcode hex colors or pixel values in components.
Always reference Tailwind token utilities (`bg-primary`, `px-lg`) or CSS variables
(`var(--color-primary)`).

**Class strings are the API.** The variant→class mapping is the real interface between
the two frameworks. Keep these maps identical. If you update one, update both.

**Prefer composition over configuration.** Use slots (Phoenix) and snippets (Svelte)
for flexible content rather than adding more and more props. A `card/1` with `:header`,
`:inner_block`, and `:footer` slots is more composable than one with 15 content attrs.

**Use `attr` declarations and TypeScript types.** Phoenix's `attr/3` and `slot/3`
macros give compile-time warnings. Svelte 5's typed `$props()` gives editor
IntelliSense. Both enforce the component contract.

**Don't abstract too early.** Start with concrete components. Extract shared patterns
(like variant class maps to a shared file) only when you have three or more components
using the same pattern.

## Related Skills

- **frontend-tailwind**: Tailwind utility patterns, responsive design, dark mode
- **frontend-design**: Aesthetic direction, typography, color theory, motion principles
- **svelte-core**: LiveSvelte integration, `live.pushEvent`, SSR, reactivity
- **elixir-liveview**: LiveView lifecycle, streams, forms
- **liveview-js-interop**: JS commands for component behavior (show/hide, transitions)
