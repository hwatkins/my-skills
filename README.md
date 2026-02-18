# My Skills

A collection of AI agent skills for the languages and frameworks I work with.

## Skills Included

### Elixir

| Skill | Description |
|-------|-------------|
| `elixir-phoenix` | Core Elixir/Phoenix development — functional patterns, Phoenix 1.7/1.8 conventions, typespecs, static analysis |
| `elixir-liveview` | Phoenix LiveView patterns for real-time applications — lifecycle, streams, events, PubSub, forms, performance |
| `elixir-ecto` | Ecto deep-dive — changesets, Multi, composable queries, migrations, optimistic locking, multi-tenancy |
| `elixir-otp` | OTP patterns — GenServer, Agent, Task, ETS, supervision trees, Registry, Oban, when NOT to use processes |
| `elixir-tdd` | TDD enforcement for Elixir — failing tests first, Mox for mocking, StreamData for property-based testing |
| `api-design` | REST API design for Phoenix — resource routing, error handling, pagination, auth, idempotency, webhooks, OpenAPI |
| `liveview-js-interop` | LiveView JS interop — JS commands, hooks, colocated hooks (LV 1.1+), DOM patching, third-party libs, LiveSvelte boundary |
| `realtime-ux` | UX patterns for real-time interfaces — optimistic UI, presence, conflict resolution, disconnect handling, loading states |

### Frontend

| Skill | Description |
|-------|-------------|
| `frontend-typescript` | TypeScript type safety — generics, utility types, discriminated unions, strict patterns |
| `frontend-tailwind` | Tailwind CSS — utility-first styling, responsive design, component patterns, dark mode |
| `frontend-design` | Distinctive, production-grade UI design — bold aesthetics, typography, color, motion, anti-AI-slop patterns |
| `component-design-system` | Unified component library — Phoenix + Svelte components sharing variant APIs, slot patterns, design tokens |
| `animation-transitions` | Coordinating LiveView JS transitions, Svelte transitions, and CSS animations — page transitions, reduced motion |

### Svelte

| Skill | Description |
|-------|-------------|
| `svelte-core` | Svelte 5 inside Phoenix LiveView via LiveSvelte — props, live.pushEvent, SSR, slots, end-to-end reactivity |

### Rust

| Skill | Description |
|-------|-------------|
| `rust-core` | Core Rust development — ownership, borrowing, lifetimes, traits, error handling, and idiomatic patterns |
| `rust-async` | Async Rust with Tokio — futures, concurrency, channels, streams, and performance |
| `rust-tdd` | Test-driven development enforcement for Rust — requires failing tests before implementation |

### 6502 Assembly

| Skill | Description |
|-------|-------------|
| `6502-assembly` | MOS 6502 assembly language — instruction set, addressing modes, algorithms, optimization, platform-specific patterns (NES, C64, Apple II, Atari, BBC Micro) |

### Database

| Skill | Description |
|-------|-------------|
| `postgresql-table-design` | PostgreSQL schema design — data types, indexing, constraints, partitioning, JSONB, performance patterns |
| `sql-optimization-patterns` | SQL query optimization — EXPLAIN analysis, indexing strategies, N+1 elimination, pagination, batch operations |

### Marketing / SaaS

| Skill | Description |
|-------|-------------|
| `email-sequence` | Email sequence design — drip campaigns, welcome sequences, onboarding, re-engagement, lifecycle emails |
| `referral-program` | Referral & affiliate program design — incentive structures, viral loops, optimization, launch checklists |
| `seo-audit` | SEO audit framework — technical SEO, on-page optimization, content quality, Core Web Vitals, E-E-A-T |
| `stripe-integration` | Stripe payment processing — checkout, subscriptions, webhooks, refunds, customer management, PCI compliance |
| `spam-prevention` | Spam signup defense — honeypots, disposable email blocking, rate limiting, progressive CAPTCHA, behavioral detection |
| `saas-security` | SaaS security — auth, MFA, session management, API protection, account takeover prevention, security headers |

### Engineering Practices

| Skill | Description |
|-------|-------------|
| `systematic-debugging` | Root-cause-first debugging — 4-phase process: investigate, pattern analysis, hypothesis testing, implementation |
| `test-driven-development` | Language-agnostic TDD discipline — the Iron Law, red-green-refactor, rationalizations, verification checklist |

