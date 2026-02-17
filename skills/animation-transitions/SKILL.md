---
name: animation-transitions
description: Coordinating three animation systems in a Phoenix LiveView + Svelte stack — LiveView JS command transitions, Svelte transitions/motion, and CSS animations/keyframes. Covers which to use when, page transitions during LiveView navigation, reduced motion, and preventing jank. Use when adding animations, transitions, page transitions, or motion to a LiveView application.
---

# Animation & Transitions

Expert guidance for coordinating animations across LiveView JS commands, Svelte transitions, and CSS — three systems that must play nicely together.

## Decision Framework: Which Animation System

```
1. CSS transitions/keyframes   → Hover states, focus rings, color changes, simple reveals
2. LiveView JS.transition      → Patch-safe class animations tied to server events
3. Svelte transition:          → Enter/exit animations for Svelte-rendered elements
4. Svelte Spring/Tween         → Physics-based or interpolated values (drag, gestures)
5. Web Animations API          → Complex programmatic sequences (rare)
```

| System | Trigger | Survives Patch? | Runs Off Main Thread? | Best For |
|--------|---------|-----------------|----------------------|----------|
| CSS `transition` | Property change | Yes (if class persists) | Yes | Hover, focus, color, size |
| CSS `@keyframes` | Class add/remove | Yes (if class persists) | Yes | Loading spinners, pulse, shake |
| `JS.transition` | Server event / user action | Yes | Yes (CSS-based) | Flash highlights, modal open/close |
| `JS.show`/`JS.hide` with `transition` | Visibility toggle | Yes | Yes | Dropdowns, modals, tooltips |
| Svelte `transition:` | Element enter/exit | N/A (Svelte DOM) | Yes (CSS mode) | List items, conditional content |
| Svelte `in:`/`out:` | Directional enter/exit | N/A | Yes (CSS mode) | Different in vs out animations |
| Svelte `animate:flip` | List reorder | N/A | Yes | Sortable lists, drag-and-drop |
| Svelte `Spring`/`Tween` | Value change | N/A | No (JS tick) | Dragging, gestures, counters |

**Rule of thumb:** CSS for steady-state styling → JS commands for LiveView DOM events → Svelte transitions for Svelte components → Spring/Tween for continuous values.

## CSS Transitions and Keyframes

### Transitions for Interactive States

```css
/* Base interactive transition — apply to all interactive elements */
.interactive {
  transition: color var(--transition-fast),
              background-color var(--transition-fast),
              border-color var(--transition-fast),
              box-shadow var(--transition-fast);
}

/* Or use Tailwind's built-in transition utilities */
```

```heex
<button class="bg-brand-600 hover:bg-brand-700 transition-colors duration-150">
  Save
</button>

<input class="border border-border focus:border-brand-500 focus:ring-2 focus:ring-brand-500
              transition-[border-color,box-shadow] duration-150" />
```

### Keyframe Animations

```css
/* assets/css/animations.css */
@keyframes fade-in {
  from { opacity: 0; }
  to { opacity: 1; }
}

@keyframes slide-up {
  from { opacity: 0; transform: translateY(0.5rem); }
  to { opacity: 1; transform: translateY(0); }
}

@keyframes slide-down {
  from { opacity: 0; transform: translateY(-0.5rem); }
  to { opacity: 1; transform: translateY(0); }
}

@keyframes scale-in {
  from { opacity: 0; transform: scale(0.95); }
  to { opacity: 1; transform: scale(1); }
}

@keyframes shake {
  0%, 100% { transform: translateX(0); }
  25% { transform: translateX(-4px); }
  75% { transform: translateX(4px); }
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.5; }
}
```

### Tailwind Animation Utilities

Register custom animations in `tailwind.config.js`:

```javascript
// tailwind.config.js
export default {
  theme: {
    extend: {
      animation: {
        "fade-in": "fade-in 200ms ease-out",
        "slide-up": "slide-up 200ms ease-out",
        "slide-down": "slide-down 200ms ease-out",
        "scale-in": "scale-in 200ms ease-out",
        "shake": "shake 400ms ease-in-out",
      },
      keyframes: {
        "fade-in": {
          from: { opacity: "0" },
          to: { opacity: "1" },
        },
        "slide-up": {
          from: { opacity: "0", transform: "translateY(0.5rem)" },
          to: { opacity: "1", transform: "translateY(0)" },
        },
        "slide-down": {
          from: { opacity: "0", transform: "translateY(-0.5rem)" },
          to: { opacity: "1", transform: "translateY(0)" },
        },
        "scale-in": {
          from: { opacity: "0", transform: "scale(0.95)" },
          to: { opacity: "1", transform: "scale(1)" },
        },
        "shake": {
          "0%, 100%": { transform: "translateX(0)" },
          "25%": { transform: "translateX(-4px)" },
          "75%": { transform: "translateX(4px)" },
        },
      },
    },
  },
};
```

