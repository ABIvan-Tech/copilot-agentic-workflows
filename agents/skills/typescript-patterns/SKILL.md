# TypeScript Patterns

## Overview

TypeScript Patterns encompasses the comprehensive set of type system features, design patterns, and best practices that enable writing type-safe, maintainable, and scalable applications. This skill covers everything from fundamental type inference and type guards to advanced generic patterns, discriminated unions, and framework-specific typing strategies for React and Node.js. Mastering these patterns allows developers to leverage TypeScript's powerful type system to catch errors at compile time, improve code documentation, enable better IDE support, and create more robust applications.

This skill is essential for building production-grade TypeScript applications across any framework or runtime. It includes practical patterns for common scenarios like API data fetching, React component composition, Express middleware, error handling, and state management. The patterns emphasize type safety without sacrificing developer experience, showing how to write types that are both correct and maintainable.

## When to Use This Skill

- When designing type-safe APIs, components, or libraries in TypeScript
- When working with complex data structures that need precise typing
- When integrating TypeScript with React, Node.js, Express, or other frameworks
- When implementing type-safe state management or data fetching patterns
- When refactoring JavaScript code to TypeScript
- When code reviews reveal type safety issues or misuse of `any`
- When setting up new TypeScript projects with proper compiler configurations
- When teaching or documenting TypeScript best practices for teams
- When debugging type-related compilation errors
- When optimizing type inference to reduce explicit type annotations

---

## 1. Type System Fundamentals

### Type Inference vs Explicit Annotations

TypeScript's type inference is powerful. Use it when types are obvious; add explicit types when they improve clarity or catch errors.

✅ **Good: Let TypeScript infer obvious types**
```typescript
// Type is inferred as number
const count = 42;

// Type is inferred as string[]
const names = ["Alice", "Bob", "Charlie"];

// Return type inferred as number
function add(a: number, b: number) {
  return a + b;
}

// Type inferred from context
const doubled = [1, 2, 3].map(n => n * 2);
```

❌ **Bad: Redundant explicit types**
```typescript
// Unnecessary explicit types
const count: number = 42;
const names: string[] = ["Alice", "Bob", "Charlie"];

function add(a: number, b: number): number {
  return a + b; // Return type is obvious
}
```

✅ **Good: Explicit types where they add value**
```typescript
// Public API: document the contract
export function processUser(userData: UserInput): Promise<User> {
  return transformUser(userData);
}

// Prevent accidental type widening
const config: Config = {
  apiUrl: "https://api.example.com",
  timeout: 5000,
};

// Complex inference: help the compiler
type Result<T> = { success: true; data: T } | { success: false; error: string };

function fetchData<T>(url: string): Promise<Result<T>> {
  // Explicit return type prevents mistakes
  return fetch(url).then(res => {
    if (res.ok) {
      return res.json().then(data => ({ success: true, data }));
    }
    return { success: false, error: res.statusText };
  });
}
```

### Union & Intersection Types

Union types represent "OR" relationships, intersections represent "AND" relationships.

✅ **Good: Union types for alternatives**
```typescript
type Status = "pending" | "approved" | "rejected";

type Result = 
  | { success: true; data: User }
  | { success: false; error: Error };

function handleResult(result: Result) {
  if (result.success) {
    console.log(result.data.name); // Type narrowed to success case
  } else {
    console.error(result.error.message); // Type narrowed to error case
  }
}

// Union of different shapes
type ApiResponse = 
  | { type: "user"; user: User }
  | { type: "error"; message: string }
  | { type: "loading" };
```

❌ **Bad: Using any instead of unions**
```typescript
// Loses type safety
function handleResult(result: any) {
  if (result.success) {
    console.log(result.data.name); // No type checking!
  }
}

// Boolean flag instead of discriminated union
type Result = {
  success: boolean;
  data?: User;     // Optional creates confusion
  error?: Error;   // Which fields are present when?
};
```

✅ **Good: Intersection types for composition**
```typescript
type Timestamped = {
  createdAt: Date;
  updatedAt: Date;
};

type Identifiable = {
  id: string;
};

type User = Identifiable & Timestamped & {
  name: string;
  email: string;
};

// Combining multiple concerns
type AuditedEntity<T> = T & {
  auditLog: AuditEntry[];
  lastModifiedBy: string;
};

type AuditedUser = AuditedEntity<User>;
```

### Literal Types & Discriminated Unions

Literal types enable exact value matching. Discriminated unions use a common property to narrow types.

✅ **Good: Discriminated unions with type guards**
```typescript
type Shape = 
  | { kind: "circle"; radius: number }
  | { kind: "rectangle"; width: number; height: number }
  | { kind: "triangle"; base: number; height: number };

function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "rectangle":
      return shape.width * shape.height;
    case "triangle":
      return 0.5 * shape.base * shape.height;
    default:
      // Exhaustiveness check
      const _exhaustive: never = shape;
      throw new Error("Unknown shape");
  }
}

// Redux-style actions
type Action = 
  | { type: "INCREMENT"; payload: number }
  | { type: "DECREMENT"; payload: number }
  | { type: "RESET" };

function reducer(state: number, action: Action): number {
  switch (action.type) {
    case "INCREMENT":
      return state + action.payload; // payload is available
    case "DECREMENT":
      return state - action.payload;
    case "RESET":
      return 0; // no payload here
  }
}
```

❌ **Bad: Weak discrimination or missing discriminant**
```typescript
// No discriminant property
type Shape = {
  radius?: number;
  width?: number;
  height?: number;
};

function area(shape: Shape): number {
  // Difficult to determine which shape this is
  if (shape.radius) {
    return Math.PI * shape.radius ** 2;
  }
  // What if both radius and width are defined?
  return shape.width! * shape.height!; // Dangerous assertions
}
```

