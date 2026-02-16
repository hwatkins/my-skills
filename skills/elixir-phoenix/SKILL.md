---
name: elixir-phoenix
description: Expert Elixir and Phoenix development with functional programming patterns, Phoenix 1.7/1.8 conventions, Ecto, and OTP. Use for any Elixir or Phoenix code.
---

# Elixir & Phoenix Development

You are an expert in Elixir and Phoenix development with deep knowledge of functional programming and concurrent systems.

## Core Principles

- Write concise, idiomatic Elixir code with accurate examples
- Follow Phoenix conventions and best practices
- Embrace functional programming patterns and immutability
- Prefer higher-order functions and recursion over imperative loops
- Use descriptive naming (e.g., `user_signed_in?`, `calculate_total`)
- **Functional core, imperative shell** — pure domain logic with side effects at boundaries

## Functional Core, Imperative Shell

Separate pure business logic from side effects:

```elixir
# ✅ Good: Pure function in context (functional core)
defmodule MyApp.Pricing do
  def calculate_total(line_items, discount_code \\ nil) do
    subtotal = Enum.reduce(line_items, 0, &(&1.price_cents * &1.quantity + &2))
    discount = apply_discount(subtotal, discount_code)
    tax = calculate_tax(subtotal - discount)
    %{subtotal: subtotal, discount: discount, tax: tax, total: subtotal - discount + tax}
  end

  defp apply_discount(subtotal, nil), do: 0
  defp apply_discount(subtotal, %{type: :percent, value: pct}), do: div(subtotal * pct, 100)
  defp apply_discount(subtotal, %{type: :fixed, value: amount}), do: min(amount, subtotal)
end

# Side effects at the boundary (imperative shell)
defmodule MyApp.Orders do
  def place_order(user, cart, discount_code) do
    totals = Pricing.calculate_total(cart.line_items, discount_code)

    Multi.new()
    |> Multi.insert(:order, Order.changeset(%Order{}, Map.put(totals, :user_id, user.id)))
    |> Multi.run(:payment, fn _, %{order: order} -> charge_payment(order) end)
    |> Repo.transaction()
  end
end
```

## Naming Conventions

- Use `snake_case` for files, functions, and variables
- Use `PascalCase` for module names
- Start function names with verbs (`create_user`, not `user_create`)
- Use `?` suffix for boolean-returning functions (`valid?`, `admin?`)
- Use `!` suffix for functions that raise on error (`get_user!`)
- Follow Phoenix conventions for contexts, schemas, and controllers

## Typespecs & Documentation

Add `@spec` and `@doc` to all public functions:

```elixir
defmodule MyApp.Accounts do
  @doc """
  Creates a user with the given attributes.

  ## Examples

      iex> create_user(%{email: "test@example.com", name: "Test"})
      {:ok, %User{}}

      iex> create_user(%{email: "invalid"})
      {:error, %Ecto.Changeset{}}
  """
  @spec create_user(map()) :: {:ok, User.t()} | {:error, Ecto.Changeset.t()}
  def create_user(attrs) do
    %User{} |> User.changeset(attrs) |> Repo.insert()
  end
end
```

## Static Analysis

- Run `mix format` on save (enforce with CI)
- Run `mix credo --strict` for code quality
- Run `mix dialyzer` for type checking
- Add `@moduledoc` to every module, `@doc` to public functions

## Technical Practices

### Elixir & Phoenix Usage

- Use Elixir's pattern matching and guards effectively
- Leverage Phoenix's built-in functions and macros
- Use Ecto effectively for database operations
- Use Phoenix LiveView for dynamic, real-time interactions

### Formatting

- Follow the Elixir Style Guide
- Use Elixir's pipe operator `|>` for function chaining
- Prefer single quotes for charlists, double quotes for strings

### Error Handling

- Use Elixir's "let it crash" philosophy and supervisor trees
- Implement proper error logging with user-friendly messages
- Use Ecto changesets for validation
- Handle errors gracefully with flash messages
- Use `with` for chaining operations that can fail (railway-oriented programming)

### HTTP Clients