## Installation

### Option 1: Using the Skills CLI (Recommended)

```bash
# Install all skills
npx skills add hwatkins/my-skills

# Install a single skill
npx skills add hwatkins/my-skills --skill elixir-phoenix

# Install multiple skills at once (e.g., all Elixir skills)
npx skills add hwatkins/my-skills --skill elixir-phoenix --skill elixir-liveview --skill elixir-ecto --skill elixir-otp --skill elixir-tdd --skill api-design

# Install globally (available in all projects)
npx skills add hwatkins/my-skills -g
```

**No npx?** Use curl to grab individual skill files:

```bash
mkdir -p .claude/skills
curl -sL https://raw.githubusercontent.com/hwatkins/my-skills/main/skills/elixir-phoenix/SKILL.md \
  -o .claude/skills/elixir-phoenix.md
```

### Option 2: Manual Installation (Claude Code)

**Project-local (recommended for team projects):**

```bash
# Copy specific skills into your project
mkdir -p .claude/skills

# Example: copy all Elixir skills
cp -r skills/elixir-phoenix .claude/skills/
cp -r skills/elixir-liveview .claude/skills/
cp -r skills/elixir-ecto .claude/skills/
cp -r skills/elixir-otp .claude/skills/
cp -r skills/elixir-tdd .claude/skills/
cp -r skills/api-design .claude/skills/

# Example: copy Rust + Svelte skills for a specific project
cp -r skills/rust-core .claude/skills/
cp -r skills/rust-async .claude/skills/
cp -r skills/svelte-core .claude/skills/
```

**Global (available in all projects):**

```bash
# Copy all skills to global directory
mkdir -p ~/.claude/skills
cp -r skills/* ~/.claude/skills/
```

**Using symlinks (for easy updates):**

```bash
# Clone the repo somewhere permanent
git clone git@github.com:hwatkins/my-skills.git ~/my-skills

# Symlink individual skills
mkdir -p ~/.claude/skills
ln -s ~/my-skills/skills/elixir-phoenix ~/.claude/skills/elixir-phoenix
ln -s ~/my-skills/skills/rust-core ~/.claude/skills/rust-core
# ... add more as needed

# Or symlink everything
for skill in ~/my-skills/skills/*/; do
  ln -s "$skill" ~/.claude/skills/$(basename "$skill")
done
```

### Option 3: Other Agents

| Agent | Project Location | Global Location |
|-------|------------------|-----------------|
| Claude Code | `.claude/skills/` | `~/.claude/skills/` |
| GitHub Copilot | `.github/skills/` | `~/.copilot/skills/` |
| Cursor | `.cursor/skills/` | `~/.cursor/skills/` |
| OpenCode | `.opencode/skills/` | `~/.opencode/skills/` |

## Updating

If you used symlinks:

```bash
cd ~/my-skills
git pull
```

If you copied files, re-copy after pulling updates.

## Usage

Once installed, the skills are automatically available. The AI agent will use them when relevant based on the skill descriptions.

You can also explicitly request a skill:

- "Use the elixir-liveview skill to help me build this feature"
- "Following TDD practices, implement this function"
- "Use the rust-async skill for this concurrent service"
- "Build this page with svelte-kit and frontend-tailwind skills"

## Customization

Feel free to fork this repo and customize the skills for your team's conventions:

1. Fork the repository
2. Edit the SKILL.md files
3. Update the remote URL in your local clone
4. Re-install or pull updates

## Skill Details

### Elixir

#### elixir-phoenix

Core Elixir and Phoenix development patterns:

- Functional core, imperative shell pattern
- Naming conventions (snake_case, PascalCase, verb-first functions)
- Typespecs (`@spec`) and documentation (`@doc`) on all public functions
- Static analysis (mix format, Credo, Dialyzer)
- Phoenix 1.7+ patterns (function components, verified routes, embed_templates)
- Phoenix 1.8 changes (Scopes, `broadcast_from`)
- HEEx template best practices
- HTTP clients with Req and behaviour-based testability
- API/implementation separation for complex contexts
- Error handling, security, and performance

#### elixir-liveview

Comprehensive LiveView patterns:

- Lifecycle management (mount, handle_params, handle_async)
- Streams for efficient list rendering
- Event handling and validation
- PubSub and real-time updates
- Form handling with changesets
- Performance optimization (temporary_assigns, debounce, throttle)
- Common mistakes and how to avoid them

#### elixir-ecto

Ecto data layer patterns:

- Changesets for validation (including non-DB embedded schemas)
- Railway-oriented programming with `with` chains
- `Ecto.Multi` for atomic transactions
- Optimistic locking for concurrent updates
- Composable and dynamic queries
- Preloading strategies (avoiding N+1)
- Migration best practices
- Multi-tenancy (schema prefix and foreign key approaches)
- Upserts and common mistakes

#### elixir-otp

OTP process design and concurrency:

- When to use GenServer vs Agent vs Task vs ETS vs Oban (decision framework)
- **Database is source of truth** — don't use GenServers for domain entities
- GenServer best practices (handle_continue, avoiding bottlenecks)
- Task.Supervisor for fault-tolerant concurrent work
- ETS for read-heavy caches
- Registry and DynamicSupervisor for dynamic processes
- Supervision tree design (one-for-one, one-for-all, rest-for-one)
- Oban for reliable background jobs
- When NOT to use processes

#### elixir-tdd

Strict test-driven development enforcement:

- The TDD cycle (Red → Green → Refactor)
- Mandatory test cases (happy path, validation, edge cases, auth, state transitions)
- LiveView testing patterns
- Test organization and fixtures
- Mox for mocking external dependencies (behaviour-based)
- StreamData for property-based testing
- What NOT to do (with examples)
- Pre-implementation checklist

#### api-design

REST API design for Phoenix:

- Resource design (URL structure, HTTP methods, naming conventions)
- Phoenix patterns (controllers, FallbackController, JSON views, router scoping)
- Consistent error responses (format, status codes, changeset errors)
- Pagination (cursor-based preferred, offset for simple cases)
- Filtering, sorting, and query parameter validation
- Authentication (Bearer tokens, scoped access)
- Idempotency keys for safe POST retries
- Outbound webhooks (event design, HMAC signing, Oban retry delivery)
- API versioning (URL-based, deprecation strategy)
- Rate limiting with response headers
- OpenAPI documentation with open_api_spex

#### liveview-js-interop

LiveView JavaScript interoperability:

- Decision framework: JS commands → colocated hooks → traditional hooks → LiveSvelte
- JS commands deep dive (show/hide, transitions, push, dispatch, exec, composing, selectors)
- Hook lifecycle (mounted, beforeUpdate, updated, destroyed, disconnected, reconnected)
- DOM patching survival (why JS state gets clobbered, 6 solutions ranked)
- Server ↔ client communication (pushEvent, handleEvent, reply callbacks, namespacing)
- Colocated hooks (LV 1.1+ / Phoenix 1.8+) with setup and esbuild config
- Third-party library integration (maps, charts, rich text editors, drag-and-drop)
- Hook ↔ LiveSvelte boundary (when to use which)
- Common use cases (clipboard, infinite scroll, local storage, keyboard shortcuts, resize observers)
- Anti-patterns (missing ids, no cleanup, DOM mutation without ignore, business logic in hooks)
- Testing (render_hook, assert_push_event, browser debugging)

#### realtime-ux

UX patterns for real-time interfaces:

- Optimistic UI (CSS loading states, JS commands for instant feedback, optimistic assigns with rollback, optimistic streams)
- Presence indicators (Phoenix.Presence in LiveView, avatar stacks, idle detection, cursor presence)
- Conflict resolution (optimistic locking, field-level merging, real-time broadcasting, pessimistic locking)
- Disconnect & recovery (reconnection mechanics, delayed/extended banners, disabling actions while offline, form auto-recovery, `phx-auto-recover`, client-side backup)
- State recovery strategies (URL-based state via `push_patch`, server-side draft saving, scroll restore hooks)
- Deployment handling (`static_changed?`, `phx-track-static`, zero-downtime deploy checklist)
- Loading and skeleton states (skeleton screens, async assigns, inline loading indicators, progress bars)
- Real-time feedback (toast notifications, live counters, typing indicators, data update highlights)
- Anti-patterns (silent disconnect failures, optimistic UI for irreversible actions, ignoring concurrent edits)

### Frontend

#### frontend-typescript

TypeScript type safety patterns for frontend development.

