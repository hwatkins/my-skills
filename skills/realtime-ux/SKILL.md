---
name: realtime-ux
description: >
  UX patterns for building real-time interfaces with Phoenix LiveView — the human
  side of real-time that's harder than the technical side. Use this skill whenever
  building features that involve: optimistic UI updates, presence indicators (who's
  online, typing indicators, live cursors), conflict resolution when multiple users
  edit the same data, graceful degradation on WebSocket disconnect, loading/skeleton
  states, real-time notifications, collaborative editing, or any LiveView feature where
  latency, concurrent users, or connection reliability affects user experience. Also
  trigger when the user mentions "optimistic UI", "presence", "typing indicator",
  "disconnect handling", "skeleton state", "loading state", "concurrent editing",
  "real-time UX", or asks how to make a LiveView feature "feel instant" or "handle
  offline gracefully".
---

# Real-Time UX Patterns

LiveView makes real-time technically easy. The hard part is UX: making interactions
feel instant, handling concurrent users gracefully, and degrading smoothly when
connections fail. This skill covers the UX patterns, not the plumbing.

## The Five UX Challenges of Real-Time

1. **Latency** — The round trip to the server is perceptible. How do you make it feel instant?
2. **Presence** — Multiple users are here. How do you show who, where, and what they're doing?
3. **Conflicts** — Two users changed the same thing. Who wins? How does the loser know?
4. **Disconnection** — The WebSocket dropped. What does the user see? What happens to their work?
5. **Loading** — Data isn't here yet. What does the user see while waiting?

## Core Principles

- **Server owns the truth** — optimistic UI is a perception layer, not a state layer. The server always has the last word.
- **Acknowledge every action** — users need immediate feedback, even if the server hasn't responded yet.
- **Degrade gracefully** — disconnects happen. The UI should communicate state honestly, not freeze silently.
- **Prevent data loss** — never let a user lose work due to a race condition, disconnect, or conflict.
- **Show, don't block** — prefer non-blocking indicators (badges, borders, toasts) over modal interruptions.

## Optimistic UI: Making It Feel Instant

The core tension: LiveView is server-rendered, but users expect client-side speed.

### Level 0: CSS Loading Feedback (Built-in)

LiveView automatically adds `phx-click-loading`, `phx-submit-loading`, and
`phx-change-loading` classes while waiting for server response. Use these for
instant visual feedback — this is the minimum for every interactive element:

```css
.phx-click-loading { opacity: 0.6; pointer-events: none; }
.phx-submit-loading .submit-label { display: none; }
.phx-submit-loading .submit-spinner { display: inline-flex; }
```

```heex
<button phx-click="save" phx-disable-with="Saving...">
  Save
</button>

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

### Level 1: JS Commands for Instant State Changes

Use `Phoenix.LiveView.JS` to apply visual changes immediately on the client,
before the server round trip:

```heex
<button
  phx-click={
    JS.push("toggle_favorite", value: %{id: @item.id})
    |> JS.toggle_class("text-yellow-500", to: "#star-#{@item.id}")
  }
>
  <.icon name="hero-star" id={"star-#{@item.id}"} class={if @item.favorited, do: "text-yellow-500"} />
</button>
```

The star turns yellow immediately. When the server responds, the DOM is patched
to the correct state. If the server rejects it, the class is corrected on the
next patch.

### Level 2: Dual-State Optimistic Pattern

For actions where rollback matters (deleting items, toggling important state),
track both "optimistic" and "confirmed" state:

```elixir
def handle_event("delete_item", %{"id" => id}, socket) do
  # Immediately mark as "deleting" (optimistic)
  send(self(), {:confirm_delete, id})

  {:noreply,
    socket
    |> update(:items, fn items ->
      Enum.map(items, fn
        %{id: ^id} = item -> Map.put(item, :deleting, true)
        item -> item
      end)
    end)
  }
end

def handle_info({:confirm_delete, id}, socket) do
  case Items.delete(id) do
    :ok ->
      {:noreply, update(socket, :items, &Enum.reject(&1, fn i -> i.id == id end))}
    {:error, _} ->
      # Rollback: unmark the item
      {:noreply,
        socket
        |> update(:items, fn items ->
          Enum.map(items, &Map.delete(&1, :deleting))
        end)
        |> put_flash(:error, "Could not delete item")
      }
  end
