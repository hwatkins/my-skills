---
name: liveview-js-interop
description: When the user needs JavaScript interoperability in Phoenix LiveView. Covers JS commands, hooks, colocated hooks (LV 1.1+), DOM patching survival, server/client communication, third-party library integration, and the LiveSvelte boundary. Use when mentioning "hooks," "phx-hook," "JS commands," "JS.push," "pushEvent," "handleEvent," "phx-update ignore," "colocated hooks," or "JavaScript in LiveView."
---

# LiveView JavaScript Interoperability

Expert guidance for integrating JavaScript with Phoenix LiveView — from zero-JS declarative commands to full third-party library integration.

## Decision Framework: When to Use What

Pick the simplest tool that solves the problem. Escalate only when needed.

```
1. phx- bindings + JS commands  → No JS file needed. Declarative, patch-safe.
2. JS.dispatch + addEventListener → Custom client events without hooks.
3. Colocated hooks (LV 1.1+)    → Small JS tied to a specific component.
4. Traditional phx-hook          → Reusable hooks shared across views.
5. LiveSvelte component          → Complex reactive UI (editors, charts, drag-and-drop).
```

| Approach | JS File? | Patch-Safe? | Reusable? | Best For |
|----------|----------|-------------|-----------|----------|
| `phx-click`, `phx-change` | No | Yes | N/A | Standard events |
| `JS.*` commands | No | Yes | Via helpers | Show/hide, transitions, class toggles, push with loading |
| `JS.dispatch` + `addEventListener` | Minimal | Yes | Yes | Clipboard, custom DOM events |
| Colocated hook | Inline | Manual | No (component-scoped) | Component-specific DOM control |
| Traditional hook | Separate file | Manual | Yes | Shared hooks, third-party libs |
| LiveSvelte | Svelte file | Yes (own DOM) | Yes | Rich interactive components |

**Rule of thumb:** Exhaust JS commands → try colocated hooks → then traditional hooks → then LiveSvelte.

### When to Promote a Hook to LiveSvelte

- The hook manages significant local state beyond simple DOM manipulation
- You're building HTML strings in JavaScript (`el.innerHTML = ...`)
- You need reactive rendering (conditionals, loops) based on client-side data
- The interaction has multiple interdependent UI elements
- You want scoped CSS and component-level encapsulation
- The hook code exceeds ~50 lines of DOM manipulation

### When to Keep It as a Hook

- Thin bridge to a third-party library (initialize, teardown, forward events)
- Pure side effect with no rendering (clipboard, scroll position, local storage)
- Single-element behavior (input formatting, focus management)
- The functionality is needed in dead views too (`mounted` works without LiveView)

## JS Commands Deep Dive

`Phoenix.LiveView.JS` provides client-side operations that **survive DOM patches** — unlike raw JS mutations which get clobbered.

### Available Commands

```elixir
alias Phoenix.LiveView.JS

# Visibility
JS.show(to: "#modal")
JS.hide(to: "#modal", transition: "fade-out")
JS.toggle(to: "#dropdown")

# Classes
JS.add_class("active", to: "#tab-1")
JS.remove_class("active", to: ".tab", transition: "fade-out")
JS.toggle_class("open", to: "#menu")

# Attributes
JS.set_attribute({"aria-expanded", "true"}, to: "#dropdown")
JS.remove_attribute("disabled", to: "#submit-btn")
JS.toggle_attribute({"aria-expanded", "true", "false"}, to: "#dropdown")

# Transitions
JS.transition("shake", to: "#error-msg")

# Focus
JS.focus(to: "#search-input")
JS.focus_first(to: "#modal")
JS.push_focus(to: "#search-input")  # Save current focus, restore with pop_focus
JS.pop_focus()

# Navigation
JS.navigate("/users/#{user.id}")
JS.patch("/users?page=2")

# Server push
JS.push("increment", value: %{id: @id}, target: @myself, loading: "#counter")

# Execute stored commands
JS.exec("data-cancel", to: "#modal")

# Custom DOM events
JS.dispatch("my_app:clipcopy", to: "#code-block")
```

### Composing Commands

Chain commands with `|>` — they execute in order on the client.

