---
name: saas-security
description: When the user needs to implement or review security for a SaaS application. Covers authentication, authorization, API security, account takeover prevention, session management, and security headers. Also use when mentioning "auth security," "API protection," "account takeover," "session hijacking," "CSRF," "XSS," or "security headers." For spam-specific issues, see spam-prevention.
---

# SaaS Security Best Practices

Expert guidance for securing SaaS applications — authentication, authorization, API protection, session management, and defense against common attacks.

## Core Principles

- **Defense in depth** — multiple layers, never rely on a single control
- **Least privilege** — grant minimum access needed, default deny
- **Fail secure** — errors should deny access, not grant it
- **Don't roll your own crypto** — use proven libraries and standards
- **Assume breach** — design systems to limit blast radius

## Authentication

### Password Security

```elixir
# Use bcrypt with sufficient cost factor (default 12 is good)
defmodule MyApp.User do
  use Ecto.Schema
  import Ecto.Changeset

  def registration_changeset(user, attrs) do
    user
    |> cast(attrs, [:email, :password])
    |> validate_required([:email, :password])
    |> validate_length(:password, min: 12, max: 72)
    |> validate_format(:email, ~r/^[^\s]+@[^\s]+$/)
    |> unique_constraint(:email)
    |> hash_password()
  end

  defp hash_password(changeset) do
    case get_change(changeset, :password) do
      nil -> changeset
      password ->
        changeset
        |> put_change(:hashed_password, Bcrypt.hash_pwd_salt(password))
        |> delete_change(:password)
    end
  end
end
```

**Password Rules:**
- Minimum 12 characters (NIST SP 800-63B)
- Maximum 72 characters (bcrypt limit)
- No composition rules (uppercase, special chars) — they don't help
- Check against breached password lists (Have I Been Pwned API)
- Rate limit login attempts

#### Breached Password Check

```elixir
def password_breached?(password) do
  hash = :crypto.hash(:sha, password) |> Base.encode16()
  prefix = String.slice(hash, 0, 5)
  suffix = String.slice(hash, 5, String.length(hash))

  case Req.get("https://api.pwnedpasswords.com/range/#{prefix}") do
    {:ok, %{status: 200, body: body}} ->
      body
      |> String.split("\r\n")
      |> Enum.any?(fn line ->
        line |> String.split(":") |> List.first() == suffix
      end)
    _ -> false  # Fail open — don't block signup if API is down
  end
end
```

### Multi-Factor Authentication (MFA)

```elixir
# TOTP (Time-based One-Time Password) — use NimbleTOTP
def generate_totp_secret do
  NimbleTOTP.secret()
end

def generate_totp_uri(secret, email) do
  NimbleTOTP.otpauth_uri("MyApp:#{email}", secret, issuer: "MyApp")
end

def verify_totp(secret, code) do
  NimbleTOTP.valid?(secret, code)
end
```

**MFA Best Practices:**
- Offer TOTP (authenticator apps) as primary MFA
- Support WebAuthn/passkeys for phishing-resistant auth
- Provide recovery codes (store hashed, show once)
- Don't use SMS for MFA if possible (SIM swap attacks)
- Require MFA for admin accounts

### Session Management

```elixir
# Session configuration
plug Plug.Session,
  store: :cookie,
  key: "_myapp_key",
  signing_salt: "random_salt_here",
  encryption_salt: "random_encryption_salt",
  max_age: 86_400,           # 24 hours
  same_site: "Lax",          # CSRF protection
  secure: true,              # HTTPS only
  http_only: true            # No JavaScript access

# Rotate session on login (prevent session fixation)
def log_in_user(conn, user) do
  conn
  |> configure_session(renew: true)  # New session ID
  |> put_session(:user_id, user.id)
  |> put_session(:live_socket_id, "users_sessions:#{user.id}")
end

# Invalidate all sessions on password change
def on_password_change(user) do
  Phoenix.PubSub.broadcast(
    MyApp.PubSub,
    "users_sessions:#{user.id}",
    :logout
  )
end
```

