---
name: elixir-ecto
description: Expert Ecto patterns for Elixir — changesets, Multi, composable queries, migrations, optimistic locking, multi-tenancy, and railway-oriented programming with `with`. Use when working with databases or data validation in Elixir.
---

# Ecto Patterns

Expert guidance for database operations, data validation, and query composition with Ecto.

## Core Principles

- Ecto is not an ORM — it's a toolkit for data mapping and validation
- Changesets are for validation AND tracking changes, not just database inserts
- Queries are composable data structures, not method chains on models
- Separate read and write concerns
- Use the database as the source of truth

## Changesets

### Validation Outside the Database

Use embedded schemas and changesets for validating data that never touches the DB:

```elixir
# ✅ Good: Validate API params with embedded schema
defmodule MyApp.Params.SearchFilter do
  use Ecto.Schema
  import Ecto.Changeset

  @primary_key false
  embedded_schema do
    field :query, :string
    field :sort_by, Ecto.Enum, values: [:name, :date, :relevance]
    field :page, :integer, default: 1
    field :per_page, :integer, default: 20
  end

  def changeset(params) do
    %__MODULE__{}
    |> cast(params, [:query, :sort_by, :page, :per_page])
    |> validate_required([:query])
    |> validate_number(:page, greater_than: 0)
    |> validate_number(:per_page, greater_than: 0, less_than_or_equal_to: 100)
  end

  def validate(params) do
    case changeset(params) |> apply_action(:validate) do
      {:ok, filter} -> {:ok, filter}
      {:error, changeset} -> {:error, changeset}
    end
  end
end
```

### Changeset Pipelines

```elixir
# ✅ Good: Composable changeset functions
defmodule MyApp.Accounts.User do
  def changeset(user, attrs) do
    user
    |> cast(attrs, [:email, :name, :role])
    |> validate_required([:email, :name])
    |> validate_format(:email, ~r/@/)
    |> unique_constraint(:email)
    |> normalize_email()
  end

  def registration_changeset(user, attrs) do
    user
    |> changeset(attrs)
    |> cast(attrs, [:password])
    |> validate_required([:password])
    |> validate_length(:password, min: 12)
    |> hash_password()
  end

  defp normalize_email(changeset) do
    update_change(changeset, :email, &String.downcase/1)
  end

  defp hash_password(changeset) do
    case get_change(changeset, :password) do
      nil -> changeset
      password -> put_change(changeset, :hashed_password, Bcrypt.hash_pwd_salt(password))
    end
  end
end
```

## Railway-Oriented Programming with `with`

Chain operations that can fail, bailing out on the first error:

```elixir
# ✅ Good: with chain for multi-step operations
def transfer_funds(from_id, to_id, amount) do
  with {:ok, from} <- Accounts.get_account(from_id),
       {:ok, to} <- Accounts.get_account(to_id),
       :ok <- validate_sufficient_funds(from, amount),
       {:ok, result} <- execute_transfer(from, to, amount) do
    {:ok, result}
  else
    {:error, :not_found} -> {:error, :account_not_found}
    {:error, :insufficient_funds} -> {:error, :insufficient_funds}
    {:error, changeset} -> {:error, changeset}
  end
end

# ❌ Bad: Nested case statements
def transfer_funds(from_id, to_id, amount) do
  case Accounts.get_account(from_id) do
    {:ok, from} ->
      case Accounts.get_account(to_id) do
        {:ok, to} ->
          case validate_sufficient_funds(from, amount) do
            :ok -> execute_transfer(from, to, amount)
            error -> error
          end
        error -> error
      end
    error -> error
  end
end
```

## Ecto.Multi for Transactions

Use `Ecto.Multi` when multiple database operations must succeed or fail together:

```elixir
# ✅ Good: Multi for atomic operations
def create_order(user, cart_items) do
  Multi.new()
  |> Multi.insert(:order, Order.changeset(%Order{}, %{user_id: user.id, status: :pending}))
  |> Multi.run(:line_items, fn repo, %{order: order} ->
    items =
      Enum.map(cart_items, fn item ->
        %{order_id: order.id, product_id: item.product_id, quantity: item.quantity}
      end)

    {count, _} = repo.insert_all(LineItem, items)

    if count == length(cart_items) do
      {:ok, count}
    else
      {:error, :incomplete_insert}
    end
  end)
  |> Multi.run(:inventory, fn repo, %{order: order} ->
    reduce_inventory(repo, cart_items)
  end)
  |> Multi.run(:total, fn repo, %{order: order, line_items: _} ->
    total = calculate_total(order.id)
    order |> Order.changeset(%{total_cents: total}) |> repo.update()
  end)
  |> Repo.transaction()
  |> case do
    {:ok, %{order: order, total: updated_order}} ->
      {:ok, updated_order}

    {:error, failed_operation, failed_value, _changes} ->
      {:error, {failed_operation, failed_value}}
  end
end

# ❌ Bad: Multiple separate Repo calls without transaction
def create_order(user, cart_items) do
  {:ok, order} = Repo.insert(Order.changeset(%Order{}, %{user_id: user.id}))
  # If this fails, orphaned order exists!
  Enum.each(cart_items, fn item ->
    Repo.insert!(LineItem.changeset(%LineItem{}, %{order_id: order.id, ...}))
  end)
end
```

## Optimistic Locking

Prevent concurrent updates from overwriting each other:

```elixir
# Schema: add a lock_version field
schema "tasks" do
  field :title, :string
  field :status, Ecto.Enum, values: [:todo, :in_progress, :done]
  field :lock_version, :integer, default: 1
  timestamps()
end

# ✅ Good: Optimistic lock on update
def update_task(task, attrs) do
  task
  |> Task.changeset(attrs)
  |> optimistic_lock(:lock_version)
  |> Repo.update()
end

# Handle the conflict
case update_task(task, attrs) do
  {:ok, updated_task} ->
    {:ok, updated_task}

  {:error, %Ecto.Changeset{errors: [lock_version: _]}} ->
    # Reload and retry, or inform the user
    {:error, :stale_data}

  {:error, changeset} ->
    {:error, changeset}
end
```

## Composable Queries

Build queries from small, reusable pieces:

```elixir
defmodule MyApp.Tasks.Query do
  import Ecto.Query

  def base, do: from(t in Task, as: :task)

  def by_status(query, status) do
    where(query, [task: t], t.status == ^status)
  end

  def by_user(query, user_id) do
    where(query, [task: t], t.user_id == ^user_id)
  end

  def search(query, nil), do: query
  def search(query, ""), do: query
  def search(query, term) do
    term = "%#{term}%"
    where(query, [task: t], ilike(t.title, ^term) or ilike(t.description, ^term))
  end

  def with_assignee(query) do
    join(query, :left, [task: t], u in assoc(t, :assignee), as: :assignee)
    |> preload([assignee: u], assignee: u)
  end

  def order_by_recent(query) do
    order_by(query, [task: t], desc: t.inserted_at)
  end

  def paginate(query, page, per_page) do
    offset = (page - 1) * per_page

    query
    |> limit(^per_page)
    |> offset(^offset)
  end
end

# Usage: compose queries from filters
def list_tasks(user_id, filters \\ %{}) do
  Query.base()
  |> Query.by_user(user_id)
  |> maybe_filter_status(filters)
  |> maybe_search(filters)
  |> Query.with_assignee()
  |> Query.order_by_recent()
  |> Query.paginate(filters[:page] || 1, filters[:per_page] || 20)
  |> Repo.all()
end

defp maybe_filter_status(query, %{status: status}) when not is_nil(status),
  do: Query.by_status(query, status)
defp maybe_filter_status(query, _), do: query

defp maybe_search(query, %{search: term}), do: Query.search(query, term)
defp maybe_search(query, _), do: query
```

## Dynamic Queries

For truly dynamic filter sets:

```elixir
def filter_tasks(filters) when is_map(filters) do
  query = from(t in Task)

  Enum.reduce(filters, query, fn
    {:status, status}, query ->
      from t in query, where: t.status == ^status

    {:assigned_to, user_id}, query ->
      from t in query, where: t.assignee_id == ^user_id

    {:created_after, date}, query ->
      from t in query, where: t.inserted_at >= ^date

    {:search, term}, query when is_binary(term) and term != "" ->
      from t in query, where: ilike(t.title, ^"%#{term}%")

    _, query ->
      query
  end)
  |> Repo.all()
end
```