```elixir
def hide_modal(js \\ %JS{}) do
  js
  |> JS.hide(transition: "fade-out", to: "#modal-overlay")
  |> JS.hide(transition: "fade-out-scale", to: "#modal-content")
  |> JS.pop_focus()
end

def show_modal(js \\ %JS{}) do
  js
  |> JS.show(transition: "fade-in", to: "#modal-overlay")
  |> JS.show(
    transition: {"ease-out duration-300", "opacity-0 translate-y-4", "opacity-100 translate-y-0"},
    to: "#modal-content"
  )
  |> JS.push_focus()
  |> JS.focus_first(to: "#modal-content")
end
```

```heex
<button phx-click={show_modal()}>Open</button>
<button phx-click={hide_modal()}>Close</button>
<div phx-click-away={hide_modal()} phx-window-keydown={hide_modal()} phx-key="escape">
  ...
</div>
```

### JS.push with Loading States

```heex
<button phx-click={
  JS.push("save", loading: "#form-container", value: %{id: @item.id})
  |> JS.add_class("saving", to: "#save-btn")
}>
  Save
</button>
```

The `loading` option adds `phx-click-loading` class to the target element while awaiting server response.

### JS.dispatch for Custom Events

Trigger custom DOM events without hooks — handle them with `window.addEventListener` in `app.js`.

```elixir
# In your template
<button phx-click={JS.dispatch("my_app:clipcopy", to: "#code-block")}>
  Copy to clipboard
</button>
<pre id="code-block"><code>{@code}</code></pre>
```

```javascript
// In app.js — runs once, handles all dispatches
window.addEventListener("my_app:clipcopy", (e) => {
  if ("clipboard" in navigator) {
    const text = e.target.textContent;
    navigator.clipboard.writeText(text);
  }
});
```

### JS.exec for Stored Commands

Store JS commands in data attributes and execute them from hooks or other commands.

```heex
<div id="notification"
     data-show={JS.show(transition: "slide-in")}
     data-hide={JS.hide(transition: "slide-out")}>
  {@message}
</div>
```

```javascript
// From a hook
this.js().exec(this.el.getAttribute("data-show"));

// From window event listener
liveSocket.execJS(el, el.getAttribute("data-hide"));
```

### DOM Selectors

```elixir
# Standard CSS selectors
JS.hide(to: "#modal")
JS.add_class("active", to: ".tab:first-child")

# Scoped selectors (relative to interacted element)
JS.show(to: {:inner, ".menu"})       # Child of clicked element
JS.hide(to: {:closest, ".dropdown"}) # Nearest ancestor
```

## Hook Lifecycle and Patterns

### The Six Callbacks

```javascript
Hooks.MyHook = {
  // Element added to DOM, LiveView mounted
  mounted() {
    // Initialize: add listeners, create instances, read data attributes
    this.chart = new Chart(this.el, { /* ... */ });
    this.handleEvent("update-data", ({ data }) => {
      this.chart.update(data);
    });
  },

  // Element about to be updated (synchronous only!)
  beforeUpdate() {
    // Save state that will be lost after DOM patch
    this._scrollTop = this.el.scrollTop;
  },

  // Element updated by server
  updated() {
    // Restore state saved in beforeUpdate
    this.el.scrollTop = this._scrollTop;
  },

  // Element removed from page
  destroyed() {
    // CLEANUP: remove listeners, destroy instances, clear timers
    this.chart.destroy();
    if (this._interval) clearInterval(this._interval);
    if (this._observer) this._observer.disconnect();
  },

  // LiveView disconnected from server
  disconnected() {
    // Optional: show offline indicator
  },

  // LiveView reconnected to server
  reconnected() {
    // Optional: refresh data
  }
};
```

### Hook Scope — Available Properties

| Property | Description |
|----------|-------------|
| `this.el` | The bound DOM element |
| `this.liveSocket` | The underlying LiveSocket instance |
| `this.pushEvent(event, payload, callback?)` | Push event to server; callback receives reply |
| `this.pushEventTo(selectorOrTarget, event, payload, callback?)` | Push to specific LiveComponent |
| `this.handleEvent(event, callback)` | Listen for server-pushed events; returns ref for removal |
| `this.removeHandleEvent(ref)` | Remove a handler registered with `handleEvent` |
| `this.upload(name, files)` | Inject files into an uploader |
| `this.uploadTo(selectorOrTarget, name, files)` | Inject files into a specific uploader |
| `this.js()` | Returns JS command interface for patch-safe DOM manipulation |

**`pushEvent` without callback returns a Promise:**

```javascript
const { reply } = await this.pushEvent("get_data", { id: 42 });
console.log(reply.result);
```

### ViewHook Subclass (Alternative)