### Type Guards & Type Narrowing

Type guards help TypeScript narrow union types to specific types.

✅ **Good: Custom type guards**
```typescript
interface User {
  type: "user";
  id: string;
  name: string;
}

interface Guest {
  type: "guest";
  sessionId: string;
}

type Visitor = User | Guest;

// User-defined type guard
function isUser(visitor: Visitor): visitor is User {
  return visitor.type === "user";
}

function greet(visitor: Visitor) {
  if (isUser(visitor)) {
    console.log(`Hello, ${visitor.name}!`); // Type narrowed to User
  } else {
    console.log(`Hello, guest ${visitor.sessionId}!`);
  }
}

// Type guard for null checking
function assertDefined<T>(value: T | null | undefined, message?: string): asserts value is T {
  if (value === null || value === undefined) {
    throw new Error(message || "Value is null or undefined");
  }
}

const user = getUserById(id);
assertDefined(user, "User not found");
console.log(user.name); // TypeScript knows user is defined
```

✅ **Good: Built-in type guards**
```typescript
function processValue(value: string | number | null) {
  // typeof guard
  if (typeof value === "string") {
    return value.toUpperCase();
  }
  
  // typeof guard
  if (typeof value === "number") {
    return value.toFixed(2);
  }
  
  // Null check
  if (value === null) {
    return "N/A";
  }
}

// instanceof guard
class ApiError extends Error {
  constructor(public statusCode: number, message: string) {
    super(message);
  }
}

function handleError(error: unknown) {
  if (error instanceof ApiError) {
    console.log(`API Error ${error.statusCode}: ${error.message}`);
  } else if (error instanceof Error) {
    console.log(`Error: ${error.message}`);
  } else {
    console.log("Unknown error");
  }
}

// in operator guard
type Dog = { bark: () => void };
type Cat = { meow: () => void };

function makeSound(animal: Dog | Cat) {
  if ("bark" in animal) {
    animal.bark();
  } else {
    animal.meow();
  }
}
```

❌ **Bad: Not using type guards properly**
```typescript
function processValue(value: string | number | null) {
  // No type narrowing - error prone
  return value.toUpperCase(); // Error: might be number or null
}

// Weak type guard
function isUser(visitor: any): boolean {
  return visitor.type === "user"; // Returns boolean, doesn't narrow type
}
```

### Type Assertions

Use type assertions sparingly and only when you have information TypeScript doesn't.

✅ **Good: Justified type assertions**
```typescript
// Working with DOM
const canvas = document.getElementById("canvas") as HTMLCanvasElement;
const ctx = canvas.getContext("2d")!; // Non-null assertion when you know it exists

// After runtime validation
function parseConfig(json: string): Config {
  const parsed = JSON.parse(json);
  
  // Validate the structure
  if (!parsed.apiUrl || typeof parsed.timeout !== "number") {
    throw new Error("Invalid config");
  }
  
  return parsed as Config; // Safe after validation
}

// const assertions for readonly
const routes = [
  { path: "/", component: "Home" },
  { path: "/about", component: "About" },
] as const; // Make it readonly and literal

type Route = typeof routes[number]; // Extract the union type
```

❌ **Bad: Using assertions to bypass type system**
```typescript
// Hiding errors with assertions
function getUser(id: string): User {
  const response = fetch(`/api/users/${id}`);
  return response as any as User; // Double assertion - very dangerous!
}

// Assertion where validation is needed
function processData(data: unknown) {
  const user = data as User; // No runtime check!
  console.log(user.name); // Might crash
}

// Non-null assertion without verification
function findUser(id: string): User {
  const users = getUsers();
  return users.find(u => u.id === id)!; // Might be undefined!
}
```

---

## 2. Advanced Types & Generics

### Generic Functions & Constraints

Generics enable reusable, type-safe code that works with multiple types.

✅ **Good: Generic functions with constraints**
```typescript
// Basic generic function
function identity<T>(value: T): T {
  return value;
}

// Generic with constraint
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { name: "Alice", age: 30 };
const name = getProperty(user, "name"); // Type: string
const age = getProperty(user, "age");   // Type: number

// Multiple type parameters
function map<T, U>(array: T[], fn: (item: T) => U): U[] {
  return array.map(fn);
}

const numbers = [1, 2, 3];
const strings = map(numbers, n => n.toString()); // Type: string[]

// Generic with multiple constraints
function merge<T extends object, U extends object>(obj1: T, obj2: U): T & U {
  return { ...obj1, ...obj2 };
}

// Constraint using another generic
function sortBy<T, K extends keyof T>(
  array: T[],
  key: K
): T[] {
  return [...array].sort((a, b) => {
    const aVal = a[key];
    const bVal = b[key];
    return aVal < bVal ? -1 : aVal > bVal ? 1 : 0;
  });
}
```

❌ **Bad: Missing constraints or over-use of any**
```typescript
// Too permissive
function getProperty(obj: any, key: string): any {
  return obj[key]; // No type safety
}

// Missing constraint
function merge<T, U>(obj1: T, obj2: U): T & U {
  return { ...obj1, ...obj2 }; // Error: might not be objects
}
```

### Conditional Types

Conditional types enable type-level if/else logic.