end
```

In the template, items with `:deleting` true get a fade-out animation:

```heex
<div
  :for={item <- @items}
  class={["transition-opacity duration-300", item[:deleting] && "opacity-30 pointer-events-none"]}
>
  {item.name}
</div>
```

### Level 3: Optimistic Assigns with Rollback

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

### Level 4: Optimistic Streams

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

### Level 5: Undo Buffer (Deferred Execution)

Instead of immediately executing destructive actions, delay execution and offer an undo window:

```elixir
def handle_event("delete_item", %{"id" => id}, socket) do
  # Hide the item immediately, but don't delete yet
  timer = Process.send_after(self(), {:confirm_delete, id}, 5_000)

  {:noreply,
    socket
    |> update(:items, fn items ->
      Enum.map(items, fn
        %{id: ^id} = item -> Map.put(item, :pending_delete, true)
        item -> item
      end)
    end)
    |> assign(undo_timer: timer, undo_item_id: id)
  }
end

def handle_event("undo_delete", _params, socket) do
  Process.cancel_timer(socket.assigns.undo_timer)

  {:noreply,
    socket
    |> update(:items, fn items ->
      Enum.map(items, &Map.delete(&1, :pending_delete))
    end)
    |> assign(undo_timer: nil, undo_item_id: nil)
  }
end

def handle_info({:confirm_delete, id}, socket) do
  Items.delete!(id)
  {:noreply,
    socket
    |> update(:items, &Enum.reject(&1, fn i -> i.id == id end))
    |> assign(undo_timer: nil, undo_item_id: nil)
  }
end
```

```heex
<div :if={@undo_item_id}
  class="fixed bottom-md left-1/2 -translate-x-1/2 z-50
         bg-foreground text-background rounded-lg px-lg py-sm
         shadow-xl flex items-center gap-md">
  <span>Item deleted</span>
  <button phx-click="undo_delete" class="font-medium underline">Undo</button>
</div>
```

### When to Optimize (Latency Thresholds)

Users perceive delays differently by interaction type:
- **Clicks/toggles**: Tolerate ~100ms before feeling sluggish
- **Typing/dragging**: Tolerate ~50ms before feeling laggy
- **Navigation**: Tolerate ~200ms before feeling slow

Optimization by measured latency:
- **< 100ms latency**: No optimization needed. User won't notice.
- **100–300ms latency**: CSS loading states (Level 0) are sufficient.
- **> 300ms latency**: JS commands (Level 1) or optimistic patterns (Level 2-5) improve perceived speed.
- **Destructive actions**: Use undo buffer (Level 5) or confirm on server first.

### When NOT to Use Optimistic UI

- **Destructive actions** (delete, cancel subscription) — confirm first, then execute
- **Actions with side effects** (send email, charge payment) — show loading, wait for confirmation
- **Validation-dependent actions** (submit form with server-side validation) — wait for server
- **Actions where rollback is confusing** (reordering a list that other users see) — show loading instead

### Rollback Strategies

| Strategy | When to use | How it works |
|----------|------------|---------------|
| **Server-corrects** | Low-risk toggles | Apply JS command; server patch fixes if wrong |
| **Pending state** | Medium-risk changes | Mark item as pending; confirm or rollback on server response |
| **Undo buffer** | Destructive actions | Don't execute for N seconds; show "Undo" toast |
| **No optimism** | Financial/critical | Show spinner; wait for server confirmation before updating UI |

### Patterns by Use Case

| Use case | Recommended approach |
|----------|---------------------|
| Like/favorite toggle | Level 1: JS.toggle_class |
| Checkbox toggle | Level 1: JS.toggle_class |
| Drag-and-drop reorder | Level 2-3: Optimistic move + rollback |
| Form submission | Level 0: phx-disable-with |
| Item deletion | Level 5: Undo buffer (5s delay) |
| Message sending | Level 4: Optimistic stream insert with "pending" indicator |
| Payment/checkout | No optimism: spinner + server confirmation |
| Real-time counters | Level 1: JS increment + server reconciliation |

## Presence Indicators

### Setting Up Phoenix Presence

```elixir
# lib/my_app_web/presence.ex
defmodule MyAppWeb.Presence do
  use Phoenix.Presence,
    otp_app: :my_app,
    pubsub_server: MyApp.PubSub
