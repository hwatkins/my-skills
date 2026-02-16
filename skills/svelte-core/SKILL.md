---
name: svelte-core
description: Svelte 5 components inside Phoenix LiveView via LiveSvelte. Covers reactivity, props from LiveView, live.pushEvent, SSR, slots, and end-to-end reactivity patterns. Use when building Svelte components in a Phoenix project.
---

# LiveSvelte — Svelte inside Phoenix LiveView

Expert guidance for building Svelte 5 components inside Phoenix LiveView using [LiveSvelte](https://github.com/woutdp/live_svelte) for end-to-end reactivity.

## Core Principles

- Svelte components live in `assets/svelte/` — they receive props from LiveView over the websocket
- **Server owns the state** — push events to LiveView, let the server update assigns, props flow back reactively
- Don't duplicate state: if LiveView has it, pass it as a prop; use local Svelte state only for UI-only concerns
- Use Svelte 5 runes (`$state`, `$derived`, `$effect`) — not legacy `$:` reactive declarations
- Use TypeScript for type safety
- SSR is enabled by default — be aware of what runs on server vs client

## LiveSvelte Usage

### Basic Component

```elixir
# LiveView
defmodule MyAppWeb.CounterLive do
  use MyAppWeb, :live_view

  def render(assigns) do
    ~H"""
    <.svelte name="Counter" props={%{number: @number}} socket={@socket} />
    """
  end

  def mount(_params, _session, socket) do
    {:ok, assign(socket, :number, 0)}
  end

  def handle_event("set_number", %{"number" => number}, socket) do
    {:noreply, assign(socket, :number, number)}
  end
end
```

```svelte
<!-- assets/svelte/Counter.svelte -->
<script lang="ts">
  // Props come from LiveView assigns — reactive over the websocket
  let { number, live } = $props();

  function increase() {
    // Push event to LiveView — server updates the assign — prop flows back
    live.pushEvent("set_number", { number: number + 1 }, () => {});
  }

  function decrease() {
    live.pushEvent("set_number", { number: number - 1 }, () => {});
  }
</script>

<p>The number is {number}</p>
<button onclick={increase}>+</button>
<button onclick={decrease}>-</button>
```

### The Components Macro

Use `LiveSvelte.Components` for a more JSX-like experience in HEEx:

```elixir
defmodule MyAppWeb.DashboardLive do
  use MyAppWeb, :live_view
  use LiveSvelte.Components

  def render(assigns) do
    ~H"""
    <.Counter number={@number} socket={@socket} />
    <.Chart data={@chart_data} socket={@socket} />
    """
  end
end
```

### The ~V Sigil (Svelte as LiveView DSL)

Use `~V` instead of `~H` to write Svelte directly in your LiveView:

```elixir
defmodule MyAppWeb.InlineSvelteLive do
  use MyAppWeb, :live_view

  def render(assigns) do
    ~V"""
    <script>
      export let count = 0
      let local_state = "only in svelte"
    </script>

    <p>Count from server: {count}</p>
    <p>Local: {local_state}</p>
    <button phx-click="increment">Server increment</button>
    <button on:click={() => local_state = "changed"}>Local change</button>
    """
  end

  def mount(_params, _session, socket) do
    {:ok, assign(socket, :count, 0)}
  end

  def handle_event("increment", _value, socket) do
    {:noreply, assign(socket, :count, socket.assigns.count + 1)}
  end
end
```

## The `live` Object

The `live` prop is automatically injected into every LiveSvelte component. Available methods:

| Method | Description |
|--------|-------------|
| `live.pushEvent(event, payload, callback)` | Push event to the parent LiveView |
| `live.pushEventTo(selector, event, payload, callback)` | Push event to a specific LiveView/component |
| `live.handleEvent(event, callback)` | Listen for server-pushed events |
| `live.removeHandleEvent(ref)` | Remove an event listener |
| `live.upload(name, files)` | Upload files |
| `live.uploadTo(selector, name, files)` | Upload files to a specific component |

**Important**: These methods only work on the client. Wrap in `onMount` or call from event handlers — never at the top level (SSR will fail).

```svelte
<script lang="ts">
  import { onMount } from "svelte";

  let { live } = $props();
  let notifications = $state<string[]>([]);

  onMount(() => {
    // Listen for server-pushed events
    const ref = live.handleEvent("new_notification", (payload) => {
      notifications = [...notifications, payload.message];
    });

    return () => live.removeHandleEvent(ref);
  });

  function dismiss(index: number) {
    live.pushEvent("dismiss_notification", { index }, () => {});
  }
</script>
```

## Svelte 5 Runes (Quick Reference)

### State & Derived

```svelte
<script lang="ts">
  // Props from LiveView (reactive over websocket)
  let { items, live } = $props();

  // Local UI state (not sent to server)
  let filter = $state("all");
  let searchQuery = $state("");

  // Derived from props + local state
  let filteredItems = $derived(
    items.filter((i) =>
      (filter === "all" || i.status === filter) &&
      i.name.toLowerCase().includes(searchQuery.toLowerCase())
    )
  );

  let count = $derived(filteredItems.length);
</script>
```

### Effects

```svelte
<script lang="ts">
  let { query } = $props();

  // ✅ Good: Effect for side effects (e.g., focus, scroll, external libs)
  $effect(() => {
    if (query) {
      document.title = `Search: ${query}`;
    }
    return () => { document.title = "My App"; };
  });

  // ❌ Bad: Using $effect to derive state
  // let count = $state(0);
  // $effect(() => { count = items.length }); // Use $derived instead!
</script>
```

## Component Props (Svelte-only patterns)

```svelte
<script lang="ts">
  import type { Snippet } from "svelte";

  interface Props {
    title: string;
    variant?: "primary" | "secondary";
    children?: Snippet;
    onclick?: () => void;
  }

  let { title, variant = "primary", children, onclick }: Props = $props();
</script>

<div class="card {variant}">
  <h2>{title}</h2>
  {#if children}
    {@render children()}
  {/if}
  {#if onclick}
    <button onclick={onclick}>Action</button>
  {/if}
</div>
```

## Slots (LiveView → Svelte)

Slot HEEx content from LiveView into Svelte components:

```elixir
# LiveView template
<.svelte name="Card">
  <p>This HEEx content is slotted into Svelte</p>
</.svelte>

# Named slots
<.svelte name="Card">
  Main content here
  <:subtitle>
    <p>Subtitle from LiveView</p>
  </:subtitle>
</.svelte>
```

```svelte
<!-- assets/svelte/Card.svelte -->
<script lang="ts">
  let { children, subtitle } = $props();
</script>

<div class="card">
  {@render children?.()}
  {#if subtitle}
    <h3>{@render subtitle()}</h3>
  {/if}
</div>
```

**Note**: Slotted content is wrapped in a `<div>` by LiveSvelte (limitation of `createRawSnippet`).

## SSR (Server-Side Rendering)

SSR is enabled by default. On first page load, Svelte renders HTML on the server, then hydrates on the client.

### Disable SSR

```elixir
# Per component
<.svelte name="HeavyChart" ssr={false} props={%{data: @data}} socket={@socket} />

# Globally in config.exs
config :live_svelte, ssr: false
```

### SSR Caveats

- `live.pushEvent` and other `live` methods don't work during SSR — wrap in `onMount`
- `window`, `document`, `localStorage` don't exist during SSR — guard with `onMount` or `browser` check
- Set `NODE_ENV=production` in production deployments to avoid memory leaks during SSR

```svelte
<script lang="ts">
  import { onMount } from "svelte";

  let { live } = $props();
  let mounted = $state(false);

  onMount(() => {
    mounted = true;
    // Safe to use browser APIs and live methods here
  });
</script>

{#if mounted}
  <InteractiveWidget />
{:else}
  <LoadingPlaceholder />
{/if}
```

## live_json (Optimized Large Data)

For large JSON payloads, use `live_json` to send diffs instead of full objects:

```elixir
def render(assigns) do
  ~H"""
  <.svelte name="DataTable" live_json_props={%{rows: @ljrows}} socket={@socket} />
  """
end

def mount(_, _, socket) do
  {:ok, LiveJson.initialize("rows", large_dataset())}
end

def handle_info({:data_updated, new_data}, socket) do
  {:noreply, LiveJson.push_patch(socket, "rows", new_data)}
end
```

Only use `live_json` for large, frequently-changing data. For small payloads, regular props are cheaper.

## Structs and Ecto

LiveSvelte serializes props to JSON. With OTP 27+ native JSON (default), structs are auto-converted to maps.

If using Jason, add `@derive`:

```elixir
defmodule MyApp.Task do
  use Ecto.Schema

  # Only include fields safe for the client
  @derive {Jason.Encoder, except: [:__meta__, :password_hash]}
  schema "tasks" do
    field :title, :string
    field :status, :string
    timestamps()
  end
end
```

## Control Flow

```svelte
{#if loading}
  <Spinner />
{:else if error}
  <p class="error">{error}</p>
{:else if items.length === 0}
  <p>No items found.</p>
{:else}
  {#each items as item (item.id)}
    <ItemCard {item} ondelete={() => live.pushEvent("delete", { id: item.id }, () => {})} />
  {/each}
{/if}
```

## LiveView Navigation from Svelte

```svelte
<!-- push_navigate (different LiveView) -->
<a href="/other-page" data-phx-link="redirect" data-phx-link-state="push">Go</a>

<!-- push_patch (same LiveView, different params) -->
<a href="/current?tab=settings" data-phx-link="patch" data-phx-link-state="push">Settings</a>
```

Useful for preserving Svelte store state across navigation.

## CSS & Styling

```svelte
<style>
  /* Scoped by default */
  .card { padding: 1rem; border-radius: 0.5rem; }

  /* Use :global() to escape scoping (e.g., style slotted HEEx content) */
  :global(.from-liveview) { color: blue; }
</style>

<!-- Dynamic classes -->
<div class="card" class:active={isActive}>
```

## Secret State Caveat

Unlike LiveView (which only sends HTML over the wire), LiveSvelte sends **JSON data** to the client. Svelte code with conditionals will contain the logic for all branches — even hidden ones.

```svelte
<!-- ❌ The admin panel code is visible in the JS bundle even when !isAdmin -->
{#if isAdmin}
  <AdminPanel {secretData} />
{/if}

<!-- ✅ Don't send secret data as props — handle it server-side in LiveView -->
```

**Rule**: Never send sensitive data as props. Use LiveView's server-side rendering for anything that should stay hidden.

## Common Mistakes

```svelte
<!-- ❌ Don't set state that should come from the server -->
<script>
  let { number, live } = $props();
  function increase() {
    number++;  // Local only! Server doesn't know about this
  }
</script>

<!-- ✅ Push to server, let props flow back -->
<script>
  let { number, live } = $props();
  function increase() {
    live.pushEvent("increment", {}, () => {});
  }
</script>

<!-- ❌ Don't call live methods at top level (breaks SSR) -->
<script>
  let { live } = $props();
  live.handleEvent("update", () => {});  // Fails during SSR!
</script>

<!-- ✅ Wrap in onMount -->
<script>
  import { onMount } from "svelte";
  let { live } = $props();
  onMount(() => {
    live.handleEvent("update", () => {});
  });
</script>

<!-- ❌ Don't use $effect to derive state -->
<script>
  let items = $state([]);
  let count = $state(0);
  $effect(() => { count = items.length }); // Wrong!
</script>

<!-- ✅ Use $derived -->
<script>
  let items = $state([]);
  let count = $derived(items.length);
</script>

<!-- ❌ Don't forget keys in each blocks -->
{#each items as item}

<!-- ✅ Always provide a key -->
{#each items as item (item.id)}
```

## File Organization

```
assets/
├── svelte/
│   ├── Counter.svelte           # Simple components at root
│   ├── Chart.svelte
│   ├── components/              # Shared sub-components
│   │   ├── Button.svelte
│   │   └── Modal.svelte
│   └── dashboard/               # Feature directories
│       ├── DashboardStats.svelte
│       └── DashboardChart.svelte
```

Reference nested components in LiveView: `<.svelte name="dashboard/DashboardStats" ... />`