```javascript
import { ViewHook } from "phoenix_live_view";

class MyHook extends ViewHook {
  mounted() {
    // Same lifecycle, class-based syntax
  }
}

let liveSocket = new LiveSocket("/live", Socket, {
  hooks: { MyHook }
});
```

### The id Requirement

**Every hooked element MUST have a unique, stable `id`.**

```heex
<!-- ✅ Good: stable id -->
<div id={"chart-#{@chart_id}"} phx-hook="Chart">...</div>

<!-- ❌ Bad: no id (silent failure — hook never mounts) -->
<div phx-hook="Chart">...</div>

<!-- ❌ Bad: dynamic id that changes on re-render -->
<div id={System.unique_integer()} phx-hook="Chart">...</div>
```

For elements inside streams, use the stream item's DOM id:

```heex
<div :for={{dom_id, item} <- @streams.items} id={dom_id} phx-hook="ItemHook">
  ...
</div>
```

### Cleanup Discipline

**Always clean up in `destroyed`.** Memory leaks are the #1 hook bug.

```javascript
Hooks.Resizable = {
  mounted() {
    // Store references for cleanup
    this._onResize = () => this.pushEvent("viewport-changed", {
      width: window.innerWidth,
      height: window.innerHeight
    });
    window.addEventListener("resize", this._onResize);

    this._interval = setInterval(() => this.pushEvent("heartbeat", {}), 30000);

    this._observer = new IntersectionObserver((entries) => {
      entries.forEach(e => {
        if (e.isIntersecting) this.pushEvent("visible", { id: this.el.id });
      });
    });
    this._observer.observe(this.el);
  },

  destroyed() {
    window.removeEventListener("resize", this._onResize);
    clearInterval(this._interval);
    this._observer.disconnect();
  }
};
```

### Storing State

Store local state on `this` — it persists across `beforeUpdate`/`updated` cycles.

```javascript
Hooks.ScrollPosition = {
  mounted() {
    this._lastScrollTop = 0;
  },
  beforeUpdate() {
    this._lastScrollTop = this.el.scrollTop;
  },
  updated() {
    this.el.scrollTop = this._lastScrollTop;
  }
};
```

## DOM Patching Survival

LiveView replaces DOM on every server update. Any JS-applied mutations (classes, attributes, innerHTML) get clobbered.

### Why JS State Gets Clobbered

```
Server renders HTML → Client receives diff → morphdom patches DOM → Your JS changes vanish
```

### Solutions (Ranked by Preference)

**1. JS commands (best)** — patch-aware by design, changes stick across patches.

```heex
<!-- This class toggle survives patches -->
<button phx-click={JS.toggle_class("active", to: "#sidebar")}>Toggle</button>
```

**2. data-* attributes as bridge** — store state the server can see.

```heex
<div id="player" phx-hook="AudioPlayer" data-volume={@volume} data-playing={@playing}>
```

```javascript
Hooks.AudioPlayer = {
  mounted() {
    this.player = new Audio();
    this._applyState();
  },
  updated() {
    // Server changed data attributes — re-apply
    this._applyState();
  },
  _applyState() {
    this.player.volume = parseFloat(this.el.dataset.volume);
    if (this.el.dataset.playing === "true") {
      this.player.play();
    } else {
      this.player.pause();
    }
  }
};
```

**3. beforeUpdate/updated dance** — save and restore state across patches.

```javascript
Hooks.PreserveScroll = {
  beforeUpdate() {
    this._scrollPos = { top: this.el.scrollTop, left: this.el.scrollLeft };
  },
  updated() {
    this.el.scrollTop = this._scrollPos.top;
    this.el.scrollLeft = this._scrollPos.left;
  }
};
```

**4. `phx-update="ignore"`** — nuclear option. LiveView stops patching this subtree entirely.

```heex
<!-- LiveView will NEVER update anything inside this div -->
<div id="map-container" phx-hook="Map" phx-update="ignore">
  <div id="map" style="height: 400px;"></div>
</div>
```

**When to use `phx-update="ignore"`:**
- Third-party libraries that manage their own DOM (maps, rich text editors, charts)
- The container will never need server-driven content updates

**When NOT to use it:**
- You need some elements inside to update from the server
- You're using it to "fix" a patching issue (find the real cause instead)

**5. `onBeforeElUpdated` callback** — preserve specific attributes across patches.