- Use `Req` for HTTP requests (modern replacement for HTTPoison/Tesla)
- Define behaviours for API clients to enable Mox testing
- Always set timeouts on external calls
- Use circuit breakers for critical external services

```elixir
# Define a behaviour for testability
defmodule MyApp.WeatherAPI do
  @callback get_forecast(String.t()) :: {:ok, map()} | {:error, term()}
end

defmodule MyApp.WeatherAPI.Client do
  @behaviour MyApp.WeatherAPI

  @impl true
  def get_forecast(city) do
    case Req.get("https://api.weather.com/forecast", params: [city: city], receive_timeout: 5_000) do
      {:ok, %{status: 200, body: body}} -> {:ok, body}
      {:ok, %{status: status}} -> {:error, {:http_error, status}}
      {:error, reason} -> {:error, reason}
    end
  end
end
```

## Performance

- Optimize with database indexing and caching (ETS, Redis)
- Use Ecto's `preload` to avoid N+1 queries
- Leverage OTP patterns for concurrent operations
- Use process pooling for resource management
- Use Oban for background jobs (emails, reports, webhooks) — never block the request path

## Phoenix 1.7+ Patterns

- Use function components with `attr` and `slot` declarations
- Use `embed_templates` for organizing template files
- Prefer verified routes (`~p"/path"`) over route helpers
- Use streams for efficient list rendering in LiveView
- Use `phx-` bindings for LiveView interactivity
- Implement `CoreComponents` for reusable UI elements

## Phoenix 1.8 Changes

### Scopes

Scopes replace the manual `on_mount` + `assign_current_user` pattern:

```elixir
# router.ex — Phoenix 1.8 scopes
scope "/", MyAppWeb do
  pipe_through [:browser, :require_authenticated_user]

  # Scope automatically assigns current_user and enforces auth
  scope MyAppWeb.UserScope do
    live "/dashboard", DashboardLive
    live "/settings", SettingsLive
  end
end
```

```elixir
# Scope module
defmodule MyAppWeb.UserScope do
  use Phoenix.Scope

  @impl true
  def assign_scope(conn_or_socket) do
    user = conn_or_socket.assigns[:current_user]
    %MyAppWeb.UserScope{user: user, id: user.id}
  end
end
```

### broadcast_from

Use `broadcast_from` to send PubSub messages to all subscribers except the sender:

```elixir
# ✅ Good: Don't echo back to the sender
def update_item(item, attrs, socket) do
  case Repo.update(Item.changeset(item, attrs)) do
    {:ok, item} ->
      Phoenix.PubSub.broadcast_from(MyApp.PubSub, self(), "items:#{item.id}", {:item_updated, item})
      {:ok, item}
    error ->
      error
  end
end
```

## HEEx Templates

- Use `{expression}` for interpolation (not `<%= %>` in function components)
- Use `:if` and `:for` attributes for conditionals and loops
- Prefer `<.link>` component over `link/2` function
- Use `<.form>` component with `for` attribute for changesets
- Implement `phx-feedback-for` to show errors only after user interaction

## Security

- Implement proper authentication and authorization (e.g., Guardian, Pow)
- Use strong parameters in controllers (params validation)
- Protect against common web vulnerabilities (XSS, CSRF, SQL injection)
- Never trust client-side data — always validate on the server

## Code Organization

- Use contexts for organizing related functionality
- Follow RESTful routing conventions
- Implement GenServers for stateful processes and background jobs
- Use Tasks for concurrent, isolated jobs
- Separate pure functions from side effects (functional core, imperative shell)
- Consider API/implementation separation for complex contexts:

```elixir
# Public API module (thin, delegates to implementations)
defmodule MyApp.Accounts do
  defdelegate create_user(attrs), to: MyApp.Accounts.CreateUser
  defdelegate authenticate(email, password), to: MyApp.Accounts.Authenticate
end

# Implementation module (focused, single responsibility)
defmodule MyApp.Accounts.CreateUser do
  def create_user(attrs) do
    %User{} |> User.registration_changeset(attrs) |> Repo.insert()
  end
end
```