```heex
<div class="animate-fade-in">Fades in on mount</div>
<div class="animate-slide-up">Slides up on mount</div>
```

## LiveView JS Command Transitions

### The Transition Tuple

`JS.show`, `JS.hide`, `JS.toggle`, and `JS.transition` accept a transition option as either a string or a 3-tuple:

```elixir
# Simple: single class applied during transition
JS.show(transition: "fade-in")

# 3-tuple: {transition-class, start-class, end-class}
JS.show(
  transition: {"ease-out duration-300", "opacity-0", "opacity-100"}
)

JS.hide(
  transition: {"ease-in duration-200", "opacity-100", "opacity-0"}
)
```

The 3-tuple works like:
1. Apply `transition-class` + `start-class` to element
2. Next frame: remove `start-class`, add `end-class`
3. After transition completes: remove all transition classes

### Modal Pattern

```elixir
def show_modal(js \\ %JS{}) do
  js
  |> JS.show(
    to: "#modal-overlay",
    transition: {"ease-out duration-300", "opacity-0", "opacity-100"}
  )
  |> JS.show(
    to: "#modal-content",
    transition: {"ease-out duration-300", "opacity-0 translate-y-4 sm:translate-y-0 sm:scale-95",
                 "opacity-100 translate-y-0 sm:scale-100"}
  )
  |> JS.push_focus()
  |> JS.focus_first(to: "#modal-content")
end

def hide_modal(js \\ %JS{}) do
  js
  |> JS.hide(
    to: "#modal-content",
    transition: {"ease-in duration-200", "opacity-100 translate-y-0 sm:scale-100",
                 "opacity-0 translate-y-4 sm:translate-y-0 sm:scale-95"}
  )
  |> JS.hide(
    to: "#modal-overlay",
    transition: {"ease-in duration-200", "opacity-100", "opacity-0"}
  )
  |> JS.pop_focus()
end
```

### Flash Message Animation

```elixir
def show_flash(js \\ %JS{}, kind) do
  js
  |> JS.show(
    to: "#flash-#{kind}",
    transition: {"ease-out duration-300", "opacity-0 translate-y-2", "opacity-100 translate-y-0"}
  )
end

def hide_flash(js \\ %JS{}, kind) do
  js
  |> JS.hide(
    to: "#flash-#{kind}",
    transition: {"ease-in duration-200", "opacity-100 translate-y-0", "opacity-0 translate-y-2"}
  )
  |> JS.push("lv:clear-flash", value: %{key: kind})
end
```

### Highlight on Update (Server Push)

```elixir
# Server: push highlight event after update
def handle_info({:item_updated, item}, socket) do
  {:noreply, push_event(socket, "highlight", %{id: "item-#{item.id}"})}
end
```

```heex
<div id={"item-#{item.id}"} data-highlight={JS.transition("animate-pulse")}>
  {item.name}
</div>
```

```javascript
// app.js
window.addEventListener("phx:highlight", (e) => {
  const el = document.getElementById(e.detail.id);
  if (el) liveSocket.execJS(el, el.getAttribute("data-highlight"));
});
```

### Dropdown Pattern

```elixir
def toggle_dropdown(js \\ %JS{}) do
  js
  |> JS.toggle(
    to: "#dropdown-menu",
    in: {"ease-out duration-100", "opacity-0 scale-95", "opacity-100 scale-100"},
    out: {"ease-in duration-75", "opacity-100 scale-100", "opacity-0 scale-95"}
  )
  |> JS.toggle_attribute({"aria-expanded", "true", "false"}, to: "#dropdown-button")
end
```

```heex
<div class="relative">
  <button id="dropdown-button" phx-click={toggle_dropdown()} aria-expanded="false">
    Options
  </button>
  <div id="dropdown-menu" class="absolute right-0 mt-2 w-48 rounded-md shadow-lg bg-white ring-1 ring-black ring-opacity-5"
       phx-click-away={JS.hide(to: "#dropdown-menu", transition: {"ease-in duration-75", "opacity-100 scale-100", "opacity-0 scale-95"})}>
    ...
  </div>
</div>
```