```javascript
// In app.js — preserve data-js-* attributes during patches
let liveSocket = new LiveSocket("/live", Socket, {
  dom: {
    onBeforeElUpdated(from, to) {
      for (const attr of from.attributes) {
        if (attr.name.startsWith("data-js-")) {
          to.setAttribute(attr.name, attr.value);
        }
      }
    }
  }
});
```

**6. `JS.ignore_attributes`** — mark specific attributes as ignored during patching (LV 1.1+).

```heex
<div id="my-el" {JS.ignore_attributes(["data-state", "class"])}>
  ...
</div>
```

## Server ↔ Client Communication

### Client → Server: pushEvent

```javascript
// Basic push
this.pushEvent("item-clicked", { id: 42 });

// With reply callback
this.pushEvent("validate", { email: value }, (reply) => {
  if (reply.valid) {
    this.el.classList.remove("error");
  } else {
    this.el.classList.add("error");
    this.el.querySelector(".error-msg").textContent = reply.message;
  }
});

// Push returns a promise if no callback given
const reply = await this.pushEvent("validate", { email: value });
```

```elixir
# Server: handle_event with reply
def handle_event("validate", %{"email" => email}, socket) do
  case Accounts.validate_email(email) do
    :ok -> {:reply, %{valid: true}, socket}
    {:error, msg} -> {:reply, %{valid: false, message: msg}, socket}
  end
end
```

### Client → Specific LiveComponent: pushEventTo

```javascript
// Target by selector
this.pushEventTo("#user-form", "validate", { field: "email", value: "test@example.com" });

// Target by element reference (preferred — avoids selector issues)
this.pushEventTo(this.el, "validate", { field: "email", value: "test@example.com" });
```

### Server → Client: push_event + handleEvent

```elixir
# Server pushes event
def handle_info({:new_points, points}, socket) do
  {:noreply, push_event(socket, "chart-update", %{points: points})}
end
```

```javascript
Hooks.Chart = {
  mounted() {
    this.chart = new Chart(this.el, { /* config */ });

    // Listen for server-pushed events
    this.handleEvent("chart-update", ({ points }) => {
      this.chart.data.datasets[0].data = points;
      this.chart.update();
    });
  }
};
```

**Important:** `push_event` is **global** — all active hooks handling that event name will receive it. Namespace events when using LiveComponents:

```elixir
# In a LiveComponent — namespace with component id
def update(%{id: id, points: points} = assigns, socket) do
  socket =
    socket
    |> assign(assigns)
    |> push_event("points-#{id}", %{points: points})

  {:ok, socket}
end
```

```javascript
Hooks.Chart = {
  mounted() {
    this.handleEvent(`points-${this.el.id}`, ({ points }) => {
      this.chart.update(points);
    });
  }
};
```

### Server → Client via Window Events

For simple cases where you don't need a hook, server-pushed events are dispatched as DOM events with a `phx:` prefix:

```elixir
{:noreply, push_event(socket, "highlight", %{id: "item-#{item.id}"})}
```

```javascript
// In app.js — no hook needed
window.addEventListener("phx:highlight", (e) => {
  const el = document.getElementById(e.detail.id);
  if (el) el.classList.add("highlight");
});
```

### handleEvent Timing

If the server pushes an event and renders content in the same response, `handleEvent` callbacks fire **after** the page updates. If the server redirects at the same time, callbacks won't fire on the old page — they fire on the new page's newly mounted hooks.

### Debouncing Pushes from Hooks

```javascript
Hooks.SearchInput = {
  mounted() {
    this._debounceTimer = null;

    this.el.addEventListener("input", (e) => {
      clearTimeout(this._debounceTimer);
      this._debounceTimer = setTimeout(() => {
        this.pushEvent("search", { query: e.target.value });
      }, 300);
    });
  },

  destroyed() {
    clearTimeout(this._debounceTimer);
  }
};
```

## Colocated Hooks (LiveView 1.1+ / Phoenix 1.8+)

Define hooks inline with the component — no separate JS file needed.

### Basic Colocated Hook

```elixir
def phone_input(assigns) do
  ~H"""
  <input
    type="text"
    name="user[phone]"
    id="user-phone"
    phx-hook=".PhoneFormat"
  />
  <script :type={Phoenix.LiveView.ColocatedHook} name=".PhoneFormat">
    export default {
      mounted() {
        this.el.addEventListener("input", (e) => {
          let match = this.el.value.replace(/\D/g, "").match(/^(\d{3})(\d{3})(\d{4})$/);
          if (match) {
            this.el.value = `${match[1]}-${match[2]}-${match[3]}`;
          }
        });
      }
    }
  </script>
  """
end
```

