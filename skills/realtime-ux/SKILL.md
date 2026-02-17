---
name: realtime-ux
description: UX patterns for real-time interfaces built with Phoenix LiveView. Covers optimistic UI, presence indicators, conflict resolution for concurrent editing, graceful disconnect handling, loading/skeleton states, and real-time feedback. Use when building collaborative features, real-time dashboards, or any LiveView interface where multiple users interact simultaneously.
---

# Real-Time UX Patterns

Expert guidance for the UX challenges that LiveView makes technically easy but experientially hard — optimistic updates, presence, conflict resolution, disconnect handling, and loading states.

## Core Principles

- **Server owns the truth** — optimistic UI is a perception layer, not a state layer. The server always has the last word.
- **Acknowledge every action** — users need immediate feedback, even if the server hasn't responded yet.
- **Degrade gracefully** — disconnects happen. The UI should communicate state honestly, not freeze silently.
- **Prevent data loss** — never let a user lose work due to a race condition, disconnect, or conflict.
- **Show, don't block** — prefer non-blocking indicators (badges, borders, toasts) over modal interruptions.

## Optimistic UI

### The Problem

LiveView round-trips take 20-100ms. For most actions this is imperceptible. But for some interactions — toggling, liking, drag-and-drop reordering — even 50ms of latency feels sluggish.

### Strategy 1: CSS Loading States (Zero JS)

LiveView automatically adds loading classes during round-trips. Use them for instant visual feedback.

```heex
<button
  phx-click="toggle_favorite"
  phx-value-id={@item.id}
  class="transition-opacity phx-click-loading:opacity-50"
>
  <.icon name={if @item.favorited, do: "hero-heart-solid", else: "hero-heart"} />
</button>
```

Available loading classes:
- `phx-click-loading` — on the clicked element
- `phx-submit-loading` — on the submitted form
- `phx-change-loading` — on the changed form
- `phx-loading` — on any element with a `phx-loading` target

### Strategy 2: JS Commands for Instant Feedback

Use `JS.push` combined with immediate visual changes:

```elixir
def toggle_favorite_button(assigns) do
  ~H"""
  <button phx-click={
    JS.push("toggle_favorite", value: %{id: @item.id})
    |> JS.toggle_class("text-red-500 text-gray-400", to: "#fav-icon-#{@item.id}")
  }>
    <.icon id={"fav-icon-#{@item.id}"}
           name="hero-heart-solid"
           class={if @item.favorited, do: "text-red-500", else: "text-gray-400"} />
  </button>
  """
end
```

The class toggles instantly. If the server disagrees, the next patch corrects it.

### Strategy 3: Optimistic Assigns with Rollback

For complex optimistic updates, apply the change immediately and let the server confirm or correct:

```elixir
def handle_event("toggle_favorite", %{"id" => id}, socket) do
  item = get_item(socket.assigns.items, id)

  # Optimistically update the UI immediately
  socket = update(socket, :items, fn items ->
    update_item(items, id, &Map.update!(&1, :favorited, fn v -> !v end))
  end)

  # Then do the actual work (may fail)
  case Items.toggle_favorite(item, socket.assigns.current_user) do
    {:ok, updated_item} ->
      # Server confirms — assign the real value (may be same as optimistic)
      {:noreply, update(socket, :items, &update_item(&1, id, fn _ -> updated_item end))}

    {:error, _reason} ->
      # Server rejects — revert to original
      {:noreply, update(socket, :items, &update_item(&1, id, fn _ -> item end))}
  end
end
```

### Strategy 4: Optimistic Streams

For stream-based lists, insert optimistically and correct on confirmation:

```elixir
def handle_event("add_comment", %{"body" => body}, socket) do
  # Create a temporary optimistic item
  temp_id = "temp-#{System.unique_integer([:positive])}"
  optimistic_comment = %{
    id: temp_id,
    body: body,
    user: socket.assigns.current_user,
    inserted_at: DateTime.utc_now(),
    pending: true
  }

  # Insert immediately
  socket = stream_insert(socket, :comments, optimistic_comment, at: 0)

  # Create for real
  case Comments.create(body, socket.assigns.current_user) do
    {:ok, real_comment} ->
      {:noreply,
       socket
       |> stream_delete(:comments, optimistic_comment)
       |> stream_insert(:comments, real_comment, at: 0)}

    {:error, _changeset} ->
      {:noreply,
       socket
       |> stream_delete(:comments, optimistic_comment)
       |> put_flash(:error, "Failed to post comment")}
  end
end
```