### List Item Enter/Exit (`phx-mounted` / `phx-remove`)

LiveView provides two special bindings for animating element lifecycle:

- `phx-mounted` — fires when the element first appears in the DOM (after a server patch adds it)
- `phx-remove` — fires when LiveView is about to remove the element, and **delays removal** until the animation completes

```elixir
def fade_in_item(js \\ %JS{}, id) do
  JS.show(js,
    to: "##{id}",
    transition: {"ease-out duration-300", "opacity-0 translate-y-2", "opacity-100 translate-y-0"},
    time: 300
  )
end

def fade_out_item(js \\ %JS{}, id) do
  JS.hide(js,
    to: "##{id}",
    transition: {"ease-in duration-200", "opacity-100", "opacity-0 -translate-x-8"},
    time: 200
  )
end
```

```heex
<div
  :for={item <- @items}
  id={"item-#{item.id}"}
  phx-mounted={fade_in_item("item-#{item.id}")}
  phx-remove={fade_out_item("item-#{item.id}")}
>
  {item.name}
</div>
```

This is the cleanest way to animate stream inserts and deletes — the server controls
the list, and the client handles the visual transitions.

## Svelte Transitions

### Built-in Transitions

```svelte
<script>
  import { fade, fly, slide, scale, blur, draw, crossfade } from "svelte/transition";
  import { flip } from "svelte/animate";

  let visible = $state(false);
</script>

<!-- Bidirectional: same animation in and out, reversible -->
{#if visible}
  <div transition:fade={{ duration: 200 }}>Fades in and out</div>
{/if}

<!-- Directional: different in and out -->
{#if visible}
  <div in:fly={{ y: 20, duration: 300 }} out:fade={{ duration: 150 }}>
    Flies in, fades out
  </div>
{/if}

<!-- Slide from side -->
{#if visible}
  <div transition:slide={{ axis: "x", duration: 200 }}>Slides horizontally</div>
{/if}
```

### Local vs Global Transitions

Svelte 5 transitions are **local by default** — they only play when the element's own condition changes, not when a parent block is created/destroyed.

```svelte
{#if outerCondition}
  {#if innerCondition}
    <!-- Only plays when innerCondition changes, not outerCondition -->
    <p transition:fade>Local (default)</p>

    <!-- Plays when either condition changes -->
    <p transition:fade|global>Global</p>
  {/if}
{/if}
```

### Transition Events

```svelte
{#if visible}
  <div
    transition:fly={{ y: 200, duration: 300 }}
    onintrostart={() => console.log("intro started")}
    onintroend={() => console.log("intro ended")}
    onoutrostart={() => console.log("outro started")}
    onoutroend={() => console.log("outro ended")}
  >
    Content
  </div>
{/if}
```

### Crossfade (Shared Element Transitions)

```svelte
<script>
  import { crossfade } from "svelte/transition";
  import { quintOut } from "svelte/easing";

  const [send, receive] = crossfade({
    duration: 300,
    easing: quintOut,
    fallback: (node) => {
      return { duration: 200, css: (t) => `opacity: ${t}` };
    },
  });

  let items = $state([
    { id: 1, name: "Item A", done: false },
    { id: 2, name: "Item B", done: false },
  ]);

  function toggle(item) {
    item.done = !item.done;
  }
</script>

<div class="grid grid-cols-2 gap-4">
  <div>
    <h2>Todo</h2>
    {#each items.filter(i => !i.done) as item (item.id)}
      <div
        in:receive={{ key: item.id }}
        out:send={{ key: item.id }}
        animate:flip={{ duration: 200 }}
        onclick={() => toggle(item)}
      >
        {item.name}
      </div>
    {/each}
  </div>
  <div>
    <h2>Done</h2>
    {#each items.filter(i => i.done) as item (item.id)}
      <div
        in:receive={{ key: item.id }}
        out:send={{ key: item.id }}
        animate:flip={{ duration: 200 }}
        onclick={() => toggle(item)}
      >
        {item.name}
      </div>
    {/each}
  </div>
</div>
```

### List Reorder Animation (FLIP)