### Key Rules

- Hook name **must start with a dot**: `name=".MyHook"`, `phx-hook=".MyHook"`
- LiveView auto-prefixes with the module name at compile time (e.g., `MyAppWeb.Components.PhoneFormat`)
- The `<script>` tag is **removed** from rendered output — code is extracted at compile time
- Colocated hooks are **component-scoped** — not reusable across modules (use traditional hooks for that)

### Setup (app.js)

```javascript
import { hooks as colocatedHooks } from "phoenix-colocated/my_app";

let liveSocket = new LiveSocket("/live", Socket, {
  hooks: { ...colocatedHooks, ...myTraditionalHooks }
});
```

### esbuild Configuration

Requires `{:esbuild, "~> 0.10"}` or later. Add `Mix.Project.build_path()` to `NODE_PATH`:

```elixir
config :esbuild,
  my_app: [
    args: ~w(js/app.js --bundle --target=es2022 --outdir=../priv/static/assets/js
             --external:/fonts/* --external:/images/* --alias:@=.),
    cd: Path.expand("../assets", __DIR__),
    env: %{
      "NODE_PATH" => [
        Path.expand("../deps", __DIR__),
        Mix.Project.build_path()
      ]
    }
  ]
```

### Colocated JS (Non-Hook Scripts)

For JS that isn't a hook (e.g., web components, utility scripts):

```elixir
<script :type={Phoenix.LiveView.ColocatedJS} name="my_utils">
  export function formatCurrency(amount) {
    return new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' }).format(amount);
  }
</script>
```

### Runtime Hooks

For cases where bundling isn't possible (e.g., LiveDashboard custom pages):

```elixir
<script :type={Phoenix.LiveView.ColocatedHook} name=".MyHook" runtime>
  {
    mounted() {
      // No export default — just the object literal
      this.el.textContent = "Hook mounted!";
    }
  }
</script>
```

Runtime hooks are **not processed by esbuild** — they run directly in the browser. Use only browser-native JS.

### When Colocated vs. Traditional

| Use Colocated When | Use Traditional When |
|--------------------|---------------------|
| Hook is specific to one component | Hook is shared across multiple views |
| Small, focused behavior | Complex, multi-file logic |
| No external dependencies | Needs npm packages |
| Phoenix 1.8+ project | Any Phoenix version |

## Third-Party JS Library Integration

### The General Pattern

```javascript
Hooks.ThirdPartyLib = {
  mounted() {
    // 1. Initialize the library
    this.instance = new Library(this.el, this._getConfig());

    // 2. Bridge: library events → server
    this.instance.on("change", (data) => {
      this.pushEvent("lib-changed", data);
    });

    // 3. Bridge: server events → library
    this.handleEvent(`update-${this.el.id}`, (data) => {
      this.instance.update(data);
    });
  },

  // 4. Teardown
  destroyed() {
    this.instance.destroy();
  },

  _getConfig() {
    // Read config from data attributes
    return JSON.parse(this.el.dataset.config || "{}");
  }
};
```

### Libraries That Manage Their Own DOM

Maps, charts, rich text editors — they create and manage DOM nodes internally. **Always use `phx-update="ignore"` on their container.**

```heex
<div id={"map-#{@id}"} phx-hook="LeafletMap" phx-update="ignore"
     data-lat={@lat} data-lng={@lng} data-zoom={@zoom}>
  <div id={"map-canvas-#{@id}"} style="height: 400px; width: 100%;"></div>
</div>
```

```javascript
Hooks.LeafletMap = {
  mounted() {
    const { lat, lng, zoom } = this.el.dataset;
    this.map = L.map(this.el.querySelector("[id$='-canvas']") || this.el.firstElementChild)
      .setView([lat, lng], zoom);

    L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png").addTo(this.map);

    this.handleEvent(`map-markers-${this.el.id}`, ({ markers }) => {
      markers.forEach(m => L.marker([m.lat, m.lng]).addTo(this.map));
    });

    this.map.on("moveend", () => {
      const center = this.map.getCenter();
      this.pushEvent("map-moved", { lat: center.lat, lng: center.lng });
    });
  },

  destroyed() {
    this.map.remove();
  }
};
```

### Chart.js Example

```heex
<div id={"chart-#{@id}"} phx-hook="ChartJS" phx-update="ignore"
     data-config={Jason.encode!(@chart_config)}>
  <canvas></canvas>
</div>
```