```heex
<div :for={{dom_id, comment} <- @streams.comments} id={dom_id}
     class={if comment.pending, do: "opacity-60", else: ""}>
  <p>{comment.body}</p>
  <span :if={comment.pending} class="text-xs text-text-secondary">Sending...</span>
</div>
```

### When NOT to Use Optimistic UI

- **Destructive actions** (delete, cancel subscription) — confirm first, then execute
- **Actions with side effects** (send email, charge payment) — show loading, wait for confirmation
- **Validation-dependent actions** (submit form with server-side validation) — wait for server
- **Actions where rollback is confusing** (reordering a list that other users see) — show loading instead

## Presence Indicators

### Setting Up Phoenix Presence in LiveView

```elixir
# lib/my_app_web/presence.ex
defmodule MyAppWeb.Presence do
  use Phoenix.Presence,
    otp_app: :my_app,
    pubsub_server: MyApp.PubSub
end
```

```elixir
# In your LiveView
def mount(_params, _session, socket) do
  if connected?(socket) do
    topic = "document:#{socket.assigns.document_id}"

    Phoenix.PubSub.subscribe(MyApp.PubSub, topic)

    {:ok, _} = MyAppWeb.Presence.track(self(), topic, socket.assigns.current_user.id, %{
      name: socket.assigns.current_user.name,
      avatar_url: socket.assigns.current_user.avatar_url,
      joined_at: DateTime.utc_now(),
      status: "active"
    })

    presences = MyAppWeb.Presence.list(topic)
    socket = assign(socket, :presences, presences)
  end

  {:ok, socket}
end

def handle_info(%Phoenix.Socket.Broadcast{event: "presence_diff", payload: diff}, socket) do
  presences =
    socket.assigns.presences
    |> MyAppWeb.Presence.sync_diff(diff)

  {:noreply, assign(socket, :presences, presences)}
end
```

### Rendering Presence

```elixir
def presence_avatars(assigns) do
  users =
    assigns.presences
    |> Enum.flat_map(fn {_user_id, %{metas: metas}} -> metas end)
    |> Enum.uniq_by(& &1.name)

  assigns = assign(assigns, :users, users)

  ~H"""
  <div class="flex -space-x-2">
    <div
      :for={user <- @users}
      class="relative"
      title={user.name}
    >
      <img
        src={user.avatar_url}
        alt={user.name}
        class={[
          "h-8 w-8 rounded-full ring-2 ring-white",
          user.status == "idle" && "opacity-50"
        ]}
      />
      <span class={[
        "absolute bottom-0 right-0 h-2.5 w-2.5 rounded-full ring-2 ring-white",
        status_color(user.status)
      ]} />
    </div>
    <div :if={length(@users) > 5}
         class="flex h-8 w-8 items-center justify-center rounded-full bg-gray-100 text-xs font-medium ring-2 ring-white">
      +{length(@users) - 5}
    </div>
  </div>
  """
end

defp status_color("active"), do: "bg-green-500"
defp status_color("idle"), do: "bg-yellow-500"
defp status_color(_), do: "bg-gray-400"
```

### Idle Detection

Track user activity and update presence status:

```javascript
Hooks.ActivityTracker = {
  mounted() {
    this._idleTimeout = null;
    this._isIdle = false;

    const resetIdle = () => {
      if (this._isIdle) {
        this._isIdle = false;
        this.pushEvent("user_active", {});
      }
      clearTimeout(this._idleTimeout);
      this._idleTimeout = setTimeout(() => {
        this._isIdle = true;
        this.pushEvent("user_idle", {});
      }, 60000); // 1 minute
    };

    this._events = ["mousemove", "keydown", "scroll", "touchstart"];
    this._events.forEach(e => window.addEventListener(e, resetIdle, { passive: true }));
    resetIdle();
  },

  destroyed() {
    clearTimeout(this._idleTimeout);
    this._events.forEach(e => window.removeEventListener(e, this._resetIdle));
  }
};
```

```elixir
def handle_event("user_idle", _params, socket) do
  topic = "document:#{socket.assigns.document_id}"
  MyAppWeb.Presence.update(self(), topic, socket.assigns.current_user.id, fn meta ->
    Map.put(meta, :status, "idle")
  end)
  {:noreply, socket}
end

def handle_event("user_active", _params, socket) do
  topic = "document:#{socket.assigns.document_id}"
  MyAppWeb.Presence.update(self(), topic, socket.assigns.current_user.id, fn meta ->
    Map.put(meta, :status, "active")
  end)
  {:noreply, socket}
end
```