✅ **Good: Conditional types for API responses**
```typescript
// Basic conditional type
type IsString<T> = T extends string ? true : false;

type A = IsString<string>;  // true
type B = IsString<number>;  // false

// Unwrap Promise types
type Awaited<T> = T extends Promise<infer U> ? U : T;

type C = Awaited<Promise<string>>;  // string
type D = Awaited<number>;           // number

// Extract function return type
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

type E = ReturnType<() => string>;      // string
type F = ReturnType<(x: number) => boolean>; // boolean

// Conditional type for nullable handling
type NonNullable<T> = T extends null | undefined ? never : T;

type G = NonNullable<string | null>;    // string
type H = NonNullable<number | undefined>; // number

// Distribution over unions
type ToArray<T> = T extends any ? T[] : never;

type I = ToArray<string | number>;  // string[] | number[]

// Practical: type-safe API wrapper
type ApiResponse<T> = {
  data: T;
  error: null;
  status: 200;
} | {
  data: null;
  error: string;
  status: number;
};

type ExtractData<T> = T extends { data: infer D } ? D : never;

async function fetchUser(): Promise<ApiResponse<User>> {
  // Implementation
  return {} as any;
}

type UserData = ExtractData<Awaited<ReturnType<typeof fetchUser>>>;
```

### Mapped Types

Mapped types transform existing types into new types.

✅ **Good: Utility types with mapped types**
```typescript
// Make all properties optional
type Partial<T> = {
  [P in keyof T]?: T[P];
};

// Make all properties required
type Required<T> = {
  [P in keyof T]-?: T[P]; // -? removes optionality
};

// Make all properties readonly
type Readonly<T> = {
  readonly [P in keyof T]: T[P];
};

// Pick specific properties
type Pick<T, K extends keyof T> = {
  [P in K]: T[P];
};

// Omit specific properties
type Omit<T, K extends keyof T> = Pick<T, Exclude<keyof T, K>>;

// Create a type with specific keys
type Record<K extends string | number | symbol, T> = {
  [P in K]: T;
};

// Practical examples
interface User {
  id: string;
  name: string;
  email: string;
  password: string;
  createdAt: Date;
}

// For updates, make fields optional
type UserUpdate = Partial<Pick<User, "name" | "email">>;

// For public display, omit sensitive fields
type PublicUser = Omit<User, "password">;

// For creation, omit generated fields
type UserCreate = Omit<User, "id" | "createdAt"> & {
  confirmPassword: string;
};

// Nullable versions
type Nullable<T> = {
  [P in keyof T]: T[P] | null;
};

type NullableUser = Nullable<User>;
```

✅ **Good: Advanced mapped types**
```typescript
// Deep partial
type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object ? DeepPartial<T[P]> : T[P];
};

// Type-safe property paths
type PathsToStringProps<T> = {
  [K in keyof T]: T[K] extends string ? K : never;
}[keyof T];

interface User {
  id: string;
  name: string;
  age: number;
  email: string;
}

type StringKeys = PathsToStringProps<User>; // "id" | "name" | "email"

// Getters type
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

type UserGetters = Getters<User>;
// {
//   getId: () => string;
//   getName: () => string;
//   getAge: () => number;
//   getEmail: () => string;
// }

// Mutable (remove readonly)
type Mutable<T> = {
  -readonly [P in keyof T]: T[P];
};
```

### Template Literal Types

Template literal types enable string manipulation at the type level.

✅ **Good: Type-safe string patterns**
```typescript
// Event system with type-safe names
type Events = "click" | "focus" | "blur";
type EventHandlers = `on${Capitalize<Events>}`;
// "onClick" | "onFocus" | "onBlur"

// CSS properties
type CSSProperty = "color" | "background-color" | "font-size";
type CSSValue<P extends CSSProperty> = 
  P extends "color" | "background-color" ? `#${string}` | `rgb(${string})` :
  P extends "font-size" ? `${number}px` | `${number}rem` :
  string;

// API routes
type HttpMethod = "GET" | "POST" | "PUT" | "DELETE";
type ApiRoute = `/api/${string}`;
type ApiEndpoint = `${HttpMethod} ${ApiRoute}`;

type UserEndpoints = 
  | "GET /api/users"
  | "GET /api/users/:id"
  | "POST /api/users"
  | "PUT /api/users/:id"
  | "DELETE /api/users/:id";

// Type-safe database queries
type Table = "users" | "posts" | "comments";
type Action = "select" | "insert" | "update" | "delete";
type Query = `${Action}_${Table}`;
// "select_users" | "insert_users" | "update_users" | ...

// Practical: environment variables
type Env = "development" | "staging" | "production";
type EnvVar = `${Uppercase<Env>}_API_URL`;
// "DEVELOPMENT_API_URL" | "STAGING_API_URL" | "PRODUCTION_API_URL"
```

### Utility Types

TypeScript's built-in utility types solve common type transformation needs.

✅ **Good: Practical utility type usage**
```typescript
interface User {
  id: string;
  name: string;
  email: string;
  age: number;
  role: "admin" | "user";
}

// Pick: Select specific properties
type UserCredentials = Pick<User, "email">;
type UserProfile = Pick<User, "name" | "email" | "age">;

// Omit: Exclude specific properties
type UserWithoutId = Omit<User, "id">;
type PublicUser = Omit<User, "email" | "age">;

// Partial: Make all properties optional
type UserUpdate = Partial<User>;

function updateUser(id: string, updates: UserUpdate) {
  // Can update any subset of fields
}

// Required: Make all properties required
type CompleteUser = Required<UserUpdate>;

// Record: Create object type with specific keys
type UserMap = Record<string, User>;
type RolePermissions = Record<User["role"], string[]>;

const permissions: RolePermissions = {
  admin: ["read", "write", "delete"],
  user: ["read"],
};

// Exclude: Remove types from union
type Role = "admin" | "user" | "guest";
type AuthenticatedRole = Exclude<Role, "guest">; // "admin" | "user"

// Extract: Keep only specified types from union
type AdminRole = Extract<Role, "admin">; // "admin"

