---
name: frontend-typescript
description: Expert TypeScript development for frontend applications. Covers type safety, patterns, generics, utility types, and best practices. Use for any TypeScript code.
---

# TypeScript Development

You are an expert in TypeScript with deep knowledge of type safety, modern patterns, and frontend development.

## Core Principles

- Leverage the type system to catch errors at compile time
- Prefer strict mode (`"strict": true` in tsconfig)
- Use types to make impossible states impossible
- Avoid `any` — use `unknown` when the type is truly unknown
- Write self-documenting code with descriptive types

## Type Fundamentals

### Prefer Interfaces for Objects, Types for Unions/Intersections

```typescript
// ✅ Good: Interface for object shapes
interface User {
  id: string;
  email: string;
  name: string;
  role: UserRole;
}

// ✅ Good: Type for unions and computed types
type UserRole = "admin" | "editor" | "viewer";
type UserWithPosts = User & { posts: Post[] };

// ✅ Good: Type for function signatures
type EventHandler<T> = (event: T) => void;
```

### Use Discriminated Unions for State

```typescript
// ✅ Good: Discriminated union — impossible states are impossible
type AsyncState<T> =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; error: Error };

function renderState<T>(state: AsyncState<T>) {
  switch (state.status) {
    case "idle":
      return null;
    case "loading":
      return <Spinner />;
    case "success":
      return <Data data={state.data} />;
    case "error":
      return <ErrorMessage error={state.error} />;
  }
}

// ❌ Bad: Separate booleans — allows impossible states
interface BadState {
  isLoading: boolean;
  isError: boolean;
  data?: User;
  error?: Error;
  // Can isLoading AND isError both be true? Unclear.
}
```

## Generics

- Use generics for reusable, type-safe abstractions
- Constrain generics with `extends` when needed
- Use meaningful names (`T` for type, `K` for key, `V` for value)

```typescript
// ✅ Good: Constrained generic
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

// ✅ Good: Generic with default
interface ApiResponse<T = unknown> {
  data: T;
  status: number;
  message: string;
}

// ✅ Good: Generic component props
interface ListProps<T> {
  items: T[];
  renderItem: (item: T) => React.ReactNode;
  keyExtractor: (item: T) => string;
}
```

## Utility Types

Use built-in utility types instead of reinventing them:

```typescript
// Partial — all properties optional
type UpdateUser = Partial<User>;

// Pick — select specific properties
type UserPreview = Pick<User, "id" | "name">;

// Omit — exclude properties
type CreateUser = Omit<User, "id" | "createdAt">;

// Record — typed key-value map
type RolePermissions = Record<UserRole, Permission[]>;

// Required — make all properties required
type CompleteUser = Required<User>;

// Extract / Exclude — filter union types
type ActiveStatus = Extract<Status, "active" | "pending">;
```

## Strict Null Handling

- Enable `strictNullChecks` (included in `strict: true`)
- Use optional chaining (`?.`) and nullish coalescing (`??`)
- Narrow types with type guards

```typescript
// ✅ Good: Type narrowing
function processUser(user: User | null) {
  if (!user) {
    return;
  }
  // TypeScript knows user is User here
  console.log(user.name);
}

// ✅ Good: Custom type guard
function isAdmin(user: User): user is AdminUser {
  return user.role === "admin";
}

// ✅ Good: Nullish coalescing
const displayName = user.name ?? "Anonymous";
const port = config.port ?? 3000;

// ❌ Bad: Non-null assertion without checking
const name = user!.name; // Dangerous
```

## Async Patterns

```typescript
// ✅ Good: Typed async functions
async function fetchUser(id: string): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  if (!response.ok) {
    throw new ApiError(response.status, await response.text());
  }
  return response.json() as Promise<User>;
}

// ✅ Good: Error handling with Result type
type Result<T, E = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E };

async function safelyFetchUser(id: string): Promise<Result<User>> {
  try {
    const user = await fetchUser(id);
    return { ok: true, value: user };
  } catch (error) {
    return { ok: false, error: error as Error };
  }
}
```

## Enums vs Const Objects

Prefer `as const` objects over enums for better tree-shaking and type inference:

```typescript
// ✅ Preferred: const object
const Status = {
  Active: "active",
  Inactive: "inactive",
  Suspended: "suspended",
} as const;

type Status = (typeof Status)[keyof typeof Status];

// Also acceptable: string enum
enum Direction {
  Up = "UP",
  Down = "DOWN",
  Left = "LEFT",
  Right = "RIGHT",
}
```

## Module Organization

- One export per concept, co-locate related types
- Use barrel exports (`index.ts`) sparingly — they can hurt tree-shaking
- Export types separately with `export type` for better erasure

```typescript
// ✅ Good: Co-located types and implementation
// user.ts
export interface User {
  id: string;
  email: string;
}

export type CreateUserInput = Omit<User, "id">;

export function createUser(input: CreateUserInput): User {
  return { id: crypto.randomUUID(), ...input };
}
```

## Zod for Runtime Validation

Use Zod to bridge compile-time types with runtime validation:

```typescript
import { z } from "zod";

const UserSchema = z.object({
  id: z.string().uuid(),
  email: z.string().email(),
  name: z.string().min(1).max(100),
  role: z.enum(["admin", "editor", "viewer"]),
});

type User = z.infer<typeof UserSchema>;

function parseUser(data: unknown): User {
  return UserSchema.parse(data);
}
```

## Common Mistakes

```typescript
// ❌ Don't use `any`
function process(data: any) { ... }

// ✅ Use `unknown` and narrow
function process(data: unknown) {
  if (typeof data === "string") { ... }
}

// ❌ Don't use type assertions carelessly
const user = data as User; // No runtime check!

// ✅ Validate at boundaries
const user = UserSchema.parse(data);

// ❌ Don't use `object` type
function process(obj: object) { ... }

// ✅ Use specific types or Record
function process(obj: Record<string, unknown>) { ... }

// ❌ Don't ignore Promise rejections
fetchUser(id).then(setUser);

// ✅ Handle errors
fetchUser(id).then(setUser).catch(handleError);
// Or use async/await with try/catch
```