```svelte
<script>
  import { flip } from "svelte/animate";
  import { fade } from "svelte/transition";

  let { items } = $props();
</script>

{#each items as item (item.id)}
  <div
    animate:flip={{ duration: 300 }}
    in:fade={{ duration: 200 }}
    out:fade={{ duration: 150 }}
  >
    {item.name}
  </div>
{/each}
```

### Custom Transition Functions

```svelte
<script>
  import { cubicOut } from "svelte/easing";

  function whoosh(node, { duration = 400, easing = cubicOut } = {}) {
    const existingTransform = getComputedStyle(node).transform.replace("none", "");
    return {
      duration,
      easing,
      css: (t, u) => `transform: ${existingTransform} scale(${t}); opacity: ${t}`,
    };
  }
</script>

{#if visible}
  <div transition:whoosh={{ duration: 300 }}>Custom transition</div>
{/if}
```

## Svelte Motion (Spring & Tween)

For continuous, physics-based animation of values — not enter/exit.

### Spring

```svelte
<script>
  import { Spring } from "svelte/motion";

  let coords = new Spring({ x: 0, y: 0 }, { stiffness: 0.1, damping: 0.25 });
</script>

<svelte:window
  onmousemove={(e) => { coords.target = { x: e.clientX, y: e.clientY }; }}
/>

<div
  class="w-8 h-8 rounded-full bg-brand-500"
  style="transform: translate({coords.current.x}px, {coords.current.y}px)"
/>
```

### Tween

```svelte
<script>
  import { Tween } from "svelte/motion";
  import { cubicOut } from "svelte/easing";

  let { progress } = $props();

  const tweenedProgress = Tween.of(() => progress, {
    duration: 400,
    easing: cubicOut,
  });
</script>

<div class="h-2 bg-gray-200 rounded-full overflow-hidden">
  <div
    class="h-full bg-brand-500 rounded-full"
    style="width: {tweenedProgress.current}%"
  />
</div>
```

### When Spring vs Tween

| Use Spring | Use Tween |
|-----------|-----------|
| Drag-and-drop position | Progress bars |
| Cursor followers | Counter animations |
| Elastic/bouncy feel | Linear/eased interpolation |
| Unknown end time (user-driven) | Known duration |
| Physics-based motion | Precise timing control |

## Page Transitions in LiveView Navigation

LiveView navigation (`live_patch`, `live_navigate`) replaces content without a full page load. This makes traditional page transitions tricky.

### Strategy 1: CSS Animations on Mount

Apply animation classes that trigger on initial render:

```heex
<main class="animate-fade-in">
  {render_slot(@inner_block)}
</main>
```

Simple but doesn't animate the exit of the old page.

### Strategy 2: Transition on Navigation Events

Use `phx:page-loading-start` and `phx:page-loading-stop` events:

```javascript
// app.js
let topbar = document.querySelector("#topbar");

window.addEventListener("phx:page-loading-start", (_info) => {
  // Fade out current content
  document.querySelector("#page-content")?.classList.add("opacity-0", "transition-opacity", "duration-150");
});

window.addEventListener("phx:page-loading-stop", (_info) => {
  // Fade in new content
  const el = document.querySelector("#page-content");
  if (el) {
    el.classList.remove("opacity-0");
    el.classList.add("animate-fade-in");
  }
});
```

### Strategy 3: View Transitions API (Modern Browsers)

```javascript
// app.js — opt into View Transitions for LiveView navigation
if (document.startViewTransition) {
  let liveSocket = new LiveSocket("/live", Socket, {
    hooks: Hooks,
    dom: {
      onBeforeElUpdated(from, to) {
        // Preserve data-js-* attributes
        for (const attr of from.attributes) {
          if (attr.name.startsWith("data-js-")) {
            to.setAttribute(attr.name, attr.value);
          }
        }
      },
      onPatchStart(container) {
        document.startViewTransition(() => {
          return new Promise((resolve) => {
            // LiveView will call onPatchEnd when done
            container.__viewTransitionResolve = resolve;
          });
        });
      },
      onPatchEnd(container) {
        container.__viewTransitionResolve?.();
      }
    }
  });
}
```

```css
/* Control the view transition animation */
::view-transition-old(root) {
  animation: fade-out 150ms ease-in;
}

::view-transition-new(root) {
  animation: fade-in 200ms ease-out;
}
```

### Strategy 4: Svelte Page Transitions (Within LiveSvelte)