```javascript
import Chart from "chart.js/auto";

Hooks.ChartJS = {
  mounted() {
    const config = JSON.parse(this.el.dataset.config);
    const canvas = this.el.querySelector("canvas");
    this.chart = new Chart(canvas, config);

    this.handleEvent(`chart-data-${this.el.id}`, ({ datasets }) => {
      this.chart.data.datasets = datasets;
      this.chart.update();
    });
  },

  destroyed() {
    this.chart.destroy();
  }
};
```

### Rich Text Editor (TipTap/ProseMirror)

```javascript
import { Editor } from "@tiptap/core";
import StarterKit from "@tiptap/starter-kit";

Hooks.RichEditor = {
  mounted() {
    this.editor = new Editor({
      element: this.el.querySelector(".editor-content"),
      extensions: [StarterKit],
      content: this.el.dataset.content || "",
      onUpdate: ({ editor }) => {
        // Debounce pushes to server
        clearTimeout(this._saveTimer);
        this._saveTimer = setTimeout(() => {
          this.pushEvent("editor-updated", { html: editor.getHTML() });
        }, 500);
      }
    });

    this.handleEvent(`set-content-${this.el.id}`, ({ html }) => {
      this.editor.commands.setContent(html, false); // false = don't emit update
    });
  },

  destroyed() {
    clearTimeout(this._saveTimer);
    this.editor.destroy();
  }
};
```

### SortableJS (Drag-and-Drop)

```heex
<div id="sortable-list" phx-hook="Sortable">
  <div :for={{dom_id, item} <- @streams.items} id={dom_id} class="sortable-item">
    {item.name}
  </div>
</div>
```

```javascript
import Sortable from "sortablejs";

Hooks.Sortable = {
  mounted() {
    this.sortable = Sortable.create(this.el, {
      animation: 150,
      onEnd: (evt) => {
        const ids = Array.from(this.el.children).map(el => el.id);
        this.pushEvent("reorder", { ids });
      }
    });
  },

  destroyed() {
    this.sortable.destroy();
  }
};
```

### Lazy Loading Libraries

```javascript
Hooks.HeavyChart = {
  async mounted() {
    // Dynamic import — only loads when hook mounts
    const { Chart } = await import("chart.js/auto");
    this.chart = new Chart(this.el.querySelector("canvas"), {
      /* config */
    });
  },

  destroyed() {
    if (this.chart) this.chart.destroy();
  }
};
```

## Hook ↔ LiveSvelte Boundary

### When Hook vs. LiveSvelte

| Use a Hook | Use LiveSvelte |
|------------|----------------|
| Thin wrapper around a JS library | Complex interactive UI with internal state |
| Simple DOM manipulation | Rich editors, data grids, drag-and-drop with state |
| No reactive template needed | Needs reactive rendering (conditional UI, lists, transitions) |
| Library has imperative API | Component has declarative props/events pattern |

### Don't Duplicate

If LiveSvelte already handles server communication via `live.pushEvent`, don't also create a hook for the same element. LiveSvelte components already have the full hook lifecycle built in.

```svelte
<!-- LiveSvelte already gives you pushEvent — no hook needed -->
<script lang="ts">
  let { data, live } = $props();

  function handleChange(newData) {
    live.pushEvent("data-changed", { data: newData });
  }

  // Listen for server events
  live.handleEvent("refresh", (payload) => {
    // update local state
  });
</script>
```

### Hooks as Thin Bridges

When you need a non-Svelte JS library alongside LiveView (not worth a full Svelte component):

```javascript
// Hook bridges a JS library to LiveView — keeps it simple
Hooks.DatePicker = {
  mounted() {
    this.picker = new Flatpickr(this.el, {
      onChange: (dates) => {
        this.pushEvent("date-selected", { date: dates[0]?.toISOString() });
      }
    });
  },
  destroyed() {
    this.picker.destroy();
  }
};
```

## Common Use Cases with Patterns

### Clipboard (Copy to Clipboard)

No hook needed — use `JS.dispatch`:

```elixir
<button phx-click={JS.dispatch("my_app:clipcopy", to: "#api-key")}>
  Copy API Key
</button>
<code id="api-key">{@api_key}</code>
```

```javascript
// app.js — one-time setup
window.addEventListener("my_app:clipcopy", (e) => {
  if ("clipboard" in navigator) {
    navigator.clipboard.writeText(e.target.textContent);
  }
});
```

### Infinite Scroll

