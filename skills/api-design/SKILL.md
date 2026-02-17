---
name: api-design
description: When the user is designing, building, or reviewing a REST/JSON API. Covers resource design, routing, error handling, pagination, authentication, idempotency, webhooks, and OpenAPI documentation. Phoenix-focused with Elixir examples. Use when mentioning "API design," "REST API," "JSON API," "API endpoints," "webhook design," or "API versioning."
---

# REST API Design

Expert guidance for designing clean, consistent, production-grade REST APIs in Phoenix.

## Core Principles

- **Resources are nouns** — `/users`, `/orders`, not `/createUser`, `/getOrders`
- **HTTP methods are verbs** — GET reads, POST creates, PUT/PATCH updates, DELETE removes
- **Consistent everywhere** — same patterns for naming, errors, pagination, auth across all endpoints
- **Stateless** — each request contains all information needed; no server-side session state
- **Predictable** — a developer who's seen one endpoint can guess how the rest work

## Resource Design

### URL Structure

```
GET    /api/v1/users              # List users
POST   /api/v1/users              # Create user
GET    /api/v1/users/:id          # Get user
PATCH  /api/v1/users/:id          # Update user
DELETE /api/v1/users/:id          # Delete user

# Nested resources (one level deep max)
GET    /api/v1/users/:user_id/orders    # List user's orders
POST   /api/v1/users/:user_id/orders    # Create order for user

# Deeper nesting → flatten with filters
GET    /api/v1/order_items?order_id=123  # Not /users/:id/orders/:id/items
```