end
```

### Tracking in a LiveView

```elixir
def mount(_params, _session, socket) do
  topic = "document:#{socket.assigns.document_id}"

  if connected?(socket) do
    Phoenix.PubSub.subscribe(MyApp.PubSub, topic)

    MyAppWeb.Presence.track(self(), topic, socket.assigns.current_user.id, %{
      name: socket.assigns.current_user.name,
      avatar_url: socket.assigns.current_user.avatar_url,
      color: user_color(socket.assigns.current_user.id),
      joined_at: System.system_time(:second),
      status: "active"
    })
  end

  presences = if connected?(socket) do
    MyAppWeb.Presence.list(topic) |> simplify_presences()
  else
    []
  end

  {:ok, assign(socket, presences: presences)}
end

def handle_info(%{event: "presence_diff", payload: _diff}, socket) do
  topic = "document:#{socket.assigns.document_id}"
  presences = MyAppWeb.Presence.list(topic) |> simplify_presences()
  {:noreply, assign(socket, presences: presences)}
end

defp simplify_presences(presences) do
  Enum.map(presences, fn {user_id, %{metas: [meta | _]}} ->
    Map.put(meta, :user_id, user_id)
  end)
end

# Assign a consistent color per user for avatars/cursors/field indicators
defp user_color(user_id) do
  colors = ~w(#3B82F6 #EF4444 #10B981 #F59E0B #8B5CF6 #EC4899 #06B6D4 #F97316)
  index = :erlang.phash2(user_id, length(colors))
  Enum.at(colors, index)
end

# Reusable helper for updating presence metadata
defp update_presence_meta(socket, updates) do
  topic = "document:#{socket.assigns.document_id}"
  user_id = socket.assigns.current_user.id
  MyAppWeb.Presence.update(self(), topic, user_id, fn meta ->
    Map.merge(meta, updates)
  end)
end
```

### Avatar Stacks

Show who's present with a compact avatar stack:

```heex
<div class="flex items-center">
  <%!-- Avatar stack --%>
  <div class="flex -space-x-2">
    <div
      :for={user <- Enum.take(@presences, 5)}
      class="relative"
      title={user.name}
    >
      <img
        src={user.avatar_url}
        alt={user.name}
        class={[
          "size-8 rounded-full border-2 border-surface object-cover",
          user.status == "idle" && "opacity-50"
        ]}
        style={"border-color: #{user.color}"}
      />
      <div class={[
        "absolute -bottom-0.5 -right-0.5 size-2.5 rounded-full border-2 border-surface",
        status_color(user.status)
      ]} />
    </div>
  </div>

  <%!-- Overflow count --%>
  <div :if={length(@presences) > 5}
    class="ml-1 flex size-8 items-center justify-center rounded-full bg-muted text-xs font-medium">
    +{length(@presences) - 5}
  </div>

  <%!-- User count label --%>
  <span class="ml-sm text-sm text-muted-foreground">
    {length(@presences)} {if length(@presences) == 1, do: "person", else: "people"} viewing
  </span>
</div>
```

```elixir
defp status_color("active"), do: "bg-green-500"
defp status_color("idle"), do: "bg-yellow-500"
defp status_color(_), do: "bg-gray-400"
```

### Typing Indicators

Debounce the typing event to avoid flooding the server:

```heex
<textarea
  phx-keyup="typing"
  phx-debounce="500"
  phx-blur="stop_typing"
/>
```

```elixir
def handle_event("typing", _params, socket) do
  update_presence_meta(socket, %{typing: true, typing_at: System.system_time(:second)})

  if timer = socket.assigns[:typing_timer] do
    Process.cancel_timer(timer)
  end
  timer = Process.send_after(self(), :clear_typing, 3_000)

  {:noreply, assign(socket, typing_timer: timer)}
end

def handle_event("stop_typing", _params, socket) do
  update_presence_meta(socket, %{typing: false})
  {:noreply, socket}
end

def handle_info(:clear_typing, socket) do
  update_presence_meta(socket, %{typing: false})
  {:noreply, socket}
end
```

Display with bouncing dots and grammatically correct text:

```heex
<div class="h-6 flex items-center">
  {#case typing_users(@presences, @current_user.id)}
    {%{count: 0}} ->

    {%{names: [name], count: 1}} ->
      <div class="flex items-center gap-xs text-sm text-muted-foreground">
        <span class="flex gap-0.5">
          <span class="size-1.5 rounded-full bg-muted-foreground animate-bounce [animation-delay:0ms]"></span>
          <span class="size-1.5 rounded-full bg-muted-foreground animate-bounce [animation-delay:150ms]"></span>
          <span class="size-1.5 rounded-full bg-muted-foreground animate-bounce [animation-delay:300ms]"></span>
        </span>
        {name} is typing
      </div>

    {%{names: [a, b], count: 2}} ->
      <span class="text-sm text-muted-foreground">{a} and {b} are typing</span>

    {%{names: [a | _], count: n}} ->
      <span class="text-sm text-muted-foreground">{a} and {n - 1} others are typing</span>
  {/case}
</div>
```

```elixir
defp typing_users(presences, current_user_id) do
  typers = presences
    |> Enum.filter(&(&1.user_id != current_user_id and &1[:typing]))
    |> Enum.map(& &1.name)

  %{names: typers, count: length(typers)}
end
```

### Active Field Indicators

Show which fields other users are currently editing:

```elixir
def handle_event("focus_field", %{"field" => field_name}, socket) do
  update_presence_meta(socket, %{active_field: field_name})
  {:noreply, socket}
end

def handle_event("blur_field", _params, socket) do
  update_presence_meta(socket, %{active_field: nil})
  {:noreply, socket}
end
```

```heex
<div class="relative">
  <input
    name="title"
    phx-focus={JS.push("focus_field", value: %{field: "title"})}
    phx-blur={JS.push("blur_field")}
    class="input"
  />
  <%!-- Show who else is editing this field --%>
  <div :for={user <- field_editors(@presences, "title", @current_user.id)}
    class="absolute -top-2 -right-2 flex items-center gap-xs rounded-full px-xs py-0.5 text-xs text-white"
    style={"background-color: #{user.color}"}
  >
    <img src={user.avatar_url} class="size-4 rounded-full" />
    {user.name |> String.split() |> hd()}
  </div>
</div>
```

```elixir
defp field_editors(presences, field_name, current_user_id) do
  Enum.filter(presences, &(&1.user_id != current_user_id and &1[:active_field] == field_name))
end
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
  update_presence_meta(socket, %{status: "idle"})
  {:noreply, socket}
end

def handle_event("user_active", _params, socket) do
  update_presence_meta(socket, %{status: "active"})
  {:noreply, socket}
end
```

### Cursor/Selection Presence

For collaborative editing — show where other users are:

```elixir
def handle_event("cursor_moved", %{"line" => line, "col" => col}, socket) do
  update_presence_meta(socket, %{cursor_line: line, cursor_col: col, last_active: DateTime.utc_now()})
  {:noreply, socket}
end
```

### UX Guidelines for Presence

1. **Animate presence changes.** Don't pop avatars in/out. Fade them.
2. **Show, don't tell.** An avatar with a green dot communicates "online" better than text.
3. **Auto-expire indicators.** Typing indicators must clear after 3-5s of inactivity.
4. **Debounce aggressively.** Don't send presence updates on every keystroke. Debounce to 300-500ms.
5. **Limit visual noise.** Don't show cursors unless the feature specifically needs them (e.g., collaborative whiteboard). For most apps, avatar stacks + typing indicators are sufficient.
6. **Handle edge cases.** A user with two tabs open appears as one presence (use the first meta). A user who closes their laptop stays "present" until the heartbeat timeout (~60s) — consider showing "idle" state after 30s of no activity.
7. **Color coding.** Assign consistent colors to users (hash the user ID). Use these colors for avatar borders, active field indicators, and cursors.
8. **Be honest about staleness.** If presence data is > 30s old, fade or hide it.
9. **Respect privacy.** Not every feature needs to show who's viewing.
10. **Throttle cursor broadcasts.** Max 10/sec — don't broadcast on every mousemove at 60fps.

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

## Disconnect & Recovery

### How LiveView Reconnection Works

When the WebSocket disconnects, LiveView's client-side library automatically:

1. Attempts reconnection immediately
2. Falls back to exponential backoff: 0s → 2s → 5s → 10s → ...
3. On reconnection, calls `mount/3` and `handle_params/3` again
4. Replays form state via `phx-auto-recover`
5. Re-renders the full page from the new server state

CSS classes applied during connection state changes:
- `phx-connected` — added to the LiveView container when connected
- `phx-disconnected` — added when connection is lost
- `phx-error` — added when an unrecoverable error occurs

Hook lifecycle callbacks:
- `disconnected()` — called on each hook when the LiveView disconnects
- `reconnected()` — called on each hook when the LiveView reconnects

### Connection State UI

#### Delayed Disconnect Banner

Don't show a disconnect banner immediately — brief disconnects during deploys shouldn't alarm users. Show after 2-3 seconds:

```javascript
// app.js
let disconnectTimeout;

window.addEventListener("phx:page-loading-start", (info) => {
  if (info.detail?.kind === "error" || info.detail?.kind === "initial") {
    disconnectTimeout = setTimeout(() => {
      document.getElementById("connection-banner")?.classList.remove("hidden");
    }, 2000);
  }
});

window.addEventListener("phx:page-loading-stop", () => {
  clearTimeout(disconnectTimeout);
  document.getElementById("connection-banner")?.classList.add("hidden");
});
```

```heex
<div id="connection-banner" class="hidden fixed top-0 inset-x-0 z-50">
  <div class="bg-yellow-500 text-yellow-950 text-sm text-center py-2 px-4 flex items-center justify-center gap-2">
    <svg class="animate-spin h-4 w-4" viewBox="0 0 24 24">
      <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4" fill="none"/>
      <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4z"/>
    </svg>
    Reconnecting to server...
  </div>
</div>
```

#### Extended Disconnect with Retry

After 15-30 seconds, offer a manual refresh:

```javascript
// app.js
let longDisconnectTimeout;

window.addEventListener("phx:page-loading-start", (info) => {
  longDisconnectTimeout = setTimeout(() => {
    document.getElementById("connection-banner-extended")?.classList.remove("hidden");
  }, 15000);
});

window.addEventListener("phx:page-loading-stop", () => {
  clearTimeout(longDisconnectTimeout);
  document.getElementById("connection-banner-extended")?.classList.add("hidden");
});
```

```heex
<div id="connection-banner-extended" class="hidden fixed inset-0 z-50 bg-black/50 flex items-center justify-center">
  <div class="bg-surface-raised rounded-xl shadow-xl p-6 max-w-sm text-center">
    <div class="text-lg font-semibold mb-2">Connection Lost</div>
    <p class="text-text-secondary mb-4">
      We're having trouble connecting to the server. Your unsaved changes are preserved.
    </p>
    <button onclick="window.location.reload()"
            class="bg-brand-600 text-white rounded-md px-4 py-2 font-medium hover:bg-brand-700">
      Refresh Page
    </button>
  </div>
</div>
```

#### Disabling Actions While Disconnected

Use CSS to disable destructive actions that require a server connection:

```css
#app.phx-disconnected [data-requires-connection] {
  opacity: 0.4;
  pointer-events: none;
  cursor: not-allowed;
}

#app.phx-disconnected [data-requires-connection]::after {
  content: " (offline)";
  font-size: 0.75rem;
}
```

```heex
<button phx-click="delete" data-requires-connection>Delete</button>
<button phx-click="save" data-requires-connection>Save</button>
```

### Form Recovery

LiveView automatically recovers form state on reconnection by re-submitting the last `phx-change` event. This works for simple forms out of the box.

#### Auto-Recovery (Default)

```heex
<form phx-change="validate" phx-submit="save">
  <input name="user[name]" value={@form[:name].value} />
  <input name="user[email]" value={@form[:email].value} />