### Cursor/Selection Presence

For collaborative editing — show where other users are:

```elixir
def handle_event("cursor_moved", %{"line" => line, "col" => col}, socket) do
  topic = "document:#{socket.assigns.document_id}"
  MyAppWeb.Presence.update(self(), topic, socket.assigns.current_user.id, fn meta ->
    Map.merge(meta, %{cursor_line: line, cursor_col: col, last_active: DateTime.utc_now()})
  end)
  {:noreply, socket}
end
```

## Conflict Resolution

### The Problem

Two users edit the same resource simultaneously. Without conflict handling, the last save wins and the first user's changes are silently lost.

### Strategy 1: Optimistic Locking (Last Write Wins with Detection)

```elixir
# Schema has a :lock_version field (integer, default 0)
def update_document(document, attrs, expected_version) do
  if document.lock_version != expected_version do
    {:error, :stale}
  else
    document
    |> Document.changeset(attrs)
    |> Ecto.Changeset.optimistic_lock(:lock_version)
    |> Repo.update()
  end
end
```

```elixir
# In LiveView
def handle_event("save", %{"document" => params}, socket) do
  case Documents.update_document(
    socket.assigns.document,
    params,
    socket.assigns.document.lock_version
  ) do
    {:ok, document} ->
      {:noreply, assign(socket, :document, document)}

    {:error, :stale} ->
      {:noreply,
       socket
       |> put_flash(:error, "This document was updated by someone else. Your changes have been preserved below.")
       |> assign(:conflict, %{
         yours: params,
         theirs: Documents.get_document!(socket.assigns.document.id)
       })}

    {:error, changeset} ->
      {:noreply, assign(socket, :changeset, changeset)}
  end
end
```

### Strategy 2: Field-Level Merging

For forms with independent fields, merge non-conflicting changes:

```elixir
def merge_changes(server_doc, user_changes, original_doc) do
  Enum.reduce(user_changes, server_doc, fn {field, new_value}, acc ->
    original_value = Map.get(original_doc, field)
    server_value = Map.get(server_doc, field)

    cond do
      # User didn't change this field — keep server value
      new_value == original_value -> acc
      # Server didn't change this field — take user's change
      server_value == original_value -> Map.put(acc, field, new_value)
      # Both changed — conflict
      true -> Map.put(acc, field, {:conflict, server_value, new_value})
    end
  end)
end
```

### Strategy 3: Real-Time Broadcasting (Prevent Conflicts)

Broadcast changes as they happen so users see each other's edits:

```elixir
def handle_event("field_changed", %{"field" => field, "value" => value}, socket) do
  # Save the change
  {:ok, document} = Documents.update_field(socket.assigns.document, field, value)

  # Broadcast to other users viewing this document
  Phoenix.PubSub.broadcast(
    MyApp.PubSub,
    "document:#{document.id}",
    {:field_updated, field, value, socket.assigns.current_user.id}
  )

  {:noreply, assign(socket, :document, document)}
end

def handle_info({:field_updated, field, value, updater_id}, socket) do
  if updater_id != socket.assigns.current_user.id do
    document = Map.put(socket.assigns.document, String.to_existing_atom(field), value)
    {:noreply,
     socket
     |> assign(:document, document)
     |> push_event("field-updated-by-other", %{field: field, user: updater_id})}
  else
    {:noreply, socket}
  end
end
```

### Strategy 4: Locking (Pessimistic)

For critical sections where concurrent editing must be prevented:

```elixir
def handle_event("start_editing", %{"section" => section}, socket) do
  case Documents.acquire_lock(socket.assigns.document, section, socket.assigns.current_user) do
    {:ok, lock} ->
      Phoenix.PubSub.broadcast(
        MyApp.PubSub,
        "document:#{socket.assigns.document.id}",
        {:section_locked, section, socket.assigns.current_user}
      )
      {:noreply, assign(socket, :active_lock, lock)}

    {:error, :locked_by, other_user} ->
      {:noreply, put_flash(socket, :info, "#{other_user.name} is editing this section")}
  end
end
```

