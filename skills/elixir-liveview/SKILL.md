---
name: elixir-liveview
description: Phoenix LiveView patterns for real-time web applications. Covers lifecycle, streams, events, PubSub, forms, and performance optimization. Use when building or debugging LiveView features.
---

# Phoenix LiveView Patterns

Expert guidance for building real-time, interactive web applications with Phoenix LiveView.

## Lifecycle & State

- Keep `mount/3` minimal — assign only what's needed for initial render
- Use `handle_params/3` for URL-driven state, not `mount/3`
- Prefer `assign_new/3` over `assign/3` when value may already exist
- Use `assign_async/3` and `start_async/3` for expensive operations
- Never block `mount/3` with slow database queries or API calls

```elixir
# ✅ Good: Minimal mount, async loading
def mount(_params, _session, socket) do
  {:ok, assign(socket, page_title: "Dashboard", loading: true)}
end

def handle_params(%{"id" => id}, _uri, socket) do
  {:noreply,
   socket
   |> assign(:id, id)
   |> start_async(:load_data, fn -> fetch_data(id) end)}
end

def handle_async(:load_data, {:ok, data}, socket) do
  {:noreply, assign(socket, data: data, loading: false)}
end

# ❌ Bad: Blocking mount
def mount(%{"id" => id}, _session, socket) do
  data = Repo.get!(Item, id) |> Repo.preload(:associations)  # Blocks render
  {:ok, assign(socket, data: data)}
end
```

## Streams for Collections

- Use streams (`stream/3`, `stream_insert/3`, `stream_delete/3`) for lists that change
- Never re-assign entire lists when only one item changes
- Use `phx-update="stream"` with streams, not `phx-update="replace"`
- Provide DOM IDs for stream items

```elixir
# ✅ Good: Using streams
def mount(_params, _session, socket) do
  {:ok, stream(socket, :items, list_items())}
end

def handle_info({:item_created, item}, socket) do
  {:noreply, stream_insert(socket, :items, item, at: 0)}
end

def handle_info({:item_deleted, item}, socket) do
  {:noreply, stream_delete(socket, :items, item)}
end
```

```heex
<%!-- Template with streams --%>
<div id="items" phx-update="stream">
  <div :for={{dom_id, item} <- @streams.items} id={dom_id}>
    <%= item.name %>
  </div>
</div>
```

```elixir
# ❌ Bad: Re-assigning entire list
def handle_info({:item_created, item}, socket) do
  items = [item | socket.assigns.items]  # Forces full re-render
  {:noreply, assign(socket, items: items)}
end
```

## Events & Messages

- Validate all params in `handle_event/3` — never trust client data
- Use `push_patch/2` for same-LiveView navigation, `push_navigate/2` for different LiveView
- Handle `handle_info/2` for PubSub, async results, and process messages
- Return `{:noreply, socket}` from event handlers, not just `socket`

```elixir
# ✅ Good: Validating event params
def handle_event("delete", %{"id" => id}, socket) do
  case Integer.parse(id) do
    {id, ""} ->
      item = Items.get_item!(id)
      # Verify authorization
      if item.user_id == socket.assigns.current_user.id do
        Items.delete_item(item)
        {:noreply, stream_delete(socket, :items, item)}
      else
        {:noreply, put_flash(socket, :error, "Not authorized")}
      end
    :error ->
      {:noreply, put_flash(socket, :error, "Invalid ID")}
  end
end

# ❌ Bad: Trusting client data
def handle_event("delete", %{"id" => id}, socket) do
  Items.delete_item!(id)  # No validation, no auth check
  {:noreply, socket}
end
```

## Components

- Use `<.live_component>` only when you need isolated state or `handle_event`
- Prefer stateless function components for pure rendering
- Always define `@impl true` for LiveView callbacks
- Use `on_mount/1` hooks for shared authentication/authorization logic

```elixir
# Function component (preferred for stateless rendering)
attr :item, :map, required: true
attr :on_delete, :any, default: nil

def item_card(assigns) do
  ~H"""
  <div class="card">
    <h3><%= @item.title %></h3>
    <button :if={@on_delete} phx-click={@on_delete} phx-value-id={@item.id}>
      Delete
    </button>
  </div>
  """
end

# LiveComponent (only when you need isolated state)
defmodule MyAppWeb.ItemFormComponent do
  use MyAppWeb, :live_component

  @impl true
  def mount(socket) do
    {:ok, assign(socket, form: to_form(%{}))}
  end

  @impl true
  def handle_event("save", params, socket) do
    # Component handles its own events
    {:noreply, socket}
  end
end
```