// NonNullable: Remove null and undefined
type MaybeString = string | null | undefined;
type DefiniteString = NonNullable<MaybeString>; // string

// ReturnType: Extract function return type
function getUser() {
  return { id: "1", name: "Alice", email: "alice@example.com" };
}

type UserType = ReturnType<typeof getUser>;

// Parameters: Extract function parameter types
function createUser(name: string, email: string, age: number) {
  return { id: crypto.randomUUID(), name, email, age };
}

type CreateUserParams = Parameters<typeof createUser>; // [string, string, number]

// Awaited: Unwrap Promise type
type AsyncUser = Awaited<Promise<User>>; // User
```

### The `infer` Keyword

The `infer` keyword extracts types from within conditional types.

✅ **Good: Advanced type extraction with infer**
```typescript
// Extract array element type
type Unpacked<T> = T extends (infer U)[] ? U : T;

type A = Unpacked<string[]>;        // string
type B = Unpacked<number>;          // number

// Extract Promise resolve type
type Awaited<T> = T extends Promise<infer U> ? Awaited<U> : T;

type C = Awaited<Promise<string>>;  // string
type D = Awaited<Promise<Promise<number>>>; // number (recursive)

// Extract function return type
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

type E = ReturnType<() => string>;  // string
type F = ReturnType<(x: number) => boolean>; // boolean

// Extract function argument types
type FirstArg<T> = T extends (first: infer F, ...args: any[]) => any ? F : never;

type G = FirstArg<(name: string, age: number) => void>; // string

// Practical: Extract state from reducer
type Action = 
  | { type: "SET_USER"; payload: User }
  | { type: "SET_LOADING"; payload: boolean };

type ExtractPayload<T, A> = T extends { type: A; payload: infer P } ? P : never;

type UserPayload = ExtractPayload<Action, "SET_USER">; // User
type LoadingPayload = ExtractPayload<Action, "SET_LOADING">; // boolean

// Extract component props
type GetComponentProps<T> = T extends React.ComponentType<infer P> ? P : never;

const MyComponent: React.FC<{ name: string; age: number }> = () => null;
type MyComponentProps = GetComponentProps<typeof MyComponent>;
// { name: string; age: number }

// Deep property access
type PropertyType<T, Path extends string> = 
  Path extends `${infer Key}.${infer Rest}`
    ? Key extends keyof T
      ? PropertyType<T[Key], Rest>
      : never
    : Path extends keyof T
      ? T[Path]
      : never;

type User = {
  profile: {
    address: {
      city: string;
    }
  }
};

type City = PropertyType<User, "profile.address.city">; // string
```

---

## 3. Type-Safe APIs & Data Fetching

### Typing HTTP Responses

Shape API response types to match your actual API contracts.

✅ **Good: Discriminated unions for API responses**
```typescript
// Result type pattern
type Result<T, E = Error> = 
  | { success: true; data: T }
  | { success: false; error: E };

// Specific API response
type ApiResponse<T> = 
  | { status: "success"; data: T; timestamp: number }
  | { status: "error"; error: string; code: number }
  | { status: "loading" };

async function fetchUser(id: string): Promise<Result<User>> {
  try {
    const response = await fetch(`/api/users/${id}`);
    
    if (!response.ok) {
      return {
        success: false,
        error: new Error(`HTTP ${response.status}: ${response.statusText}`)
      };
    }
    
    const data = await response.json();
    return { success: true, data };
  } catch (error) {
    return {
      success: false,
      error: error instanceof Error ? error : new Error("Unknown error")
    };
  }
}

// Usage with type narrowing
const result = await fetchUser("123");
if (result.success) {
  console.log(result.data.name); // Type: User
} else {
  console.error(result.error.message); // Type: Error
}
```

❌ **Bad: Loose typing or throwing errors**
```typescript
// No error handling in return type
async function fetchUser(id: string): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  return response.json(); // What if it fails?
}

// Forces try/catch everywhere
async function getUser(id: string): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  if (!response.ok) throw new Error("Failed"); // Caller must handle
  return response.json();
}
```

### Type-Safe API Clients

Build API clients with full type safety from request to response.

✅ **Good: Generic API client**
```typescript
type HttpMethod = "GET" | "POST" | "PUT" | "DELETE" | "PATCH";

interface RequestConfig<T = unknown> {
  method: HttpMethod;
  body?: T;
  headers?: Record<string, string>;
  params?: Record<string, string>;
}

class ApiClient {
  constructor(private baseUrl: string) {}

  private async request<TResponse, TRequest = undefined>(
    endpoint: string,
    config: RequestConfig<TRequest>
  ): Promise<Result<TResponse>> {
    const url = new URL(endpoint, this.baseUrl);
    
    if (config.params) {
      Object.entries(config.params).forEach(([key, value]) => {
        url.searchParams.append(key, value);
      });
    }

    try {
      const response = await fetch(url.toString(), {
        method: config.method,
        headers: {
          "Content-Type": "application/json",
          ...config.headers,
        },
        body: config.body ? JSON.stringify(config.body) : undefined,
      });

      if (!response.ok) {
        return {
          success: false,
          error: new Error(`HTTP ${response.status}`)
        };
      }

      const data = await response.json();
      return { success: true, data };
    } catch (error) {
      return {
        success: false,
        error: error instanceof Error ? error : new Error("Unknown error")
      };
    }
  }

  async get<T>(endpoint: string, params?: Record<string, string>) {
    return this.request<T>(endpoint, { method: "GET", params });
  }

  async post<TResponse, TRequest>(endpoint: string, body: TRequest) {
    return this.request<TResponse, TRequest>(endpoint, { method: "POST", body });
  }