```heex
<div id="infinite-scroll" phx-hook="InfiniteScroll" data-page={@page}>
  <div :for={{dom_id, item} <- @streams.items} id={dom_id}>
    {item.name}
  </div>
  <div id="scroll-sentinel" class="h-1"></div>
</div>
```

```javascript
Hooks.InfiniteScroll = {
  mounted() {
    this._observer = new IntersectionObserver((entries) => {
      entries.forEach(entry => {
        if (entry.isIntersecting) {
          const page = parseInt(this.el.dataset.page);
          this.pushEvent("load-more", { page: page + 1 });
        }
      });
    }, { rootMargin: "200px" });

    const sentinel = this.el.querySelector("#scroll-sentinel");
    if (sentinel) this._observer.observe(sentinel);
  },

  updated() {
    // Re-observe after DOM update if sentinel was replaced
    const sentinel = this.el.querySelector("#scroll-sentinel");
    if (sentinel) this._observer.observe(sentinel);
  },

  destroyed() {
    this._observer.disconnect();
  }
};
```

### Local Storage Persistence

```javascript
Hooks.LocalStorage = {
  mounted() {
    // Restore saved value
    const key = this.el.dataset.key;
    const saved = localStorage.getItem(key);
    if (saved) {
      this.pushEvent("restore-local", { key, value: JSON.parse(saved) });
    }

    // Listen for server-pushed saves
    this.handleEvent(`save-local-${key}`, ({ value }) => {
      localStorage.setItem(key, JSON.stringify(value));
    });
  }
};
```

### Scroll to Element

```javascript
Hooks.ScrollTo = {
  mounted() {
    this.handleEvent("scroll-to", ({ id }) => {
      const el = document.getElementById(id);
      if (el) el.scrollIntoView({ behavior: "smooth", block: "center" });
    });
  }
};
```

### Keyboard Shortcuts

```javascript
Hooks.KeyboardShortcuts = {
  mounted() {
    this._handler = (e) => {
      // Ignore if typing in an input
      if (e.target.matches("input, textarea, select, [contenteditable]")) return;

      if (e.key === "/" && !e.ctrlKey) {
        e.preventDefault();
        this.pushEvent("focus-search", {});
      }
      if (e.key === "Escape") {
        this.pushEvent("close-modal", {});
      }
      if (e.ctrlKey && e.key === "k") {
        e.preventDefault();
        this.pushEvent("open-command-palette", {});
      }
    };
    window.addEventListener("keydown", this._handler);
  },

  destroyed() {
    window.removeEventListener("keydown", this._handler);
  }
};
```

### Viewport / Resize Observer

```javascript
Hooks.ViewportTracker = {
  mounted() {
    this._observer = new ResizeObserver((entries) => {
      for (const entry of entries) {
        this.pushEvent("viewport-resized", {
          width: entry.contentRect.width,
          height: entry.contentRect.height
        });
      }
    });
    this._observer.observe(this.el);
  },

  destroyed() {
    this._observer.disconnect();
  }
};
```

## Anti-Patterns and Pitfalls

### Using hooks for things JS commands already do

```heex
<!-- ❌ Bad: hook just to toggle a class -->
<div phx-hook="ToggleClass">...</div>

<!-- ✅ Good: JS command -->
<button phx-click={JS.toggle_class("active", to: "#sidebar")}>Toggle</button>
```

### Forgetting id on hooked elements

```heex
<!-- ❌ Silent failure — hook never mounts, no error -->
<div phx-hook="MyHook">...</div>

<!-- ✅ Always include a unique id -->
<div id="my-element" phx-hook="MyHook">...</div>
```

### Not cleaning up in destroyed

```javascript
// ❌ Memory leak — listener persists after element removed
Hooks.Bad = {
  mounted() {
    window.addEventListener("resize", this.onResize);
  }
  // Missing destroyed()!
};

// ✅ Always clean up
Hooks.Good = {
  mounted() {
    this._onResize = () => { /* ... */ };
    window.addEventListener("resize", this._onResize);
  },
  destroyed() {
    window.removeEventListener("resize", this._onResize);
  }
};
```

### Mutating DOM without phx-update="ignore"

```javascript
// ❌ This innerHTML will be clobbered on next server patch
Hooks.Bad = {
  mounted() {
    this.el.innerHTML = "<p>Custom content</p>";
  }
};

// ✅ Use phx-update="ignore" on the container, or use JS commands
```

### Overusing phx-update="ignore"