</form>
```

On reconnect, LiveView collects all form field values and re-submits the `phx-change` event, restoring server state to match what the user sees.

#### Custom Recovery for Complex Forms

For multi-step wizards or forms with dynamic fields, use `phx-auto-recover`:

```heex
<form phx-change="validate_step" phx-submit="save" phx-auto-recover="recover_form">
  <!-- Dynamic fields based on current step -->
</form>
```

```elixir
def handle_event("recover_form", params, socket) do
  # Reconstruct form state from params, possibly adjusting the step
  {:noreply, assign(socket, form: rebuild_form(params))}
end
```

#### Protecting Unsaved Work (Client-Side Backup)

For complex forms where auto-recovery isn't sufficient:

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

### State Recovery Strategies

#### URL-Based State (Preferred)

Store important state in URL query params. This survives reconnection, page refresh, browser back, and link sharing:

```elixir
# Instead of:
def handle_event("select_tab", %{"tab" => tab}, socket) do
  {:noreply, assign(socket, active_tab: tab)}
end

# Use:
def handle_event("select_tab", %{"tab" => tab}, socket) do
  {:noreply, push_patch(socket, to: ~p"/dashboard?tab=#{tab}")}
end

def handle_params(%{"tab" => tab}, _uri, socket) do
  {:noreply, assign(socket, active_tab: tab)}