  async put<TResponse, TRequest>(endpoint: string, body: TRequest) {
    return this.request<TResponse, TRequest>(endpoint, { method: "PUT", body });
  }

  async delete<T>(endpoint: string) {
    return this.request<T>(endpoint, { method: "DELETE" });
  }
}

// Type-safe usage
interface User {
  id: string;
  name: string;
  email: string;
}

interface CreateUserRequest {
  name: string;
  email: string;
}

const api = new ApiClient("https://api.example.com");

// GET with type inference
const users = await api.get<User[]>("/users");

// POST with request and response types
const newUser = await api.post<User, CreateUserRequest>("/users", {
  name: "Alice",
  email: "alice@example.com"
});
```

### Runtime Validation with Zod

Combine TypeScript's compile-time types with runtime validation.

✅ **Good: Zod schemas for API validation**
```typescript
import { z } from "zod";

// Define schema
const UserSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1).max(100),
  email: z.string().email(),
  age: z.number().int().positive().optional(),
  role: z.enum(["admin", "user", "guest"]),
  createdAt: z.string().datetime(),
});

// Extract TypeScript type from schema
type User = z.infer<typeof UserSchema>;

// Validate API response
async function fetchUser(id: string): Promise<Result<User>> {
  const response = await fetch(`/api/users/${id}`);
  
  if (!response.ok) {
    return { success: false, error: new Error("Fetch failed") };
  }

  const data = await response.json();
  
  // Runtime validation
  const parsed = UserSchema.safeParse(data);
  
  if (!parsed.success) {
    return {
      success: false,
      error: new Error(`Validation failed: ${parsed.error.message}`)
    };
  }

  return { success: true, data: parsed.data };
}

// Complex nested schemas
const AddressSchema = z.object({
  street: z.string(),
  city: z.string(),
  state: z.string().length(2),
  zip: z.string().regex(/^\d{5}$/),
});

const UserProfileSchema = UserSchema.extend({
  address: AddressSchema,
  preferences: z.object({
    theme: z.enum(["light", "dark"]),
    notifications: z.boolean(),
  }),
});

type UserProfile = z.infer<typeof UserProfileSchema>;
```

### Error Handling Patterns

Handle errors in a type-safe, predictable way.

✅ **Good: Structured error types**
```typescript
// Base error type
class AppError extends Error {
  constructor(
    message: string,
    public code: string,
    public statusCode: number = 500
  ) {
    super(message);
    this.name = this.constructor.name;
  }
}

// Specific error types
class ValidationError extends AppError {
  constructor(message: string, public fields: Record<string, string>) {
    super(message, "VALIDATION_ERROR", 400);
  }
}

class NotFoundError extends AppError {
  constructor(resource: string, id: string) {
    super(`${resource} with id ${id} not found`, "NOT_FOUND", 404);
  }
}

class UnauthorizedError extends AppError {
  constructor(message: string = "Unauthorized") {
    super(message, "UNAUTHORIZED", 401);
  }
}

// Type-safe error handling
type ServiceResult<T> = Result<T, AppError>;

async function getUserWithPosts(userId: string): Promise<ServiceResult<UserWithPosts>> {
  const userResult = await fetchUser(userId);
  
  if (!userResult.success) {
    return {
      success: false,
      error: new NotFoundError("User", userId)
    };
  }

  const postsResult = await fetchUserPosts(userId);
  
  if (!postsResult.success) {
    return {
      success: false,
      error: new AppError("Failed to fetch posts", "POSTS_ERROR")
    };
  }

  return {
    success: true,
    data: {
      ...userResult.data,
      posts: postsResult.data
    }
  };
}
```

---

## 4. React + TypeScript Patterns

### Component Prop Types

Define clear interfaces for component props.

✅ **Good: Well-typed React components**
```typescript
import React from "react";

// Basic component props
interface ButtonProps {
  children: React.ReactNode;
  onClick: () => void;
  variant?: "primary" | "secondary" | "danger";
  disabled?: boolean;
  className?: string;
}

export const Button: React.FC<ButtonProps> = ({
  children,
  onClick,
  variant = "primary",
  disabled = false,
  className = "",
}) => {
  return (
    <button
      onClick={onClick}
      disabled={disabled}
      className={`btn btn-${variant} ${className}`}
    >
      {children}
    </button>
  );
};

// Extending HTML attributes
interface InputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  label: string;
  error?: string;
}

export const Input: React.FC<InputProps> = ({ label, error, ...props }) => {
  return (
    <div>
      <label>{label}</label>
      <input {...props} />
      {error && <span className="error">{error}</span>}
    </div>
  );
};

// Union type for different configurations
type CardProps = 
  | { variant: "user"; user: User }
  | { variant: "product"; product: Product }
  | { variant: "placeholder" };

export const Card: React.FC<CardProps> = (props) => {
  if (props.variant === "user") {
    return <div>User: {props.user.name}</div>;
  }
  
  if (props.variant === "product") {
    return <div>Product: {props.product.name}</div>;
  }
  
  return <div>Loading...</div>;
};
```

### Hooks Typing

Type hooks correctly for maximum type safety.

✅ **Good: Strongly typed hooks**
```typescript
import { useState, useEffect, useRef, useContext } from "react";

// useState with explicit type
const [user, setUser] = useState<User | null>(null);

// useState with type inference
const [count, setCount] = useState(0); // inferred as number

// useEffect with async
useEffect(() => {
  let cancelled = false;

  async function fetchData() {
    const result = await fetchUser("123");
    if (!cancelled && result.success) {
      setUser(result.data);
    }
  }

  fetchData();

  return () => {
    cancelled = true;
  };
}, []);