### Token Security

```elixir
# API tokens — use signed, time-limited tokens
def generate_api_token(user) do
  token = :crypto.strong_rand_bytes(32) |> Base.url_encode64(padding: false)
  hashed = :crypto.hash(:sha256, token) |> Base.encode16()

  %ApiToken{}
  |> ApiToken.changeset(%{
    user_id: user.id,
    token_hash: hashed,
    expires_at: DateTime.utc_now() |> DateTime.add(90, :day)
  })
  |> Repo.insert()

  # Return unhashed token to user (only time it's visible)
  {:ok, token}
end

# Verify: hash the provided token and compare
def verify_api_token(token) do
  hashed = :crypto.hash(:sha256, token) |> Base.encode16()

  Repo.get_by(ApiToken, token_hash: hashed)
  |> case do
    %{expires_at: expires_at} = api_token ->
      if DateTime.compare(expires_at, DateTime.utc_now()) == :gt do
        {:ok, api_token}
      else
        {:error, :expired}
      end
    nil -> {:error, :invalid}
  end
end
```

**Token Rules:**
- Store only hashed tokens in the database
- Set expiration on all tokens
- Allow users to revoke tokens
- Use separate tokens for different scopes
- Never log tokens

## Authorization

### Role-Based Access Control (RBAC)

```elixir
defmodule MyApp.Authorization do
  @roles_hierarchy %{
    admin: [:admin, :manager, :member],
    manager: [:manager, :member],
    member: [:member]
  }

  def authorize(user, required_role) do
    allowed_roles = Map.get(@roles_hierarchy, user.role, [])
    if required_role in allowed_roles do
      :ok
    else
      {:error, :unauthorized}
    end
  end
end

# In a plug
defmodule MyAppWeb.Plugs.RequireRole do
  import Plug.Conn

  def init(role), do: role

  def call(conn, role) do
    case MyApp.Authorization.authorize(conn.assigns.current_user, role) do
      :ok -> conn
      {:error, :unauthorized} ->
        conn
        |> put_status(403)
        |> Phoenix.Controller.put_view(MyAppWeb.ErrorJSON)
        |> Phoenix.Controller.render("403.json")
        |> halt()
    end
  end
end
```

### Resource-Level Authorization

```elixir
# Always check ownership — never trust user input for resource access
def show(conn, %{"id" => id}) do
  project = Projects.get_project!(id)

  if project.organization_id == conn.assigns.current_user.organization_id do
    render(conn, :show, project: project)
  else
    conn |> put_status(404) |> render("404.json")  # 404, not 403 (don't leak existence)
  end
end
```