```heex
<!-- ❌ Now the server can never update this content -->
<div phx-update="ignore">
  <h1>{@title}</h1>  <!-- This will never update! -->
  <div id="chart"></div>
</div>

<!-- ✅ Only ignore the library's container -->
<h1>{@title}</h1>
<div id="chart-container" phx-hook="Chart" phx-update="ignore">
  <canvas></canvas>
</div>
```

### Putting business logic in hooks

```javascript
// ❌ Bad: pricing logic in JS
Hooks.PriceCalculator = {
  mounted() {
    this.el.addEventListener("input", (e) => {
      const qty = parseInt(e.target.value);
      const price = qty > 10 ? qty * 0.9 : qty * 1.0; // Business logic!
      this.el.querySelector(".total").textContent = `$${price}`;
    });
  }
};

// ✅ Good: push to server, let server calculate
Hooks.PriceCalculator = {
  mounted() {
    this.el.addEventListener("input", (e) => {
      this.pushEvent("calculate-price", { quantity: e.target.value });
    });
  }
};
```

### Race conditions between pushEvent and DOM patches

```javascript
// ❌ Risky: reading DOM right after push (patch may arrive between)
this.pushEvent("save", data);
const value = this.el.querySelector(".result").textContent; // May be stale

// ✅ Use reply callback to get authoritative data
this.pushEvent("save", data, (reply) => {
  // reply contains the server's response
  this.el.querySelector(".result").textContent = reply.message;
});
```

### Hooking stream elements without stable IDs

```heex
<!-- ❌ Bad: id changes on re-render, hook remounts every time -->
<div :for={item <- @items} id={"item-#{:rand.uniform(1000)}"} phx-hook="ItemHook">

<!-- ✅ Good: use stream dom_id (stable) -->
<div :for={{dom_id, item} <- @streams.items} id={dom_id} phx-hook="ItemHook">
```

## Testing

### Testing Hook Events with LiveViewTest

```elixir
# Simulate a hook pushing an event
test "handles hook push event", %{conn: conn} do
  {:ok, view, _html} = live(conn, "/my-page")

  # Simulate pushEvent from a hook
  render_hook(view, "reorder", %{"ids" => ["item-1", "item-3", "item-2"]})

  # Assert the reorder was applied
  assert render(view) =~ "item-1"
end

# Target a specific element
render_hook(element(view, "#sortable-list"), "reorder", %{"ids" => ids})
```

### Testing Server → Client Events

```elixir
test "pushes chart data to client", %{conn: conn} do
  {:ok, view, _html} = live(conn, "/dashboard")

  # Trigger server action that pushes event
  send(view.pid, {:new_data, [1, 2, 3]})

  # Assert the event was pushed
  assert_push_event(view, "chart-update", %{points: [1, 2, 3]})
end
```

### Browser Testing with Wallaby

For full integration testing of JS behavior:

```elixir
test "copy button copies to clipboard", %{session: session} do
  session
  |> visit("/api-keys")
  |> click(button("Copy API Key"))
  # Clipboard testing requires browser-specific approaches
end
```

### Debugging in Browser

```javascript
// Enable LiveView debug logging
liveSocket.enableDebug();

// Inspect registered hooks
console.log(liveSocket.hooks);

// Enable latency simulation
liveSocket.enableLatencySim(1000); // 1 second delay
```

## File Organization

```
assets/
├── js/
│   ├── app.js                  # LiveSocket setup, hook registration
│   └── hooks/
│       ├── index.js            # Re-exports all hooks
│       ├── infinite_scroll.js  # One file per hook
│       ├── chart.js
│       └── sortable.js
```

```javascript
// assets/js/hooks/index.js
import InfiniteScroll from "./infinite_scroll";
import Chart from "./chart";
import Sortable from "./sortable";

export default { InfiniteScroll, Chart, Sortable };
```

```javascript
// assets/js/app.js
import { hooks as colocatedHooks } from "phoenix-colocated/my_app";
import manualHooks from "./hooks";

let liveSocket = new LiveSocket("/live", Socket, {
  hooks: { ...colocatedHooks, ...manualHooks },
  params: { _csrf_token: csrfToken }
});
```

Colocated hooks live inline in their component files — no separate JS file needed.

## Related Skills

- **elixir-liveview**: LiveView lifecycle, streams, events, PubSub
- **svelte-core**: LiveSvelte component patterns, `live.pushEvent`, SSR
- **frontend-typescript**: TypeScript for hook type safety
- **frontend-tailwind**: Transition classes for JS commands