end
```

#### Server-Side Draft Saving

For state that can't live in the URL (draft content, complex filters):

```elixir
def mount(_params, _session, socket) do
  draft = Drafts.get_for_user(socket.assigns.current_user.id)
  {:ok, assign(socket, form: form_from_draft(draft))}
end

def handle_event("validate", params, socket) do
  Drafts.save(socket.assigns.current_user.id, params)
  {:noreply, assign(socket, form: changeset(params))}
end
```

#### Client-Side State with Hooks

For ephemeral state like scroll position:

```javascript
const ScrollRestore = {
  mounted() {
    const saved = sessionStorage.getItem(`scroll:${this.el.id}`);
    if (saved) this.el.scrollTop = parseInt(saved);
  },
  updated() {
    sessionStorage.setItem(`scroll:${this.el.id}`, this.el.scrollTop);
  },
  reconnected() {
    const saved = sessionStorage.getItem(`scroll:${this.el.id}`);
    if (saved) this.el.scrollTop = parseInt(saved);
  }
};
```

#### Reconnection State Refresh

Ensure `mount/3` can rebuild state from URL params + database, not just socket assigns:

```elixir
def mount(_params, _session, socket) do
  if connected?(socket) do
    Phoenix.PubSub.subscribe(MyApp.PubSub, "document:#{socket.assigns.document_id}")
  end

  # Always load fresh data on mount (handles reconnection)
  document = Documents.get_document!(socket.assigns.document_id)
  {:ok, assign(socket, :document, document)}