**Authorization Rules:**
- Check authorization on every request (not just UI hiding)
- Return 404 for unauthorized resources (don't leak existence)
- Scope all database queries to the current user/org
- Never pass user IDs in hidden form fields — use session

## API Security

### Rate Limiting

```elixir
# Different limits for different endpoints
defmodule MyAppWeb.RateLimiter do
  use Plug.Builder

  plug :rate_limit

  defp rate_limit(conn, _opts) do
    key = rate_limit_key(conn)
    limit = rate_limit_for(conn.request_path)

    case Hammer.check_rate(key, 60_000, limit) do
      {:allow, count} ->
        conn
        |> put_resp_header("x-ratelimit-limit", to_string(limit))
        |> put_resp_header("x-ratelimit-remaining", to_string(limit - count))
      {:deny, _} ->
        conn
        |> put_status(429)
        |> put_resp_header("retry-after", "60")
        |> send_resp(429, "Too many requests")
        |> halt()
    end
  end

  defp rate_limit_for("/api/auth" <> _), do: 10   # Auth: 10/min
  defp rate_limit_for("/api/" <> _), do: 100       # API: 100/min
  defp rate_limit_for(_), do: 200                   # Default: 200/min

  defp rate_limit_key(conn) do
    case conn.assigns[:current_user] do
      %{id: id} -> "api:user:#{id}"
      nil -> "api:ip:#{conn.remote_ip |> :inet.ntoa() |> to_string()}"
    end
  end
end
```

### Input Validation

```elixir
# Validate and sanitize ALL input
defmodule MyApp.Params do
  use Ecto.Schema
  import Ecto.Changeset

  @primary_key false
  embedded_schema do
    field :page, :integer, default: 1
    field :per_page, :integer, default: 20
    field :search, :string
    field :sort_by, Ecto.Enum, values: [:name, :created_at, :updated_at]
    field :sort_order, Ecto.Enum, values: [:asc, :desc], default: :desc
  end

  def validate(params) do
    %__MODULE__{}
    |> cast(params, [:page, :per_page, :search, :sort_by, :sort_order])
    |> validate_number(:page, greater_than: 0)
    |> validate_number(:per_page, greater_than: 0, less_than_or_equal_to: 100)
    |> validate_length(:search, max: 200)
    |> apply_action(:validate)
  end
end
```

**Input Rules:**
- Validate type, length, format, and range on ALL inputs
- Use allowlists, not blocklists
- Parameterize all database queries (Ecto does this by default)
- Sanitize HTML output (Phoenix does this by default with `<%= %>`)
- Reject unexpected fields

### CORS Configuration

```elixir
# In endpoint.ex or a plug
plug Corsica,
  origins: ["https://app.myapp.com", "https://myapp.com"],
  allow_methods: ["GET", "POST", "PUT", "PATCH", "DELETE"],
  allow_headers: ["authorization", "content-type"],
  allow_credentials: true,
  max_age: 86_400
```

**Never use `origins: "*"` with credentials.**

## Security Headers

```elixir
# In your endpoint or a plug
defmodule MyAppWeb.SecurityHeaders do
  import Plug.Conn

  def init(opts), do: opts

  def call(conn, _opts) do
    conn
    |> put_resp_header("strict-transport-security", "max-age=63072000; includeSubDomains; preload")
    |> put_resp_header("x-content-type-options", "nosniff")
    |> put_resp_header("x-frame-options", "DENY")
    |> put_resp_header("x-xss-protection", "0")  # Disabled — use CSP instead
    |> put_resp_header("referrer-policy", "strict-origin-when-cross-origin")
    |> put_resp_header("permissions-policy", "camera=(), microphone=(), geolocation=()")
    |> put_resp_header("content-security-policy", csp_header())
  end

  defp csp_header do
    "default-src 'self'; " <>
    "script-src 'self'; " <>
    "style-src 'self' 'unsafe-inline'; " <>
    "img-src 'self' data: https:; " <>
    "font-src 'self'; " <>
    "connect-src 'self' https://api.stripe.com; " <>
    "frame-ancestors 'none';"
  end
end
```

## Account Takeover Prevention

### Brute Force Protection

```elixir
defmodule MyApp.LoginThrottle do
  def check_login_allowed(email, ip) do
    ip_string = ip |> :inet.ntoa() |> to_string()

    with {:allow, _} <- Hammer.check_rate("login:ip:#{ip_string}", 60_000, 20),
         {:allow, _} <- Hammer.check_rate("login:email:#{email}", 300_000, 5) do
      :ok
    else
      {:deny, _} -> {:error, :rate_limited}
    end
  end

  # After failed login
  def record_failed_attempt(email, ip) do
    # Log for monitoring
    Logger.warning("Failed login attempt",
      email: email,
      ip: ip |> :inet.ntoa() |> to_string()
    )
  end

  # After successful login from new device/location
  def notify_new_login(user, conn) do
    ip = conn.remote_ip |> :inet.ntoa() |> to_string()
    ua = Plug.Conn.get_req_header(conn, "user-agent") |> List.first()

    MyApp.Mailer.deliver_login_notification(user, %{
      ip: ip,
      user_agent: ua,
      time: DateTime.utc_now()
    })
  end
end
```

### Sensitive Action Protection

```elixir
# Require password re-entry for sensitive actions
def require_sudo(conn, _opts) do
  last_auth = get_session(conn, :sudo_confirmed_at)

  if last_auth && DateTime.diff(DateTime.utc_now(), last_auth, :minute) < 15 do
    conn
  else
    conn
    |> put_flash(:info, "Please confirm your password to continue.")
    |> redirect(to: ~p"/confirm-password?return_to=#{conn.request_path}")
    |> halt()
  end
end
```

**Require re-authentication for:**
- Password changes
- Email changes
- MFA setup/removal
- API token creation
- Account deletion
- Billing changes
- Admin actions

## Data Protection

### Encryption at Rest

```elixir
# Encrypt sensitive fields using Cloak
defmodule MyApp.Encrypted.Binary do
  use Cloak.Ecto.Binary, vault: MyApp.Vault
end

defmodule MyApp.User do
  use Ecto.Schema

  schema "users" do
    field :email, :string
    field :ssn, MyApp.Encrypted.Binary  # Encrypted at rest
    field :ssn_hash, Cloak.Ecto.SHA256  # For lookups
  end
end
```

### Audit Logging

```elixir
defmodule MyApp.AuditLog do
  use Ecto.Schema

  schema "audit_logs" do
    field :user_id, :integer
    field :action, :string
    field :resource_type, :string
    field :resource_id, :integer
    field :metadata, :map
    field :ip_address, :string
    timestamps(updated_at: false)
  end

  def log(user, action, resource, metadata \\ %{}, conn \\ nil) do
    %__MODULE__{}
    |> Ecto.Changeset.change(%{
      user_id: user.id,
      action: action,
      resource_type: resource.__struct__ |> to_string(),
      resource_id: resource.id,
      metadata: metadata,
      ip_address: if(conn, do: conn.remote_ip |> :inet.ntoa() |> to_string())
    })
    |> Repo.insert()
  end
end

# Usage
AuditLog.log(current_user, "updated", project, %{changes: changeset.changes}, conn)
```

**Log these events:**
- Login/logout (success and failure)
- Password changes
- Permission changes
- Data exports
- Admin actions
- API token creation/revocation
- Billing changes

## Security Checklist

### Authentication
- [ ] Passwords hashed with bcrypt (cost 12+)
- [ ] Minimum 12 character passwords
- [ ] Breached password checking
- [ ] MFA available (TOTP or WebAuthn)
- [ ] Session rotation on login
- [ ] Session invalidation on password change
- [ ] Secure cookie settings (HttpOnly, Secure, SameSite)

### Authorization
- [ ] Server-side authorization on every request
- [ ] Resource-level access checks (not just role checks)
- [ ] 404 for unauthorized resources (don't leak existence)
- [ ] Database queries scoped to current user/org

### API
- [ ] Rate limiting on all endpoints
- [ ] Input validation on all parameters
- [ ] CORS properly configured
- [ ] API tokens hashed in database
- [ ] Token expiration enforced

### Headers & Transport
- [ ] HTTPS everywhere (HSTS enabled)
- [ ] Security headers configured
- [ ] CSP header set
- [ ] No sensitive data in URLs

### Monitoring
- [ ] Failed login attempts logged
- [ ] Audit trail for sensitive actions
- [ ] Alerting on anomalous patterns
- [ ] Regular dependency vulnerability scanning

## Common Mistakes

- **Storing plaintext passwords** — always hash with bcrypt/argon2
- **JWT without expiration** — always set `exp` claim
- **Trusting client-side authorization** — always verify server-side
- **Logging sensitive data** — never log passwords, tokens, PII
- **Using `origins: "*"` with credentials** — CORS misconfiguration
- **No rate limiting on auth endpoints** — enables brute force
- **Sequential/predictable IDs in URLs** — use UUIDs for public-facing resources
- **Not invalidating sessions on password change** — allows persistent access after compromise

## Related Skills

- **spam-prevention**: Signup abuse, bot detection, disposable email blocking
- **stripe-integration**: Payment security, PCI compliance
- **postgresql-table-design**: Row-level security, encryption at rest
- **elixir-phoenix**: Phoenix-specific security patterns