## Performance

- Use `temporary_assigns` for large lists rendered once
- Debounce rapid events with `phx-debounce` (forms: 300ms, search: 500ms)
- Use `phx-throttle` for scroll/resize events
- Avoid assigns that change on every render — LiveView diffs on assign changes

```elixir
# ✅ Good: Temporary assigns for large static lists
def mount(_params, _session, socket) do
  {:ok, assign(socket, items: list_all_items()), temporary_assigns: [items: []]}
end
```

```heex
<%!-- Debounce search input --%>
<input type="text" name="q" phx-change="search" phx-debounce="500" />

<%!-- Throttle scroll events --%>
<div phx-hook="InfiniteScroll" phx-throttle="100">
```

## PubSub & Real-time

- Subscribe in `mount/3` only when `connected?(socket)` is true
- Unsubscribe happens automatically on disconnect — don't manage manually
- Broadcast granular topics (e.g., `"task:#{id}"`) not global topics
- Use `Phoenix.Presence` for user tracking, not custom GenServers

```elixir
@impl true
def mount(%{"id" => id}, _session, socket) do
  if connected?(socket) do
    # Subscribe only when connected (not during static render)
    Phoenix.PubSub.subscribe(MyApp.PubSub, "item:#{id}")
  end

  {:ok, assign(socket, id: id, item: get_item(id))}
end

@impl true
def handle_info({:item_updated, item}, socket) do
  {:noreply, assign(socket, item: item)}
end

# Broadcasting from context
def update_item(item, attrs) do
  case Repo.update(Item.changeset(item, attrs)) do
    {:ok, item} ->
      Phoenix.PubSub.broadcast(MyApp.PubSub, "item:#{item.id}", {:item_updated, item})
      {:ok, item}
    error ->
      error
  end
end
```

## Forms

- Use `to_form/1` to convert changesets to form structs
- Handle `phx-change` for validation, `phx-submit` for persistence
- Use `phx-trigger-action` for traditional form submissions when needed
- Implement `phx-feedback-for` to show errors only after user interaction

```elixir
@impl true
def mount(_params, _session, socket) do
  changeset = Items.change_item(%Item{})
  {:ok, assign(socket, form: to_form(changeset))}
end

@impl true
def handle_event("validate", %{"item" => params}, socket) do
  changeset =
    %Item{}
    |> Items.change_item(params)
    |> Map.put(:action, :validate)

  {:noreply, assign(socket, form: to_form(changeset))}
end

@impl true
def handle_event("save", %{"item" => params}, socket) do
  case Items.create_item(params) do
    {:ok, item} ->
      {:noreply,
       socket
       |> put_flash(:info, "Created!")
       |> push_navigate(to: ~p"/items/#{item}")}

    {:error, changeset} ->
      {:noreply, assign(socket, form: to_form(changeset))}
  end
end
```

```heex
<.form for={@form} phx-change="validate" phx-submit="save">
  <.input field={@form[:title]} label="Title" phx-feedback-for="item[title]" />
  <.input field={@form[:body]} type="textarea" label="Body" />
  <.button>Save</.button>
</.form>
```

## Navigation

- Use `push_patch/2` when staying on the same LiveView (updates URL, calls `handle_params`)
- Use `push_navigate/2` when going to a different LiveView
- Use `<.link patch={...}>` and `<.link navigate={...}>` in templates

```elixir
# Same LiveView, different params (e.g., pagination, filters)
{:noreply, push_patch(socket, to: ~p"/items?page=#{page + 1}")}

# Different LiveView
{:noreply, push_navigate(socket, to: ~p"/items/#{item.id}")}
```

## Common Mistakes

```elixir
# ❌ Don't use assigns directly in comprehensions
<div :for={item <- @items}>  # Fine for small lists
  
# ✅ Use streams for dynamic lists
<div :for={{id, item} <- @streams.items} id={id}>

# ❌ Don't subscribe in mount without checking connected?
def mount(_, _, socket) do
  PubSub.subscribe(...)  # Will fail on static render

# ✅ Check connected? first
def mount(_, _, socket) do
  if connected?(socket), do: PubSub.subscribe(...)

# ❌ Don't forget @impl true
def mount(_, _, socket) do  # Missing @impl

# ✅ Always use @impl true
@impl true
def mount(_, _, socket) do
```