If your page content is a Svelte component, use Svelte transitions directly:

```svelte
<script>
  import { fade } from "svelte/transition";
  let { pageData } = $props();
</script>

{#key pageData.id}
  <div in:fade={{ duration: 200, delay: 100 }} out:fade={{ duration: 100 }}>
    <!-- Page content -->
  </div>
{/key}
```

## Cross-System Animation Coordination

### Boundary Rules

The fundamental rule: **each element has one animation owner.**

- Elements rendered by HEEx (Phoenix templates) → animated by JS commands or CSS
- Elements rendered by Svelte (`{#if}`, `{#each}`) → animated by Svelte transitions
- Elements that exist in both contexts → use CSS as the neutral ground

Never do this:
```svelte
<!-- BAD: Svelte trying to transition a LiveView-managed element -->
<div id="liveview-element" transition:fade>...</div>
```

LiveView will patch this element and Svelte's transition state will be lost or conflict.

### Shared Timing Tokens

Define animation timing as CSS custom properties so both systems use the same durations:

```css
:root {
  /* Duration tokens */
  --duration-fast: 150ms;
  --duration-normal: 200ms;
  --duration-slow: 300ms;
  --duration-slower: 500ms;

  /* Easing tokens */
  --ease-in: cubic-bezier(0.4, 0, 1, 1);
  --ease-out: cubic-bezier(0, 0, 0.2, 1);
  --ease-in-out: cubic-bezier(0.4, 0, 0.2, 1);
  --ease-bounce: cubic-bezier(0.34, 1.56, 0.64, 1);
}
```

Reference in Svelte:
```svelte
<script>
  const DURATION_NORMAL = 200; // Matches --duration-normal
  const DURATION_SLOW = 300;   // Matches --duration-slow
</script>

{#if show}
  <div transition:fly={{ y: 10, duration: DURATION_NORMAL }}>...</div>
{/if}
```

Reference in Phoenix JS commands:
```elixir
# time: values should match the CSS duration tokens
@duration_fast 150
@duration_normal 200
@duration_slow 300

def fade_in(js \\ %JS{}, selector) do
  JS.show(js, to: selector,
    transition: {"ease-out duration-200", "opacity-0", "opacity-100"},
    time: @duration_normal
  )
end
```

### LiveView ↔ Svelte Event Handoffs

#### LiveView Triggers Svelte Animation

```elixir
# Server pushes event to client
def handle_event("open_editor", %{"id" => id}, socket) do
  {:noreply, push_event(socket, "editor:open", %{id: id})}
end
```

```javascript
// Hook receives event and bridges to Svelte via CustomEvent
const EditorBridge = {
  mounted() {
    this.handleEvent("editor:open", ({id}) => {
      window.dispatchEvent(new CustomEvent("editor:open", { detail: { id } }));
    });
  }
};
```

```svelte
<script>
  import { fly } from "svelte/transition";
  import { onMount } from "svelte";

  let isOpen = $state(false);
  let editorId = $state(null);

  onMount(() => {
    const handler = (e) => {
      editorId = e.detail.id;
      isOpen = true;
    };
    window.addEventListener("editor:open", handler);
    return () => window.removeEventListener("editor:open", handler);
  });
</script>

{#if isOpen}
  <div transition:fly={{ x: 300, duration: 300 }}>
    <!-- Svelte-owned editor component -->
  </div>
{/if}
```

#### Svelte Triggers LiveView Animation

```svelte
<script>
  function closePanel() {
    const hook = document.querySelector("[phx-hook='PanelBridge']").__phxHook;
    hook.pushEvent("panel:closed", {});
  }
</script>

<button onclick={closePanel}>Close</button>
```

```elixir
def handle_event("panel:closed", _params, socket) do
  {:noreply,
    socket
    |> push_event("animate:panel-close", %{})
    |> assign(panel_open: false)
  }
end
```

#### Server-Triggered Svelte Tween

```elixir
# Server pushes animation trigger
def handle_info({:score_updated, score}, socket) do
  {:noreply,
   socket
   |> assign(:score, score)
   |> push_event("score-changed", %{score: score})}
end
```

```svelte
<script>
  import { Tween } from "svelte/motion";
  import { cubicOut } from "svelte/easing";

  let { score, live } = $props();

  const displayScore = new Tween(score, { duration: 600, easing: cubicOut });

  // React to prop changes from LiveView
  $effect(() => {
    displayScore.target = score;
  });
</script>

<span class="text-4xl font-bold tabular-nums">
  {Math.round(displayScore.current)}
</span>
```