**Rules:**
- Plural nouns for collections (`/users`, not `/user`)
- Kebab-case for multi-word resources (`/order-items` or snake_case `/order_items` — pick one, be consistent)
- No verbs in URLs — use HTTP methods instead
- Max one level of nesting; flatten deeper relationships with query params
- Use UUIDs for public-facing IDs (don't expose sequential integers)

### HTTP Methods

| Method | Purpose | Idempotent | Request Body | Response |
|--------|---------|------------|--------------|----------|
| GET | Read resource(s) | Yes | No | 200 + data |
| POST | Create resource | No | Yes | 201 + created resource |
| PATCH | Partial update | No* | Yes | 200 + updated resource |
| PUT | Full replace | Yes | Yes | 200 + replaced resource |
| DELETE | Remove resource | Yes | No | 204 (no body) |

*PATCH is idempotent if applied to the same state, but not guaranteed across concurrent changes.

### Actions That Don't Fit CRUD

Some operations aren't resource CRUD. Use sub-resource verbs sparingly:

```
POST /api/v1/users/:id/activate       # State transition
POST /api/v1/orders/:id/cancel        # State transition
POST /api/v1/reports/generate          # Trigger async job
```

## Phoenix Router & Controllers

### Router

```elixir
# lib/my_app_web/router.ex
scope "/api/v1", MyAppWeb.API.V1, as: :api_v1 do
  pipe_through [:api, :authenticate_api]

  resources "/users", UserController, except: [:new, :edit]
  resources "/orders", OrderController, except: [:new, :edit] do
    resources "/items", OrderItemController, only: [:index, :create]
  end

  # Custom actions
  post "/orders/:id/cancel", OrderController, :cancel
end
```

### Controller Pattern

```elixir
defmodule MyAppWeb.API.V1.UserController do
  use MyAppWeb, :controller

  action_fallback MyAppWeb.FallbackController

  def index(conn, params) do
    with {:ok, pagination} <- validate_pagination(params),
         {:ok, filters} <- validate_filters(params, [:status, :role]) do
      page = Accounts.list_users(filters, pagination)

      conn
      |> put_status(200)
      |> render(:index, users: page.entries, pagination: page)
    end
  end

  def show(conn, %{"id" => id}) do
    with {:ok, user} <- Accounts.get_user(id) do
      render(conn, :show, user: user)
    end
  end

  def create(conn, %{"user" => user_params}) do
    with {:ok, user} <- Accounts.create_user(user_params) do
      conn
      |> put_status(201)
      |> put_resp_header("location", ~p"/api/v1/users/#{user.id}")
      |> render(:show, user: user)
    end
  end

  def update(conn, %{"id" => id, "user" => user_params}) do
    with {:ok, user} <- Accounts.get_user(id),
         {:ok, updated} <- Accounts.update_user(user, user_params) do
      render(conn, :show, user: updated)
    end
  end

  def delete(conn, %{"id" => id}) do
    with {:ok, user} <- Accounts.get_user(id),
         {:ok, _} <- Accounts.delete_user(user) do
      send_resp(conn, 204, "")
    end
  end
end
```

### FallbackController

Centralize error handling — don't repeat error rendering in every action.

```elixir
defmodule MyAppWeb.FallbackController do
  use MyAppWeb, :controller

  def call(conn, {:error, :not_found}) do
    conn
    |> put_status(404)
    |> put_view(json: MyAppWeb.ErrorJSON)
    |> render("404.json")
  end

  def call(conn, {:error, :unauthorized}) do
    conn
    |> put_status(401)
    |> put_view(json: MyAppWeb.ErrorJSON)
    |> render("401.json")
  end

  def call(conn, {:error, :forbidden}) do
    conn
    |> put_status(403)
    |> put_view(json: MyAppWeb.ErrorJSON)
    |> render("403.json")
  end

  def call(conn, {:error, %Ecto.Changeset{} = changeset}) do
    conn
    |> put_status(422)
    |> put_view(json: MyAppWeb.ErrorJSON)
    |> render("422.json", changeset: changeset)
  end

  def call(conn, {:error, :rate_limited}) do
    conn
    |> put_status(429)
    |> put_resp_header("retry-after", "60")
    |> put_view(json: MyAppWeb.ErrorJSON)
    |> render("429.json")
  end
end
```

### JSON Views

```elixir
defmodule MyAppWeb.API.V1.UserJSON do
  def index(%{users: users, pagination: page}) do
    %{
      data: for(user <- users, do: data(user)),
      meta: %{
        total: page.total_entries,
        page: page.page_number,
        per_page: page.page_size,
        total_pages: page.total_pages
      }
    }
  end

  def show(%{user: user}) do
    %{data: data(user)}
  end

  defp data(user) do
    %{
      id: user.id,
      email: user.email,
      name: user.name,
      role: user.role,
      inserted_at: user.inserted_at,
      updated_at: user.updated_at
    }
  end
end
```

## Error Responses

### Consistent Error Format

Every error response follows the same shape:

```json
{
  "error": {
    "type": "validation_error",
    "message": "Request validation failed",
    "details": [
      {"field": "email", "message": "has already been taken"},
      {"field": "name", "message": "can't be blank"}
    ]
  }
}
```

```elixir
defmodule MyAppWeb.ErrorJSON do
  def render("401.json", _assigns) do
    %{error: %{type: "unauthorized", message: "Invalid or missing authentication"}}
  end

  def render("403.json", _assigns) do
    %{error: %{type: "forbidden", message: "You don't have permission for this action"}}
  end

  def render("404.json", _assigns) do
    %{error: %{type: "not_found", message: "Resource not found"}}
  end

  def render("422.json", %{changeset: changeset}) do
    errors =
      Ecto.Changeset.traverse_errors(changeset, fn {msg, opts} ->
        Regex.replace(~r"%{(\w+)}", msg, fn _, key ->
          opts |> Keyword.get(String.to_existing_atom(key), key) |> to_string()
        end)
      end)

    details =
      Enum.flat_map(errors, fn {field, messages} ->
        Enum.map(messages, &%{field: field, message: &1})
      end)

    %{error: %{type: "validation_error", message: "Validation failed", details: details}}
  end

  def render("429.json", _assigns) do
    %{error: %{type: "rate_limited", message: "Too many requests, please try again later"}}
  end
end
```

### HTTP Status Codes

| Code | When to Use |
|------|-------------|
| 200 | Successful read or update |
| 201 | Resource created (include `Location` header) |
| 204 | Successful delete (no body) |
| 400 | Malformed request (bad JSON, missing required params) |
| 401 | Missing or invalid authentication |
| 403 | Authenticated but not authorized |
| 404 | Resource not found (also use for unauthorized resource access to avoid leaking existence) |
| 409 | Conflict (duplicate, state conflict) |
| 422 | Validation errors (well-formed request, invalid data) |
| 429 | Rate limited |
| 500 | Server error (never intentionally return this) |

## Pagination

### Cursor-Based (Preferred)

Better performance than offset for large datasets. No skipped/duplicated items.

```elixir
defmodule MyApp.Pagination do
  import Ecto.Query

  def paginate(query, %{cursor: nil, per_page: per_page}) do
    query
    |> order_by([r], desc: r.inserted_at, desc: r.id)
    |> limit(^(per_page + 1))
    |> Repo.all()
    |> build_page(per_page)
  end

  def paginate(query, %{cursor: cursor, per_page: per_page}) do
    {inserted_at, id} = decode_cursor!(cursor)

    query
    |> where([r], {r.inserted_at, r.id} < {^inserted_at, ^id})
    |> order_by([r], desc: r.inserted_at, desc: r.id)
    |> limit(^(per_page + 1))
    |> Repo.all()
    |> build_page(per_page)
  end

  defp build_page(records, per_page) do
    has_more = length(records) > per_page
    entries = Enum.take(records, per_page)
    next_cursor = if has_more, do: encode_cursor(List.last(entries))

    %{entries: entries, has_more: has_more, next_cursor: next_cursor}
  end

  defp encode_cursor(record) do
    {record.inserted_at, record.id}
    |> :erlang.term_to_binary()
    |> Base.url_encode64(padding: false)
  end

  defp decode_cursor!(cursor) do
    cursor
    |> Base.url_decode64!(padding: false)
    |> :erlang.binary_to_term([:safe])
  end
end
```

**Response:**

```json
{
  "data": [...],
  "meta": {
    "has_more": true,
    "next_cursor": "g3QAAAACZAAJaW5zZXJ0ZWRfYXQ..."
  }
}
```

### Offset-Based (Simple Cases)

Use only for small datasets or when clients need page numbers.

```elixir
def list_users(params) do
  page = Map.get(params, "page", 1)
  per_page = params |> Map.get("per_page", 20) |> min(100)

  query = from(u in User, order_by: [desc: u.inserted_at])

  total = Repo.aggregate(query, :count)
  entries = query |> limit(^per_page) |> offset(^((page - 1) * per_page)) |> Repo.all()

  %{entries: entries, total: total, page: page, per_page: per_page,
    total_pages: ceil(total / per_page)}
end
```

## Filtering & Sorting

```elixir
defmodule MyApp.QueryFilters do
  import Ecto.Query

  @allowed_filters [:status, :role, :created_after, :created_before, :search]
  @allowed_sorts [:name, :email, :inserted_at]

  def apply_filters(query, params) do
    Enum.reduce(params, query, fn
      {"status", status}, q -> where(q, [r], r.status == ^status)
      {"role", role}, q -> where(q, [r], r.role == ^role)
      {"created_after", date}, q -> where(q, [r], r.inserted_at >= ^date)
      {"created_before", date}, q -> where(q, [r], r.inserted_at <= ^date)
      {"search", term}, q ->
        where(q, [r], ilike(r.name, ^"%#{term}%") or ilike(r.email, ^"%#{term}%"))
      _, q -> q  # Ignore unknown filters
    end)
  end

  def apply_sort(query, params) do
    sort_by = params |> Map.get("sort_by", "inserted_at") |> String.to_existing_atom()
    sort_order = if Map.get(params, "sort_order") == "asc", do: :asc, else: :desc

    if sort_by in @allowed_sorts do
      order_by(query, [r], [{^sort_order, field(r, ^sort_by)}])
    else
      order_by(query, [r], [desc: r.inserted_at])
    end
  end
end
```

**Request:** `GET /api/v1/users?status=active&sort_by=name&sort_order=asc&per_page=25`

## Authentication

### Bearer Token (API Keys)

```elixir
defmodule MyAppWeb.Plugs.AuthenticateAPI do
  import Plug.Conn

  def init(opts), do: opts

  def call(conn, _opts) do
    with ["Bearer " <> token] <- get_req_header(conn, "authorization"),
         {:ok, api_token} <- Accounts.verify_api_token(token) do
      conn
      |> assign(:current_user, api_token.user)
      |> assign(:api_token, api_token)
    else
      _ ->
        conn
        |> put_status(401)
        |> Phoenix.Controller.put_view(json: MyAppWeb.ErrorJSON)
        |> Phoenix.Controller.render("401.json")
        |> halt()
    end
  end
end
```

### Scoped Access

```elixir
# Scope all queries to the authenticated user/org
def index(conn, params) do
  org_id = conn.assigns.current_user.organization_id
  projects = Projects.list_for_org(org_id, params)
  render(conn, :index, projects: projects)
end

# Never trust user-supplied IDs without scoping
def show(conn, %{"id" => id}) do
  org_id = conn.assigns.current_user.organization_id

  case Projects.get_for_org(org_id, id) do
    {:ok, project} -> render(conn, :show, project: project)
    {:error, :not_found} -> {:error, :not_found}
  end
end
```

## Idempotency

For non-idempotent operations (POST), support idempotency keys to allow safe retries.

```elixir
defmodule MyAppWeb.Plugs.Idempotency do
  import Plug.Conn

  def init(opts), do: opts

  def call(%{method: "POST"} = conn, _opts) do
    case get_req_header(conn, "idempotency-key") do
      [key] when byte_size(key) > 0 ->
        case MyApp.IdempotencyStore.get(key) do
          {:ok, cached_response} ->
            conn
            |> put_status(cached_response.status)
            |> put_resp_content_type("application/json")
            |> send_resp(cached_response.status, cached_response.body)
            |> halt()
          :miss ->
            conn
            |> assign(:idempotency_key, key)
            |> register_before_send(&cache_response/1)
        end
      _ -> conn
    end
  end

  def call(conn, _opts), do: conn

  defp cache_response(conn) do
    if key = conn.assigns[:idempotency_key] do
      MyApp.IdempotencyStore.put(key, %{
        status: conn.status,
        body: conn.resp_body
      }, ttl: :timer.hours(24))
    end
    conn
  end
end
```

**Client usage:**
```
POST /api/v1/orders
Idempotency-Key: ord_req_abc123
Content-Type: application/json

{"order": {"product_id": "...", "quantity": 1}}
```

## Webhooks (Outbound)

When your SaaS sends events to customer endpoints.

### Event Design

```elixir
defmodule MyApp.Webhooks do
  def deliver(event_type, resource, webhook_url, signing_secret) do
    payload = %{
      id: Ecto.UUID.generate(),
      type: event_type,
      created_at: DateTime.utc_now(),
      data: resource
    }

    body = Jason.encode!(payload)
    signature = sign_payload(body, signing_secret)

    headers = [
      {"content-type", "application/json"},
      {"x-webhook-signature", signature},
      {"x-webhook-id", payload.id},
      {"x-webhook-timestamp", to_string(payload.created_at)}
    ]

    # Use Oban for reliable delivery with retries
    %{url: webhook_url, body: body, headers: headers}
    |> MyApp.Workers.WebhookDelivery.new(max_attempts: 5)
    |> Oban.insert()
  end

  defp sign_payload(body, secret) do
    :crypto.mac(:hmac, :sha256, secret, body) |> Base.encode16(case: :lower)
  end
end
```

### Event Types

Use past-tense, dot-namespaced event types:

```
user.created
user.updated
user.deleted
order.created
order.paid
order.canceled
subscription.activated
subscription.canceled
invoice.payment_succeeded
invoice.payment_failed
```

### Retry Strategy

```elixir
defmodule MyApp.Workers.WebhookDelivery do
  use Oban.Worker,
    queue: :webhooks,
    max_attempts: 5,
    # Exponential backoff: 1min, 5min, 25min, 2hr, 10hr
    backoff: {MyApp.Workers.WebhookDelivery, :backoff}

  @impl Oban.Worker
  def perform(%Oban.Job{args: %{"url" => url, "body" => body, "headers" => headers}}) do
    case Req.post(url, body: body, headers: headers, receive_timeout: 10_000) do
      {:ok, %{status: status}} when status in 200..299 -> :ok
      {:ok, %{status: status}} -> {:error, "HTTP #{status}"}
      {:error, reason} -> {:error, reason}
    end
  end

  def backoff(%Oban.Job{attempt: attempt}) do
    trunc(:math.pow(5, attempt) * 60)
  end
end
```

## Versioning

### URL Versioning (Recommended)

```elixir
# router.ex
scope "/api/v1", MyAppWeb.API.V1 do
  pipe_through [:api, :authenticate_api]
  resources "/users", UserController, except: [:new, :edit]
end

scope "/api/v2", MyAppWeb.API.V2 do
  pipe_through [:api, :authenticate_api]
  resources "/users", UserController, except: [:new, :edit]
end
```

**Rules:**
- Only bump version for **breaking changes** (removing fields, changing types, renaming)
- Adding new fields is NOT a breaking change
- Support previous version for at least 6-12 months
- Document deprecation timeline in API docs
- Use the same context functions; only views/controllers differ between versions

## Rate Limiting

```elixir
defmodule MyAppWeb.Plugs.RateLimit do
  import Plug.Conn

  def init(opts), do: opts

  def call(conn, opts) do
    limit = Keyword.get(opts, :limit, 100)
    window = Keyword.get(opts, :window_ms, 60_000)
    key = rate_limit_key(conn)

    case Hammer.check_rate(key, window, limit) do
      {:allow, count} ->
        conn
        |> put_resp_header("x-ratelimit-limit", to_string(limit))
        |> put_resp_header("x-ratelimit-remaining", to_string(limit - count))

      {:deny, _} ->
        {:error, :rate_limited}
    end
  end

  defp rate_limit_key(conn) do
    case conn.assigns[:current_user] do
      %{id: id} -> "api:user:#{id}"
      nil -> "api:ip:#{conn.remote_ip |> :inet.ntoa() |> to_string()}"
    end
  end
end

# In router — different limits for different endpoints
scope "/api/v1" do
  pipe_through [:api, :authenticate_api]

  scope "/" do
    pipe_through [{MyAppWeb.Plugs.RateLimit, limit: 100, window_ms: 60_000}]
    resources "/users", UserController
  end

  scope "/" do
    pipe_through [{MyAppWeb.Plugs.RateLimit, limit: 10, window_ms: 60_000}]
    post "/reports/generate", ReportController, :generate
  end
end
```

## API Documentation

### OpenAPI with `open_api_spex`

```elixir
# mix.exs
{:open_api_spex, "~> 3.18"}

# Define schemas
defmodule MyAppWeb.Schemas.User do
  require OpenApiSpex
  alias OpenApiSpex.Schema

  OpenApiSpex.schema(%{
    title: "User",
    type: :object,
    required: [:email, :name],
    properties: %{
      id: %Schema{type: :string, format: :uuid},
      email: %Schema{type: :string, format: :email},
      name: %Schema{type: :string, minLength: 1},
      role: %Schema{type: :string, enum: ["admin", "member"]},
      inserted_at: %Schema{type: :string, format: :"date-time"},
      updated_at: %Schema{type: :string, format: :"date-time"}
    }
  })
end

# In controller — annotate operations
use OpenApiSpex.ControllerSpecs

operation :index,
  summary: "List users",
  parameters: [
    page: [in: :query, type: :integer, description: "Page number"],
    per_page: [in: :query, type: :integer, description: "Items per page (max 100)"],
    status: [in: :query, type: :string, description: "Filter by status"]
  ],
  responses: [
    ok: {"User list", "application/json", MyAppWeb.Schemas.UserListResponse}
  ]
```

## Design Checklist

### Before Building
- [ ] Resources identified and named (plural nouns)
- [ ] URL structure documented
- [ ] HTTP methods mapped to operations
- [ ] Error response format defined
- [ ] Pagination strategy chosen (cursor vs offset)
- [ ] Authentication method chosen
- [ ] Rate limits defined per endpoint
- [ ] Versioning strategy decided

### Before Shipping
- [ ] All endpoints return consistent error format
- [ ] Validation errors include field-level details
- [ ] Pagination works on all list endpoints
- [ ] Rate limiting active with appropriate headers
- [ ] Authentication required on all non-public endpoints
- [ ] All queries scoped to authenticated user/org
- [ ] Idempotency keys supported on POST endpoints
- [ ] OpenAPI spec generated and accessible
- [ ] Webhook signatures implemented (if applicable)

## Common Mistakes

- **Verbs in URLs** — `POST /api/createUser` → `POST /api/v1/users`
- **Inconsistent error formats** — use FallbackController to centralize
- **Exposing sequential IDs** — use UUIDs for public-facing resources
- **No pagination** — every list endpoint must paginate
- **Offset pagination on large tables** — use cursor-based
- **Trusting client IDs** — always scope queries to authenticated user/org
- **No rate limiting** — every endpoint needs limits
- **Breaking changes without versioning** — removing/renaming fields breaks clients
- **Leaking internal errors** — never expose stack traces or Ecto errors directly

## Related Skills

- **elixir-phoenix**: Phoenix controller patterns, routing, plugs
- **saas-security**: Authentication, authorization, API token management
- **sql-optimization-patterns**: Cursor pagination, query optimization
- **postgresql-table-design**: Schema design for API-backed resources
- **stripe-integration**: Webhook patterns (inbound)