end
```

### Deployment Handling

#### Detecting New Deployments

```elixir
def mount(_params, _session, socket) do
  {:ok, assign(socket, static_changed?: static_changed?(socket))}
end
```

```heex
<div :if={@static_changed?} class="fixed bottom-4 inset-x-4 z-50 mx-auto max-w-md">
  <div class="bg-brand-600 text-white rounded-lg px-4 py-3 shadow-xl flex items-center justify-between">
    <span class="text-sm">A new version is available</span>
    <a href="" class="text-sm font-semibold underline">Refresh</a>
  </div>
</div>
```

#### Zero-Downtime Deploy Checklist

1. Use `phx-track-static` on CSS and JS includes in your root layout
2. Check `static_changed?/1` in mount to detect stale clients
3. Use rolling deploys so at least one node is always available
4. Test form recovery: `liveSocket.disconnect(); setTimeout(() => liveSocket.connect(), 500);`
5. Ensure `mount/3` can rebuild state from URL params + database (not just socket assigns)

### Offline-Capable Actions

For critical actions that should survive disconnects:

```javascript
Hooks.ResilientAction = {
  mounted() {
    this.el.addEventListener("click", (e) => {
      const action = this.el.dataset.action;
      const payload = JSON.parse(this.el.dataset.payload || "{}");

      this.pushEvent(action, payload, (reply) => {
        this._removeFromQueue(action, payload);
      });

      this._addToQueue(action, payload);
    });
  },

  reconnected() {
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

### Loading State Hierarchy

From worst to best user experience:

1. **Nothing** — User wonders if anything is happening (never do this)
2. **Spinner** — User knows something is loading but not what
3. **Progress bar** — User knows roughly how long to wait
4. **Skeleton** — User can see the shape of what's coming, feels faster
5. **Optimistic render** — User sees the data immediately (stale or predicted)

| Content type | Recommended | Why |
|-------------|-------------|-----|
| Button action | `phx-disable-with` | Inline, minimal disruption |
| Small data fetch | Spinner | Content area is small |
| Page/section load | Skeleton | Gives spatial context |
| Known data shape | Skeleton matching layout | Reduces perceived wait time |
| List/feed | Progressive (stream) | Load what you can, show more on demand |
| Previously loaded data | Stale-while-revalidate | Show old data immediately, update in background |

### Reusable Skeleton Primitives

```elixir
defmodule MyAppWeb.UI.Skeleton do
  use Phoenix.Component

  attr :class, :string, default: nil

  def skeleton(assigns) do
    ~H"""
    <div class={["animate-pulse rounded-md bg-muted", @class]} />
    """
  end

  def skeleton_text(assigns) do
    ~H"""
    <div class="space-y-2">
      <.skeleton class="h-4 w-full" />
      <.skeleton class="h-4 w-5/6" />
      <.skeleton class="h-4 w-4/6" />
    </div>
    """
  end

  def skeleton_card(assigns) do
    ~H"""
    <div class="rounded-lg border border-border p-lg space-y-md">
      <div class="flex items-center gap-md">
        <.skeleton class="size-10 rounded-full" />
        <div class="space-y-xs flex-1">
          <.skeleton class="h-4 w-1/3" />
          <.skeleton class="h-3 w-1/4" />
        </div>
      </div>
      <.skeleton_text />
    </div>
    """
  end

  def skeleton_table(assigns) do
    assigns = assign_new(assigns, :rows, fn -> 5 end)
    assigns = assign_new(assigns, :cols, fn -> 4 end)

    ~H"""
    <div class="space-y-sm">
      <div class="flex gap-lg border-b border-border pb-sm">
        <.skeleton :for={_ <- 1..@cols} class="h-4 flex-1" />
      </div>
      <div :for={_ <- 1..@rows} class="flex gap-lg py-sm">
        <.skeleton :for={_ <- 1..@cols} class="h-4 flex-1" />
      </div>
    </div>
    """
  end
end
```

### Using Skeletons with assign_async

```elixir
def mount(_params, _session, socket) do
  {:ok,
    socket
    |> assign_async(:dashboard_stats, fn ->
      {:ok, %{dashboard_stats: Stats.compute_dashboard()}}
    end)
    |> assign_async(:recent_activity, fn ->
      {:ok, %{recent_activity: Activity.recent(limit: 10)}}
    end)
  }
end
```

```heex
<div class="grid grid-cols-3 gap-lg">
  <.async_result :let={stats} assign={@dashboard_stats}>
    <:loading>
      <.skeleton class="h-24 rounded-lg" />
      <.skeleton class="h-24 rounded-lg" />
      <.skeleton class="h-24 rounded-lg" />
    </:loading>
    <:failed :let={_reason}>
      <div class="col-span-3 text-center text-destructive py-lg">
        Failed to load stats.
        <button phx-click="retry_stats" class="underline ml-xs">Retry</button>
      </div>
    </:failed>
    <.stat_card :for={stat <- stats} title={stat.label} value={stat.value} />
  </.async_result>
</div>

<div class="mt-xl">
  <.async_result :let={activities} assign={@recent_activity}>
    <:loading>
      <.skeleton_card />
      <.skeleton_card />
      <.skeleton_card />
    </:loading>
    <:failed :let={_reason}>
      <p class="text-destructive">Could not load activity.</p>
    </:failed>
    <.activity_item :for={activity <- activities} activity={activity} />
  </.async_result>
</div>
```

### Inline Loading Indicators

For actions that take time but don't need a full skeleton:

```heex
<button phx-click="export_csv" class="inline-flex items-center gap-2">
  <svg class="hidden phx-click-loading:block size-4 animate-spin" viewBox="0 0 24 24">
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
     class="w-full bg-muted rounded-full h-2 overflow-hidden">
  <div class="bg-primary h-full rounded-full transition-all duration-300"
       style={"width: #{@import_progress}%"} />
</div>
```

### Progressive Loading with Streams

For lists and feeds, load incrementally rather than showing a skeleton for the whole list:

```elixir
def mount(_params, _session, socket) do
  {:ok,
    socket
    |> stream(:messages, Messages.list(limit: 20))
    |> assign(page: 1, loading_more: false, has_more: true)
  }
end

def handle_event("load_more", _params, socket) do
  next_page = socket.assigns.page + 1
  messages = Messages.list(page: next_page, limit: 20)

  {:noreply,
    socket
    |> stream(:messages, messages)
    |> assign(page: next_page, loading_more: false, has_more: length(messages) == 20)
  }
end
```

```heex
<div id="messages" phx-update="stream">
  <div :for={{dom_id, message} <- @streams.messages} id={dom_id}
    phx-mounted={JS.transition({"ease-out duration-200", "opacity-0", "opacity-100"}, time: 200)}
  >
    <.message_bubble message={message} />
  </div>
</div>

<div :if={@loading_more} class="flex justify-center py-md">
  <svg class="animate-spin size-5 text-muted-foreground" viewBox="0 0 24 24">
    <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4" fill="none"/>
    <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4z"/>
  </svg>
</div>

<button
  :if={@has_more && !@loading_more}
  phx-click="load_more"
  class="w-full py-md text-sm text-muted-foreground hover:text-foreground transition-colors"
>
  Load more
</button>
```

#### Infinite Scroll with Viewport Hook

```javascript
const InfiniteScroll = {
  mounted() {
    this._observer = new IntersectionObserver(
      (entries) => {
        if (entries[0].isIntersecting) {
          this.pushEvent("load_more", {});
        }
      },
      { rootMargin: "200px" }
    );
    this._observer.observe(this.el);
  },
  destroyed() {
    this._observer?.disconnect();
  }
};
```

```heex
<div :if={@has_more} id="scroll-sentinel" phx-hook="InfiniteScroll"></div>
```

### Stale-While-Revalidate

Show previously loaded data immediately, then refresh in the background:

```elixir
def mount(_params, _session, socket) do
  cached = Cache.get("dashboard:#{socket.assigns.current_user.id}")

  socket = if cached do
    assign(socket, data: cached, stale: true)
  else
    assign(socket, data: nil, stale: false)
  end

  if connected?(socket), do: send(self(), :refresh_data)

  {:ok, socket}
end

def handle_info(:refresh_data, socket) do
  fresh_data = DataSource.fetch()
  Cache.put("dashboard:#{socket.assigns.current_user.id}", fresh_data)
  {:noreply, assign(socket, data: fresh_data, stale: false)}
end
```

```heex
<div class="relative">
  <div :if={@stale} class="absolute top-xs right-xs">
    <div class="flex items-center gap-xs text-xs text-muted-foreground">
      <svg class="animate-spin size-3" viewBox="0 0 24 24">
        <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4" fill="none"/>
        <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4z"/>
      </svg>
      Updating...
    </div>
  </div>

  <div :if={@data}>
    <%!-- Render data (cached or fresh) --%>
  </div>
  <div :if={!@data}>
    <.skeleton_card />
  </div>
</div>
```

### Loading Anti-Patterns

1. **Full-page spinner.** Never block the entire page with a spinner. Use skeletons
   for the loading area and keep the rest of the page interactive.

2. **Skeleton flash.** If data loads in < 200ms, the skeleton appears and disappears
   too quickly, creating a flash. Add a minimum display time or delay the skeleton.

3. **Mismatched skeleton shape.** A skeleton that looks nothing like the real content
   confuses users. Match the approximate shape, spacing, and count of the real content.

4. **No error state.** Always handle the failed case of `assign_async`. Show a helpful
   error message with a retry option, not a blank page.

5. **Loading state that blocks navigation.** Users should be able to navigate away
   from a loading page. Don't trap them.

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