### Common Coordination Patterns

#### LiveView Modal with Svelte Content

LiveView handles the modal shell animation; Svelte handles internal animations:

```elixir
def show_modal(js \\ %JS{}) do
  js
  |> JS.show(to: "#modal-overlay", transition: {"ease-out duration-300", "opacity-0", "opacity-100"})
  |> JS.show(to: "#modal-content", transition: {"ease-out duration-300", "opacity-0 scale-95", "opacity-100 scale-100"})
end
```

```heex
<div id="modal-overlay" class="fixed inset-0 bg-black/50 hidden" />
<div id="modal-content" class="fixed inset-0 flex items-center justify-center hidden">
  <div class="bg-white rounded-lg shadow-xl max-w-lg w-full p-6">
    <%!-- Svelte handles internal animations --%>
    <.svelte name="DataEditor" props={%{data: @data}} socket={@socket} />
  </div>
</div>
```

#### Staggered List with Mixed Rendering

LiveView renders the list structure; items that need rich interactivity are Svelte islands:

```heex
<div id="items" phx-update="stream">
  <div
    :for={{dom_id, item} <- @streams.items}
    id={dom_id}
    phx-mounted={JS.transition({"ease-out duration-300", "opacity-0 translate-y-2", "opacity-100 translate-y-0"}, time: 300)}
    phx-remove={JS.hide(transition: {"ease-in duration-200", "opacity-100", "opacity-0"}, time: 200)}
  >
    <%!-- Simple items: Phoenix renders directly --%>
    <div :if={item.type == "simple"} class="p-md">
      {item.name}
    </div>
    <%!-- Complex items: Svelte island handles internal state and animation --%>
    <div :if={item.type == "interactive"} phx-hook="InteractiveItem" data-item-id={item.id}>
    </div>
  </div>
</div>
```

#### Shared Loading State

Both systems show loading indicators. Coordinate via CSS classes — CSS is the neutral ground:

```css
/* Global loading overlay, triggered by either system */
.app-loading .loading-indicator { display: flex; }
.app-loading .content-area { opacity: 0.5; pointer-events: none; }
```

LiveView sets the class via JS:
```elixir
JS.add_class("app-loading", to: "#app")
```

Svelte sets it via DOM:
```svelte
<script>
  function startLoading() {
    document.getElementById("app")?.classList.add("app-loading");
  }
  function stopLoading() {
    document.getElementById("app")?.classList.remove("app-loading");
  }
</script>
```

Both frameworks manipulate classes; CSS handles the visual.

## Reduced Motion

**Always respect `prefers-reduced-motion`.** This is an accessibility requirement.

### CSS

```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

Or in Tailwind:

```heex
<div class="animate-slide-up motion-reduce:animate-none">Content</div>
```

### Svelte

```svelte
<script>
  import { prefersReducedMotion } from "svelte/motion";
  import { fly, fade } from "svelte/transition";

  let visible = $state(false);
</script>