#### frontend-tailwind

Tailwind CSS utility-first styling patterns.

#### frontend-design

Distinctive, production-grade UI design principles.

#### component-design-system

Unified component library architecture:

- Design token system (Tailwind config as token layer, CSS custom properties for runtime theming)
- Phoenix function component patterns (variant + size API, icon buttons, named slots, table with slot attributes, badge, form input)
- Svelte component patterns (mirroring variant API, Svelte 5 snippets, generic data table)
- Cross-boundary consistency (shared vocabulary table, matching variant values, shared class maps)
- File organization (where Phoenix components vs Svelte components live)
- When Phoenix vs when Svelte (decision table)
- Anti-patterns (duplicating components, inconsistent variants, hardcoded colors, skipping :global)

#### animation-transitions

Coordinating three animation systems:

- Decision framework: CSS → JS commands → Svelte transitions → Spring/Tween
- CSS transitions and keyframes (interactive states, Tailwind animation utilities)
- LiveView JS command transitions (3-tuple syntax, modal/flash/dropdown patterns, highlight on update)
- Svelte transitions (built-in: fade/fly/slide/scale/blur/crossfade, local vs global, custom functions)
- Svelte motion (Spring for physics-based, Tween for interpolated values)
- Page transitions in LiveView navigation (CSS on mount, navigation events, View Transitions API, Svelte page transitions)
- Coordinating systems together (LiveView modal with Svelte content, server-triggered Svelte animation)
- Reduced motion (CSS, Svelte prefersReducedMotion, Tailwind motion-reduce)
- Performance (composite properties only, CSS mode over tick, stagger animations)

### Rust

#### rust-core

Core Rust development patterns:

- Ownership, borrowing, and lifetimes
- Error handling with `thiserror` and `anyhow`
- Traits and generics
- Pattern matching and destructuring
- Iterators and functional patterns
- Module organization and project structure
- Common crates reference

#### rust-async

Async Rust with Tokio:

- Runtime setup and task spawning
- Channels (mpsc, broadcast, oneshot, watch)
- Select and timeouts
- Shared state (Arc, Mutex, RwLock)
- Graceful shutdown patterns
- Streams and backpressure
- Common async mistakes and performance tips

#### rust-tdd

Strict test-driven development for Rust:

- The TDD cycle (Red → Green → Refactor)
- Test organization (`#[cfg(test)]`, integration tests)
- Mandatory test cases (happy path, errors, edge cases, type safety)
- Async testing with `#[tokio::test]`
- Property-based testing with `proptest`
- Parameterized tests with `rstest`

### Svelte

#### svelte-core (LiveSvelte)

Svelte 5 inside Phoenix LiveView via LiveSvelte:

- End-to-end reactivity (server owns state, props flow over websocket)
- `live.pushEvent` / `live.handleEvent` for LiveView communication
- The `~V` sigil for inline Svelte in LiveViews
- Components macro for JSX-like HEEx usage
- SSR with hydration (and when to disable it)
- Slots from LiveView into Svelte
- `live_json` for optimized large data transfer
- Structs/Ecto serialization
- Secret state caveat (JSON vs HTML over the wire)
- Svelte 5 runes (`$state`, `$derived`, `$effect`)

### Database

#### postgresql-table-design

PostgreSQL schema design and best practices:

- Data types (BIGINT identity, TEXT, TIMESTAMPTZ, NUMERIC, JSONB, arrays, ranges, vectors)
- Forbidden types (serial, varchar, money, timestamp without tz)
- Indexing strategies (B-tree, GIN, GiST, BRIN, partial, expression, covering)
- Constraints (PK, FK with manual indexes, UNIQUE NULLS NOT DISTINCT, CHECK, EXCLUDE)
- Partitioning (range, list, hash, TimescaleDB)
- JSONB guidance (GIN indexes, jsonb_path_ops, generated columns)
- Performance patterns (update-heavy, insert-heavy, upsert-friendly)
- Safe schema evolution (transactional DDL, concurrent indexes)
- Row-level security, extensions, generated columns

#### sql-optimization-patterns

SQL query optimization and performance:

- EXPLAIN plan analysis (Seq Scan, Index Scan, join methods, cost metrics)
- Index strategies (B-tree, GIN, GiST, BRIN, partial, expression, covering)
- N+1 query elimination (JOINs, batch loading)
- Cursor-based pagination (replacing slow OFFSET)
- Aggregate optimization (COUNT, GROUP BY, covering indexes)
- Subquery optimization (correlated → JOIN, CTEs, window functions)
- Batch operations (INSERT, UPDATE, COPY)
- Materialized views and partitioning
- Monitoring queries (pg_stat_statements, missing/unused indexes)
- Common pitfalls and best practices

### Marketing / SaaS

#### email-sequence

Email sequence and drip campaign design:

- Sequence types (welcome, lead nurture, re-engagement, onboarding, post-purchase)
- Timing and delay strategy
- Subject line and preview text patterns
- Email copy structure (hook, context, value, CTA)
- Email types by category (onboarding, retention, billing, win-back, campaigns)
- Output format templates for sequence planning
- CTA guidelines and formatting best practices

#### referral-program

Referral and affiliate program design:

- Referral vs affiliate program selection
- The referral loop (trigger → share → convert → reward)
- Incentive structures (single-sided, double-sided, tiered)
- Share mechanism design and trigger moment identification
- Program optimization (A/B tests, common problems & fixes)
- Key metrics (referral rate, conversion, LTV, CAC comparison)
- Launch checklist (before, during, post-launch)
- Referral email sequences

#### seo-audit

SEO audit and diagnosis:

- Audit framework (crawlability, indexation, technical, on-page, content, authority)
- Technical SEO (robots.txt, sitemaps, site architecture, crawl budget)
- Core Web Vitals (LCP, INP, CLS targets and speed factors)
- On-page optimization (titles, meta descriptions, headings, internal linking)
- Content quality assessment (E-E-A-T signals, depth, engagement)
- Common issues by site type (SaaS, e-commerce, blog, local)
- Audit report output format with prioritized action plans

#### stripe-integration

Stripe payment processing integration:

- Payment flows (hosted checkout, Payment Intents, Setup Intents)
- Subscription billing (products, prices, invoices, customer portal)
- Webhook handling (signature verification, idempotency, critical events)
- Customer management (payment methods, metadata)
- Refunds and dispute handling
- Testing with test cards and test mode
- Best practices (PCI compliance, SCA, error handling)

#### spam-prevention

Spam signup and bot account defense:

- Layered defense strategy (invisible first, friction only when needed)
- Honeypot fields and time-based detection
- Disposable email domain blocking and MX validation
- IP and fingerprint-based rate limiting
- Progressive CAPTCHA (Cloudflare Turnstile, hCaptcha, ALTCHA)
- Post-signup behavioral scoring
- Automated cleanup of unconfirmed accounts
- Implementation checklist (minimum, enhanced, advanced)

#### saas-security

SaaS application security best practices:

- Password security (bcrypt, breached password checking, NIST guidelines)
- MFA (TOTP, WebAuthn/passkeys, recovery codes)
- Session management (rotation, invalidation, secure cookies)
- API token security (hashed storage, expiration, revocation)
- Authorization (RBAC, resource-level checks, scoped queries)
- API rate limiting and input validation
- Security headers (HSTS, CSP, CORS)
- Account takeover prevention (brute force, sudo mode)
- Data protection (encryption at rest, audit logging)

### Engineering Practices

#### systematic-debugging

Root-cause-first debugging methodology:

- The Iron Law: no fixes without root cause investigation first
- Phase 1: Root cause investigation (error reading, reproduction, evidence gathering)
- Phase 2: Pattern analysis (find working examples, compare differences)
- Phase 3: Hypothesis and testing (scientific method, minimal changes)
- Phase 4: Implementation (failing test, single fix, verify)
- 3+ failed fixes = question the architecture
- Red flags and common rationalizations
- Multi-component diagnostic instrumentation

#### test-driven-development

Language-agnostic TDD methodology and discipline:

- The Iron Law: no production code without a failing test first
- Red-Green-Refactor cycle with mandatory verification steps
- Why order matters (tests-first vs tests-after)
- Common rationalizations and rebuttals
- Red flags that mean "stop and start over"
- Bug fix workflow (failing test → fix → verify)
- Verification checklist before marking work complete
- When stuck: simplify design, not tests

## License

MIT License - See [LICENSE](LICENSE) for details.