```heex
<div class={[
  "p-4 rounded border-2 transition-colors",
  @locked_by && @locked_by.id != @current_user.id && "border-yellow-400 bg-yellow-50",
  @locked_by && @locked_by.id == @current_user.id && "border-brand-500",
  !@locked_by && "border-transparent hover:border-gray-200"
]}>
  <div :if={@locked_by && @locked_by.id != @current_user.id}
       class="text-xs text-yellow-700 mb-2 flex items-center gap-1">
    <span class="h-2 w-2 rounded-full bg-yellow-500 animate-pulse" />
    {@locked_by.name} is editing
  </div>
  ...
</div>
```

### Choosing a Strategy

| Strategy | Complexity | UX | Best For |
|----------|-----------|-----|----------|
| Optimistic locking | Low | Good (detect + resolve) | Forms, settings, CMS |
| Field-level merge | Medium | Great (auto-merge) | Multi-field forms |
| Real-time broadcast | Medium | Great (see changes live) | Collaborative docs |
| Pessimistic locking | Low | Acceptable (turn-based) | Critical data, billing |

## Graceful Disconnect Handling

### The Disconnect Experience

When a LiveView WebSocket disconnects, LiveView automatically:
1. Adds `phx-loading` class to the root element
2. Attempts reconnection with exponential backoff
3. Restores state on reconnection

### Visual Disconnect Indicator

```heex
<div id="connection-status"
     class="fixed top-0 inset-x-0 z-50 transition-transform duration-300"
     phx-disconnected={JS.show(to: "#connection-status", transition: {"ease-out duration-300", "-translate-y-full", "translate-y-0"})}
     phx-connected={JS.hide(to: "#connection-status", transition: {"ease-in duration-200", "translate-y-0", "-translate-y-full"})}>
  <div class="bg-yellow-500 text-yellow-900 text-center text-sm py-2 px-4">
    <span class="inline-flex items-center gap-2">
      <svg class="h-4 w-4 animate-spin" viewBox="0 0 24 24">...</svg>
      Connection lost. Reconnecting...
    </span>
  </div>
</div>
```

### Protecting Unsaved Work

```javascript
Hooks.UnsavedChanges = {
  mounted() {
    this._hasChanges = false;

    this.handleEvent("changes_saved", () => {
      this._hasChanges = false;
    });

    this.el.addEventListener("input", () => {
      this._hasChanges = true;
    });

    this._beforeUnload = (e) => {
      if (this._hasChanges) {
        e.preventDefault();
        e.returnValue = "";
      }
    };
    window.addEventListener("beforeunload", this._beforeUnload);
  },

  disconnected() {
    if (this._hasChanges) {
      // Save to localStorage as backup
      const formData = new FormData(this.el);
      const data = Object.fromEntries(formData);
      localStorage.setItem(`unsaved-${this.el.id}`, JSON.stringify(data));
    }
  },

  reconnected() {
    const saved = localStorage.getItem(`unsaved-${this.el.id}`);
    if (saved) {
      this.pushEvent("restore_unsaved", JSON.parse(saved));
      localStorage.removeItem(`unsaved-${this.el.id}`);
    }
  },

  destroyed() {
    window.removeEventListener("beforeunload", this._beforeUnload);
  }
};
```

### Reconnection State Refresh

```elixir
def mount(_params, _session, socket) do
  if connected?(socket) do
    # Subscribe to real-time updates
    Phoenix.PubSub.subscribe(MyApp.PubSub, "document:#{socket.assigns.document_id}")
  end

  # Always load fresh data on mount (handles reconnection)
  document = Documents.get_document!(socket.assigns.document_id)
  {:ok, assign(socket, :document, document)}
end
```

### Offline-Capable Actions

For critical actions that should survive disconnects:

```javascript
Hooks.ResilientAction = {
  mounted() {
    this.el.addEventListener("click", (e) => {
      const action = this.el.dataset.action;
      const payload = JSON.parse(this.el.dataset.payload || "{}");

      // Try to push immediately
      this.pushEvent(action, payload, (reply) => {
        // Success — remove from queue
        this._removeFromQueue(action, payload);
      });

      // Also queue in case of disconnect
      this._addToQueue(action, payload);
    });
  },

  reconnected() {
    // Replay queued actions
    const queue = JSON.parse(localStorage.getItem("action-queue") || "[]");
    queue.forEach(({ action, payload }) => {
      this.pushEvent(action, payload, () => {
        this._removeFromQueue(action, payload);
      });
    });
  },

  _addToQueue(action, payload) {
    const queue = JSON.parse(localStorage.getItem("action-queue") || "[]");
    queue.push({ action, payload, timestamp: Date.now() });
    localStorage.setItem("action-queue", JSON.stringify(queue));
  },

  _removeFromQueue(action, payload) {
    const queue = JSON.parse(localStorage.getItem("action-queue") || "[]");
    const filtered = queue.filter(q => !(q.action === action && JSON.stringify(q.payload) === JSON.stringify(payload)));
    localStorage.setItem("action-queue", JSON.stringify(filtered));
  }
};
```