## Preloading (Avoiding N+1)

```elixir
# ✅ Good: Preload in the query
def list_posts_with_authors do
  from(p in Post,
    join: a in assoc(p, :author),
    preload: [author: a],
    order_by: [desc: p.inserted_at]
  )
  |> Repo.all()
end

# ✅ Good: Selective preload with query
def list_posts_with_recent_comments do
  recent_comments = from(c in Comment, order_by: [desc: c.inserted_at], limit: 5)

  from(p in Post, preload: [comments: ^recent_comments])
  |> Repo.all()
end

# ❌ Bad: N+1 — loading associations in a loop
posts = Repo.all(Post)
Enum.map(posts, fn post ->
  Repo.preload(post, :author)  # One query per post!
end)
```

## Migrations

```elixir
# ✅ Good: Safe migration with index
defmodule MyApp.Repo.Migrations.CreateTasks do
  use Ecto.Migration

  def change do
    create table(:tasks) do
      add :title, :string, null: false
      add :status, :string, null: false, default: "todo"
      add :user_id, references(:users, on_delete: :delete_all), null: false
      add :lock_version, :integer, default: 1
      timestamps()
    end

    create index(:tasks, [:user_id])
    create index(:tasks, [:status])
    create index(:tasks, [:user_id, :status])
  end
end

# ✅ Good: Data migration separated from schema migration
# Step 1: Add column (nullable)
def change do
  alter table(:users) do
    add :display_name, :string
  end
end

# Step 2: Backfill data (in a separate migration or mix task)
# Step 3: Make column non-nullable
def change do
  alter table(:users) do
    modify :display_name, :string, null: false
  end
end
```

## Multi-Tenancy

### Query Prefix (Schema-Based)

```elixir
# Per-tenant schema prefix
def list_tasks(tenant) do
  Task
  |> Ecto.Queryable.to_query()
  |> Map.put(:prefix, "tenant_#{tenant.id}")
  |> Repo.all()
end

# Or set prefix on the repo call
Repo.all(Task, prefix: "tenant_#{tenant.id}")
Repo.insert(changeset, prefix: "tenant_#{tenant.id}")
```

### Foreign Key (Column-Based)

```elixir
# Scope all queries by tenant_id
def list_tasks(tenant_id) do
  from(t in Task, where: t.tenant_id == ^tenant_id)
  |> Repo.all()
end

# Plug to set tenant in conn
def set_tenant(conn, _opts) do
  tenant_id = conn.assigns.current_user.tenant_id
  assign(conn, :tenant_id, tenant_id)
end
```

## Upserts

```elixir
# ✅ Good: Upsert with on_conflict
def upsert_setting(user_id, key, value) do
  %Setting{}
  |> Setting.changeset(%{user_id: user_id, key: key, value: value})
  |> Repo.insert(
    on_conflict: [set: [value: value, updated_at: DateTime.utc_now()]],
    conflict_target: [:user_id, :key]
  )
end
```

## Common Mistakes

```elixir
# ❌ Don't use Repo in schemas
defmodule MyApp.User do
  def create(attrs) do
    %__MODULE__{} |> changeset(attrs) |> Repo.insert()  # Schema shouldn't know about Repo
  end
end

# ✅ Keep Repo calls in context modules
defmodule MyApp.Accounts do
  def create_user(attrs) do
    %User{} |> User.changeset(attrs) |> Repo.insert()
  end
end

# ❌ Don't use floats for money
schema "products" do
  field :price, :float  # Floating point errors!
end

# ✅ Use integer cents or Decimal
schema "products" do
  field :price_cents, :integer
  # or
  field :price, :decimal
end

# ❌ Don't forget to handle Repo.transaction errors properly
{:ok, result} = Repo.transaction(multi)  # Crashes on error!

# ✅ Pattern match both cases
case Repo.transaction(multi) do
  {:ok, %{user: user}} -> {:ok, user}
  {:error, :user, changeset, _} -> {:error, changeset}
end
```