// useRef for DOM elements
const inputRef = useRef<HTMLInputElement>(null);

useEffect(() => {
  inputRef.current?.focus();
}, []);

// useRef for mutable values
const renderCount = useRef(0);

useEffect(() => {
  renderCount.current += 1;
});

// Custom hooks with generics
function useAsync<T>(
  asyncFn: () => Promise<T>
): {
  data: T | null;
  loading: boolean;
  error: Error | null;
  execute: () => Promise<void>;
} {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);

  const execute = async () => {
    setLoading(true);
    setError(null);
    
    try {
      const result = await asyncFn();
      setData(result);
    } catch (err) {
      setError(err instanceof Error ? err : new Error("Unknown error"));
    } finally {
      setLoading(false);
    }
  };

  return { data, loading, error, execute };
}

// Usage
const { data: user, loading, error, execute } = useAsync<User>(() =>
  fetch("/api/user").then(r => r.json())
);
```

### Context API Typing

Type Context API for compile-time safety.

✅ **Good: Type-safe Context**
```typescript
import React, { createContext, useContext, useState } from "react";

// Define context value type
interface AuthContextValue {
  user: User | null;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
  isAuthenticated: boolean;
}

// Create context with undefined default
const AuthContext = createContext<AuthContextValue | undefined>(undefined);

// Provider component
export const AuthProvider: React.FC<{ children: React.ReactNode }> = ({
  children
}) => {
  const [user, setUser] = useState<User | null>(null);

  const login = async (email: string, password: string) => {
    const result = await loginApi(email, password);
    if (result.success) {
      setUser(result.data);
    }
  };

  const logout = () => {
    setUser(null);
  };

  const value: AuthContextValue = {
    user,
    login,
    logout,
    isAuthenticated: user !== null,
  };

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
};

// Custom hook with type checking
export function useAuth(): AuthContextValue {
  const context = useContext(AuthContext);
  
  if (context === undefined) {
    throw new Error("useAuth must be used within AuthProvider");
  }
  
  return context;
}

// Usage
function ProfilePage() {
  const { user, logout } = useAuth(); // Fully typed!
  
  return (
    <div>
      <h1>{user?.name}</h1>
      <button onClick={logout}>Logout</button>
    </div>
  );
}
```

### Generic Components

Create reusable components that work with any data type.

✅ **Good: Generic List component**
```typescript
interface ListProps<T> {
  items: T[];
  renderItem: (item: T, index: number) => React.ReactNode;
  keyExtractor: (item: T) => string | number;
  emptyMessage?: string;
}

function List<T>({
  items,
  renderItem,
  keyExtractor,
  emptyMessage = "No items"
}: ListProps<T>) {
  if (items.length === 0) {
    return <div>{emptyMessage}</div>;
  }

  return (
    <ul>
      {items.map((item, index) => (
        <li key={keyExtractor(item)}>
          {renderItem(item, index)}
        </li>
      ))}
    </ul>
  );
}

// Usage with full type inference
interface User {
  id: string;
  name: string;
  email: string;
}

function UserList({ users }: { users: User[] }) {
  return (
    <List
      items={users}
      renderItem={(user) => (
        <div>
          {user.name} - {user.email}
        </div>
      )}
      keyExtractor={(user) => user.id}
      emptyMessage="No users found"
    />
  );
}

// Generic Select component
interface SelectProps<T> {
  options: T[];
  value: T;
  onChange: (value: T) => void;
  getLabel: (option: T) => string;
  getValue: (option: T) => string | number;
}

function Select<T>({
  options,
  value,
  onChange,
  getLabel,
  getValue
}: SelectProps<T>) {
  return (
    <select
      value={getValue(value)}
      onChange={(e) => {
        const selected = options.find(
          opt => getValue(opt).toString() === e.target.value
        );
        if (selected) onChange(selected);
      }}
    >
      {options.map((option) => (
        <option key={getValue(option)} value={getValue(option)}>
          {getLabel(option)}
        </option>
      ))}
    </select>
  );
}
```

---

## 5. Node.js + TypeScript Patterns

### Express Middleware Typing

Type Express middleware and request handlers correctly.

✅ **Good: Type-safe Express setup**
```typescript
import express, { Request, Response, NextFunction, RequestHandler } from "express";

// Extend Express types for custom properties
declare global {
  namespace Express {
    interface Request {
      user?: User;
      requestId?: string;
    }
  }
}

// Type-safe middleware
const authMiddleware: RequestHandler = async (req, res, next) => {
  const token = req.headers.authorization?.split(" ")[1];
  
  if (!token) {
    return res.status(401).json({ error: "No token provided" });
  }

  try {
    const user = await verifyToken(token);
    req.user = user;
    next();
  } catch (error) {
    res.status(401).json({ error: "Invalid token" });
  }
};

// Generic async handler wrapper
function asyncHandler<T = any>(
  fn: (req: Request, res: Response, next: NextFunction) => Promise<T>
): RequestHandler {
  return (req, res, next) => {
    Promise.resolve(fn(req, res, next)).catch(next);
  };
}

// Type-safe route handlers
interface CreateUserBody {
  name: string;
  email: string;
}

interface UserParams {
  id: string;
}

const createUser = asyncHandler<void>(async (req, res) => {
  const body = req.body as CreateUserBody;
  
  // Validation
  if (!body.name || !body.email) {
    res.status(400).json({ error: "Name and email required" });
    return;
  }

  const user = await createUserInDb(body);
  res.status(201).json(user);
});

const getUser = asyncHandler<void>(async (req, res) => {
  const { id } = req.params as UserParams;
  
  const user = await getUserFromDb(id);
  
  if (!user) {
    res.status(404).json({ error: "User not found" });
    return;
  }

  res.json(user);
});