## Loading and Skeleton States

### Skeleton Screens

Show the shape of content before data arrives:

```elixir
def skeleton_card(assigns) do
  ~H"""
  <div class="animate-pulse rounded-lg border border-border p-4">
    <div class="h-4 w-3/4 rounded bg-gray-200 mb-3" />
    <div class="h-3 w-full rounded bg-gray-200 mb-2" />
    <div class="h-3 w-5/6 rounded bg-gray-200 mb-2" />
    <div class="h-3 w-2/3 rounded bg-gray-200" />
  </div>
  """
end

def skeleton_table_row(assigns) do
  ~H"""
  <tr class="animate-pulse">
    <td :for={_ <- 1..@columns} class="px-3 py-3">
      <div class="h-3 rounded bg-gray-200" />
    </td>
  </tr>
  """
end
```

```heex
<div :if={@loading}>
  <.skeleton_card :for={_ <- 1..3} />
</div>
<div :if={!@loading}>
  <.card :for={item <- @items} item={item} />
</div>
```

### Async Assigns with Loading States

```elixir
def mount(_params, _session, socket) do
  {:ok,
   socket
   |> assign(:page_title, "Dashboard")
   |> assign_async(:stats, fn -> {:ok, %{stats: Analytics.get_stats()}} end)
   |> assign_async(:recent_activity, fn -> {:ok, %{recent_activity: Activity.recent()}} end)}
end
```

```heex
<.async_result :let={stats} assign={@stats}>
  <:loading>
    <div class="grid grid-cols-3 gap-4">
      <div :for={_ <- 1..3} class="animate-pulse h-24 rounded-lg bg-gray-200" />
    </div>
  </:loading>
  <:failed :let={_reason}>
    <div class="text-red-600 p-4 rounded-lg bg-red-50">
      Failed to load stats.
      <button phx-click="retry_stats" class="underline">Retry</button>
    </div>
  </:failed>
  <.stats_grid stats={stats} />
</.async_result>
```

### Inline Loading Indicators

For actions that take time but don't need a full skeleton:

```heex
<button phx-click="export_csv" class="inline-flex items-center gap-2">
  <svg class="hidden phx-click-loading:block h-4 w-4 animate-spin" viewBox="0 0 24 24">
    <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4" fill="none" />
    <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4z" />
  </svg>
  <span class="phx-click-loading:hidden">Export CSV</span>
  <span class="hidden phx-click-loading:inline">Exporting...</span>
</button>
```

### Progress Indicators for Long Operations

```elixir
def handle_event("start_import", %{"file" => file}, socket) do
  # Start async work that reports progress
  task = Task.async(fn ->
    Importer.import_file(file, fn progress ->
      send(socket.root_pid, {:import_progress, progress})
    end)
  end)

  {:noreply, assign(socket, import_progress: 0, import_task: task)}
end

def handle_info({:import_progress, progress}, socket) do
  {:noreply, assign(socket, :import_progress, progress)}
end
```

```heex
<div :if={@import_progress > 0 && @import_progress < 100}
     class="w-full bg-gray-200 rounded-full h-2 overflow-hidden">
  <div class="bg-brand-500 h-full rounded-full transition-all duration-300"
       style={"width: #{@import_progress}%"} />
</div>
```

## Real-Time Feedback Patterns

### Toast Notifications for Background Events

```elixir
def handle_info({:comment_added, comment}, socket) do
  if comment.user_id != socket.assigns.current_user.id do
    {:noreply,
     socket
     |> stream_insert(:comments, comment)
     |> put_flash(:info, "#{comment.user.name} added a comment")}
  else
    {:noreply, socket}
  end
end
```

### Live Counters and Badges

```elixir
def handle_info({:new_notification, _notification}, socket) do
  {:noreply, update(socket, :unread_count, &(&1 + 1))}
end
```

```heex
<div class="relative">
  <.icon name="hero-bell" />
  <span :if={@unread_count > 0}
        class="absolute -top-1 -right-1 flex h-4 w-4 items-center justify-center rounded-full bg-red-500 text-[10px] font-bold text-white">
    {if @unread_count > 9, do: "9+", else: @unread_count}
  </span>
</div>
```

### Typing Indicators