{#if visible}
  <div
    transition:fly={{
      y: prefersReducedMotion.current ? 0 : 200,
      duration: prefersReducedMotion.current ? 0 : 300
    }}
  >
    Respects motion preferences
  </div>
{/if}
```

### LiveView JS Commands

```elixir
# Use shorter durations that can be overridden by CSS media query
def show_modal(js \\ %JS{}) do
  js
  |> JS.show(
    to: "#modal",
    transition: {"ease-out duration-300 motion-reduce:duration-0",
                 "opacity-0 scale-95",
                 "opacity-100 scale-100"}
  )
end
```

## Performance Guidelines

### Animate Only Composite Properties

Cheap (GPU-composited):
- `transform` (translate, scale, rotate)
- `opacity`
- `filter` (blur, brightness)

Expensive (trigger layout/paint):
- `width`, `height`, `top`, `left`
- `margin`, `padding`
- `border-width`
- `font-size`

```css
/* ✅ Good: only transforms and opacity */
.slide-in {
  animation: slide-in 200ms ease-out;
}
@keyframes slide-in {
  from { transform: translateY(10px); opacity: 0; }
  to { transform: translateY(0); opacity: 1; }
}

/* ❌ Bad: animating height triggers layout on every frame */
@keyframes expand {
  from { height: 0; }
  to { height: auto; }
}
```

### Svelte: Prefer CSS Mode Over Tick

```svelte
<script>
  // ✅ CSS mode — runs off main thread
  function goodTransition(node, { duration = 300 }) {
    return {
      duration,
      css: (t) => `opacity: ${t}; transform: scale(${0.95 + 0.05 * t})`,
    };
  }

  // ❌ Tick mode — runs on main thread, can cause jank
  function badTransition(node, { duration = 300 }) {
    return {
      duration,
      tick: (t) => {
        node.style.opacity = t;
        node.style.transform = `scale(${0.95 + 0.05 * t})`;
      },
    };
  }
</script>
```

### Stagger Animations

Don't animate everything at once. Stagger for perceived performance:

```heex
<div :for={{dom_id, item} <- @streams.items} id={dom_id}
     class="animate-slide-up"
     style={"animation-delay: #{item.index * 50}ms; animation-fill-mode: backwards;"}>
  {item.name}
</div>
```

```svelte
{#each items as item, i (item.id)}
  <div
    in:fly={{ y: 20, duration: 200, delay: i * 50 }}
    out:fade={{ duration: 100 }}
  >
    {item.name}
  </div>
{/each}
```

### Scroll-Triggered Animations (IntersectionObserver Hook)

When JS commands aren't enough, use a hook with `IntersectionObserver` for
scroll-triggered reveal animations:

```javascript
const AnimateOnScroll = {
  mounted() {
    this._observer = new IntersectionObserver((entries) => {
      entries.forEach(entry => {
        if (entry.isIntersecting) {
          entry.target.classList.add("animate-slide-up-fade");
          this._observer.unobserve(entry.target);
        }
      });
    }, { threshold: 0.1 });

    this.el.querySelectorAll("[data-animate]").forEach(el => {
      this._observer.observe(el);
    });
  },
  destroyed() {
    this._observer?.disconnect();
  }
};
```

```heex
<div id="scroll-container" phx-hook="AnimateOnScroll">
  <div data-animate class="opacity-0">First item</div>
  <div data-animate class="opacity-0">Second item</div>
</div>
```

Hook lifecycle callbacks relevant to animation:
- `mounted()` — element added to DOM, start entrance animations
- `updated()` — element patched, animate changes
- `destroyed()` — element about to be removed, clean up observers/timers
- `disconnected()` — connection lost, consider pausing animations
- `reconnected()` — connection restored, resume

## Anti-Patterns

### Animating LiveView-patched elements without JS commands

```heex
<!-- ❌ CSS animation replays on every server patch -->
<div class="animate-fade-in">{@content}</div>

<!-- ✅ Use JS.transition for one-shot animations triggered by events -->
<div id="content" data-highlight={JS.transition("animate-pulse")}>{@content}</div>
```

### Using Svelte transitions on phx-update="ignore" containers

```heex
<!-- ❌ Svelte transitions won't work — LiveView controls this DOM -->
<div phx-update="ignore">
  <.svelte name="AnimatedList" ... />  <!-- Svelte can't animate enter/exit -->
</div>

<!-- ✅ Let Svelte own its DOM entirely via LiveSvelte -->
<.svelte name="AnimatedList" props={%{items: @items}} socket={@socket} />
```

### Mixing transition systems on the same element

```
❌ CSS transition + JS.transition on the same property = race condition
❌ Svelte transition: + manual classList.add in onMount = conflict
✅ One animation system per element per property
```

### Forgetting animation-fill-mode

```css
/* ❌ Element snaps back to original state after animation */
.animate-slide-up {
  animation: slide-up 200ms ease-out;
}

/* ✅ Element stays at final state */
.animate-slide-up {
  animation: slide-up 200ms ease-out forwards;
}
```

### Over-animating

```
❌ Every element bounces, slides, and fades on every interaction
✅ Animate meaningful state changes: enter, exit, error, success, reorder
   Keep hover/focus transitions subtle (150ms, ease)
   Reserve dramatic animations for key moments (modal open, page transition)
```

## Related Skills

- **liveview-js-interop**: JS commands, hooks, DOM patching survival
- **svelte-core**: LiveSvelte integration, reactivity, `live.pushEvent`
- **component-design-system**: Design tokens, Tailwind config for animation values
- **frontend-design**: Motion principles, aesthetic direction
- **frontend-tailwind**: Tailwind transition/animation utilities