// Error handler
const errorHandler = (
  err: Error,
  req: Request,
  res: Response,
  next: NextFunction
) => {
  console.error(err.stack);

  if (err instanceof ValidationError) {
    res.status(400).json({ error: err.message, fields: err.fields });
    return;
  }

  if (err instanceof NotFoundError) {
    res.status(404).json({ error: err.message });
    return;
  }

  res.status(500).json({ error: "Internal server error" });
};

// App setup
const app = express();
app.use(express.json());
app.use(authMiddleware);
app.post("/users", createUser);
app.get("/users/:id", getUser);
app.use(errorHandler);
```

### Async Patterns

Handle async operations with proper typing.

✅ **Good: Type-safe async patterns**
```typescript
// Promise with specific return type
async function fetchUserData(id: string): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  return response.json();
}

// Promise.all with tuple types
async function loadDashboard(userId: string) {
  const [user, posts, comments] = await Promise.all([
    fetchUser(userId),
    fetchPosts(userId),
    fetchComments(userId),
  ]);
  
  return { user, posts, comments };
}

// Type inference works!
const dashboard = await loadDashboard("123");
// dashboard.user is typed as User
// dashboard.posts is typed as Post[]

// Parallel execution with different types
async function loadHomepage() {
  const results = await Promise.allSettled([
    fetchFeaturedPosts(),
    fetchTrendingTopics(),
    fetchUserNotifications(),
  ]);

  return {
    posts: results[0].status === "fulfilled" ? results[0].value : [],
    trending: results[1].status === "fulfilled" ? results[1].value : [],
    notifications: results[2].status === "fulfilled" ? results[2].value : [],
  };
}

// Retry logic with generics
async function retry<T>(
  fn: () => Promise<T>,
  maxAttempts: number = 3,
  delay: number = 1000
): Promise<T> {
  let lastError: Error;

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error instanceof Error ? error : new Error("Unknown error");
      
      if (attempt < maxAttempts) {
        await new Promise(resolve => setTimeout(resolve, delay * attempt));
      }
    }
  }

  throw lastError!;
}

// Usage
const user = await retry(() => fetchUser("123"), 3, 2000);
```

### Configuration Typing

Type application configuration for compile-time safety.

✅ **Good: Strongly typed config**
```typescript
// config.ts
interface DatabaseConfig {
  host: string;
  port: number;
  database: string;
  user: string;
  password: string;
  ssl: boolean;
}

interface RedisConfig {
  host: string;
  port: number;
  password?: string;
  db: number;
}

interface ApiConfig {
  port: number;
  cors: {
    origin: string | string[];
    credentials: boolean;
  };
  rateLimit: {
    windowMs: number;
    max: number;
  };
}

interface AppConfig {
  env: "development" | "staging" | "production";
  database: DatabaseConfig;
  redis: RedisConfig;
  api: ApiConfig;
  jwt: {
    secret: string;
    expiresIn: string;
  };
}

// Type-safe environment variable parsing
function getEnvVar(key: string, defaultValue?: string): string {
  const value = process.env[key];
  
  if (value === undefined) {
    if (defaultValue !== undefined) {
      return defaultValue;
    }
    throw new Error(`Environment variable ${key} is required`);
  }
  
  return value;
}

function getEnvNumber(key: string, defaultValue?: number): number {
  const value = process.env[key];
  
  if (value === undefined) {
    if (defaultValue !== undefined) {
      return defaultValue;
    }
    throw new Error(`Environment variable ${key} is required`);
  }
  
  const parsed = parseInt(value, 10);
  
  if (isNaN(parsed)) {
    throw new Error(`Environment variable ${key} must be a number`);
  }
  
  return parsed;
}

// Build config
export const config: AppConfig = {
  env: getEnvVar("NODE_ENV", "development") as AppConfig["env"],
  database: {
    host: getEnvVar("DB_HOST", "localhost"),
    port: getEnvNumber("DB_PORT", 5432),
    database: getEnvVar("DB_NAME"),
    user: getEnvVar("DB_USER"),
    password: getEnvVar("DB_PASSWORD"),
    ssl: getEnvVar("DB_SSL", "false") === "true",
  },
  redis: {
    host: getEnvVar("REDIS_HOST", "localhost"),
    port: getEnvNumber("REDIS_PORT", 6379),
    password: process.env.REDIS_PASSWORD,
    db: getEnvNumber("REDIS_DB", 0),
  },
  api: {
    port: getEnvNumber("PORT", 3000),
    cors: {
      origin: getEnvVar("CORS_ORIGIN", "*"),
      credentials: true,
    },
    rateLimit: {
      windowMs: 15 * 60 * 1000,
      max: 100,
    },
  },
  jwt: {
    secret: getEnvVar("JWT_SECRET"),
    expiresIn: getEnvVar("JWT_EXPIRES_IN", "7d"),
  },
};
```

---

## 6. Common Pitfalls

### Overuse of `any`

Using `any` defeats TypeScript's purpose.

❌ **Bad: Excessive any usage**
```typescript
function processData(data: any): any {
  return data.map((item: any) => item.value);
}

const apiClient: any = createApiClient();
```

✅ **Good: Use proper types or unknown**
```typescript
function processData<T extends { value: unknown }>(data: T[]): unknown[] {
  return data.map(item => item.value);
}

// Use unknown when type is truly unknown
function parseJson(json: string): unknown {
  return JSON.parse(json);
}