```elixir
def handle_event("typing", _params, socket) do
  topic = "chat:#{socket.assigns.room_id}"
  Phoenix.PubSub.broadcast(MyApp.PubSub, topic, {:typing, socket.assigns.current_user})

  # Auto-clear after 3 seconds
  if socket.assigns[:typing_timer] do
    Process.cancel_timer(socket.assigns.typing_timer)
  end
  timer = Process.send_after(self(), {:clear_typing, socket.assigns.current_user.id}, 3000)

  {:noreply, assign(socket, :typing_timer, timer)}
end

def handle_info({:typing, user}, socket) do
  if user.id != socket.assigns.current_user.id do
    typers = Map.put(socket.assigns.typers, user.id, user.name)
    {:noreply, assign(socket, :typers, typers)}
  else
    {:noreply, socket}
  end
end

def handle_info({:clear_typing, user_id}, socket) do
  typers = Map.delete(socket.assigns.typers, user_id)
  {:noreply, assign(socket, :typers, typers)}
end
```

```heex
<div :if={map_size(@typers) > 0} class="text-xs text-text-secondary flex items-center gap-1 h-5">
  <span class="flex gap-0.5">
    <span class="h-1.5 w-1.5 rounded-full bg-text-secondary animate-bounce [animation-delay:0ms]" />
    <span class="h-1.5 w-1.5 rounded-full bg-text-secondary animate-bounce [animation-delay:150ms]" />
    <span class="h-1.5 w-1.5 rounded-full bg-text-secondary animate-bounce [animation-delay:300ms]" />
  </span>
  {typing_text(@typers)}
</div>
```

```elixir
defp typing_text(typers) do
  names = Map.values(typers)
  case names do
    [name] -> "#{name} is typing"
    [a, b] -> "#{a} and #{b} are typing"
    [a | _rest] -> "#{a} and #{length(names) - 1} others are typing"
    [] -> ""
  end
end
```

### Live Data Updates with Highlights

```elixir
def handle_info({:price_updated, symbol, price}, socket) do
  {:noreply,
   socket
   |> stream_insert(:prices, %{id: symbol, symbol: symbol, price: price, updated: true})
   |> push_event("highlight", %{id: "price-#{symbol}"})}
end
```

```heex
<tr :for={{dom_id, price} <- @streams.prices} id={dom_id}
    data-highlight={JS.transition("bg-yellow-100", to: "##{dom_id}")}>
  <td>{price.symbol}</td>
  <td class="tabular-nums">{price.price}</td>
</tr>
```

## Anti-Patterns

### Silent failures on disconnect

```
❌ User submits form → disconnect → nothing happens → user confused
✅ Show connection status banner + queue action for replay on reconnect
```

### Optimistic UI for irreversible actions

```
❌ Optimistically showing "Email sent!" before server confirms
✅ Show spinner → wait for server → then confirm
```

### Ignoring concurrent edits

```
❌ Two users edit same field → last save silently wins → first user's work lost
✅ Use optimistic locking + conflict UI, or broadcast changes in real-time
```

### Full-page loading spinners

```
❌ Blank page with centered spinner while data loads
✅ Skeleton screens that match the shape of real content
   Progressive loading: show what you have, load the rest async
```

### Blocking UI during background operations

```
❌ Modal: "Please wait while we process your request..."
✅ Non-blocking toast: "Processing..." → "Done!" with the UI still interactive
```

### Not debouncing presence updates

```elixir
# ❌ Broadcasting cursor position on every mousemove (60fps = 60 broadcasts/sec)
def handle_event("cursor_moved", coords, socket) do
  Phoenix.PubSub.broadcast(...)
end

# ✅ Throttle to reasonable frequency
def handle_event("cursor_moved", coords, socket) do
  now = System.monotonic_time(:millisecond)
  if now - (socket.assigns[:last_cursor_broadcast] || 0) > 100 do  # Max 10/sec
    Phoenix.PubSub.broadcast(...)
    {:noreply, assign(socket, :last_cursor_broadcast, now)}
  else
    {:noreply, socket}
  end
end
```

## Related Skills

- **elixir-liveview**: LiveView lifecycle, streams, PubSub, async assigns
- **elixir-otp**: GenServer, PubSub, supervision for presence and background tasks
- **liveview-js-interop**: Hooks for disconnect handling, JS commands for loading states
- **animation-transitions**: Transition patterns for loading states and presence indicators
- **svelte-core**: LiveSvelte for complex real-time interactive components