// Then narrow the type
const data = parseJson(jsonString);
if (isUser(data)) {
  console.log(data.name); // Type narrowed to User
}
```

### Type Assertion Abuse

Don't use assertions to bypass type checking.

❌ **Bad: Assertions without validation**
```typescript
const user = apiResponse as User; // No runtime check!
const element = document.getElementById("app") as HTMLDivElement; // Might be null
```

✅ **Good: Validate before asserting**
```typescript
function isUser(value: unknown): value is User {
  return (
    typeof value === "object" &&
    value !== null &&
    "name" in value &&
    "email" in value
  );
}

const parsed = JSON.parse(jsonString);
if (isUser(parsed)) {
  const user = parsed; // Type is narrowed, no assertion needed
}

const element = document.getElementById("app");
if (element instanceof HTMLDivElement) {
  // Safe to use as HTMLDivElement
}
```

### Overly Complex Types

Keep types maintainable.

❌ **Bad: Unreadable complex type**
```typescript
type ComplexType<T> = T extends (infer U)[]
  ? U extends { a: infer A; b: infer B }
    ? A extends string
      ? B extends number
        ? { processed: true; value: `${A}-${B}` }
        : never
      : never
    : never
  : never;
```

✅ **Good: Break down complex types**
```typescript
type ExtractArrayElement<T> = T extends (infer U)[] ? U : never;

type HasProperties<T> = T extends { a: infer A; b: infer B }
  ? { a: A; b: B }
  : never;

type ProcessedValue<A, B> = A extends string
  ? B extends number
    ? { processed: true; value: `${A}-${B}` }
    : never
  : never;

// Combine them
type ComplexType<T> = ProcessedValue<
  HasProperties<ExtractArrayElement<T>>["a"],
  HasProperties<ExtractArrayElement<T>>["b"]
>;
```

### Missing Null/Undefined Checks

Always enable `strictNullChecks` in tsconfig.json.

❌ **Bad: Assuming values exist**
```typescript
function getUserName(id: string): string {
  const user = users.find(u => u.id === id);
  return user.name; // Error: user might be undefined
}
```

✅ **Good: Handle null/undefined**
```typescript
function getUserName(id: string): string | undefined {
  const user = users.find(u => u.id === id);
  return user?.name;
}

// Or throw if it must exist
function getUserNameOrThrow(id: string): string {
  const user = users.find(u => u.id === id);
  
  if (!user) {
    throw new Error(`User ${id} not found`);
  }
  
  return user.name;
}
```

### Type vs Interface Confusion

Know when to use each.

✅ **Good: Use interface for object shapes**
```typescript
// Use interface for objects that might be extended
interface User {
  id: string;
  name: string;
}

interface Admin extends User {
  permissions: string[];
}

// Can be merged (declaration merging)
interface Window {
  myApp: MyApp;
}
```

✅ **Good: Use type for unions, intersections, and utilities**
```typescript
// Use type for unions
type Status = "pending" | "approved" | "rejected";

// Use type for complex compositions
type Result<T> = { success: true; data: T } | { success: false; error: Error };

// Use type for mapped types
type Readonly<T> = {
  readonly [P in keyof T]: T[P];
};

// Use type for tuple types
type Point = [number, number];
```

---

## Checklist

### Type System Basics
- [ ] Let TypeScript infer types when they're obvious
- [ ] Add explicit types for public APIs and complex logic
- [ ] Use union types for alternatives, intersection types for composition
- [ ] Implement discriminated unions with a common `type` or `kind` field
- [ ] Create custom type guards for complex type narrowing
- [ ] Use type assertions only after runtime validation

### Advanced Types
- [ ] Use generics for reusable, type-safe functions and components
- [ ] Add constraints to generics (`extends`) to ensure correct usage
- [ ] Leverage conditional types for type-level logic
- [ ] Use mapped types to transform existing types
- [ ] Utilize built-in utility types (Pick, Omit, Partial, Required, etc.)
- [ ] Use `infer` keyword for advanced type extraction

### API & Data Handling
- [ ] Return Result/Either types instead of throwing errors
- [ ] Validate API responses at runtime (use Zod or similar)
- [ ] Type all async functions with Promise<T>
- [ ] Create type-safe API clients with proper generics
- [ ] Handle loading, error, and success states in data fetching
- [ ] Use discriminated unions for API response states

### React Patterns
- [ ] Define explicit prop interfaces for all components
- [ ] Type event handlers correctly (e.g., `React.MouseEvent<HTMLButtonElement>`)
- [ ] Use `React.FC<Props>` or explicit function signatures
- [ ] Type useState with explicit types for complex state
- [ ] Type useRef for both DOM elements and mutable values
- [ ] Create type-safe Context with proper Provider and hook patterns
- [ ] Use generic components for reusable UI elements

### Node.js Patterns
- [ ] Extend Express types for custom Request properties
- [ ] Type all middleware and route handlers
- [ ] Wrap async handlers to catch promise rejections
- [ ] Create strongly typed configuration from environment variables
- [ ] Type database queries and models
- [ ] Use proper error classes with type guards

### Configuration
- [ ] Enable `strict: true` in tsconfig.json
- [ ] Enable `strictNullChecks` to catch null/undefined errors
- [ ] Enable `noImplicitAny` to avoid accidental any usage
- [ ] Enable `strictFunctionTypes` for safer function typing
- [ ] Use `esModuleInterop` for better module compatibility

### Code Quality
- [ ] Avoid `any` - use `unknown` and type guards instead
- [ ] Don't use type assertions without validation
- [ ] Check for null/undefined before accessing properties
- [ ] Break down complex types into smaller, named types
- [ ] Use interface for object shapes, type for unions/utilities
- [ ] Document complex types with comments
- [ ] Prefer type inference over explicit annotations when obvious
- [ ] Use exhaustiveness checking in switch statements with `never`
