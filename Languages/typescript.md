# TypeScript Cheatsheet

## Mental Model

TypeScript is **JavaScript with a type system**. It compiles down to plain JS — no runtime overhead, no new syntax at execution time. The type system is structural (duck typing): if a value has the right shape, it satisfies the type, regardless of what class it came from. Types exist only at compile time; `tsc` erases them all. The goal isn't to replicate Java or C# — it's to catch the specific class of bugs that hit JS developers most: undefined is not a function, cannot read property of null, wrong argument types.

---

## Install & Minimal Setup

```bash
# Install TypeScript
npm install -g typescript
npm install --save-dev typescript @types/node ts-node

# Init config
tsc --init    # creates tsconfig.json

# Compile
tsc           # compile all files per tsconfig.json
tsc index.ts  # compile one file

# Run directly (dev only)
npx ts-node index.ts
npx tsx index.ts    # faster alternative

# Watch mode
tsc --watch
```

```json
// tsconfig.json — recommended settings
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "CommonJS",       // or "ESNext" for ESM
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,             // enables all strict checks — always enable
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,        // emit .d.ts files (for libraries)
    "sourceMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

---

## Core Concepts

### 1. Basic Types

```typescript
// Primitives
let name:    string  = "Joshua";
let age:     number  = 28;
let active:  boolean = true;
let nothing: null    = null;
let missing: undefined = undefined;

// Arrays
let scores: number[]     = [1, 2, 3];
let names:  Array<string> = ["a", "b"];  // generic form

// Tuple — fixed-length array with known types at each position
let coords: [number, number] = [19.4, -99.1];
let entry:  [string, number] = ["score", 95];

// any — escape hatch, disables type checking (avoid)
let data: any = JSON.parse(rawJson);

// unknown — safer any — must narrow type before using
let input: unknown = getUserInput();
if (typeof input === "string") {
    input.toUpperCase();  // ok — narrowed to string
}

// never — value that never exists (exhaustive checks, throwing functions)
function fail(msg: string): never {
    throw new Error(msg);
}

// void — function with no meaningful return
function log(msg: string): void {
    console.log(msg);
}
```

### 2. Type Aliases & Interfaces

```typescript
// Type alias — can be a union, primitive, tuple, mapped type
type UserId = number;
type Status = "active" | "inactive" | "pending";  // string literal union
type Callback = (error: Error | null, result: string) => void;

type Point = {
    x: number;
    y: number;
    label?: string;  // optional field
};

// Interface — for objects and classes, can be extended/merged
interface User {
    readonly id: number;   // readonly — can't reassign after creation
    name: string;
    email: string;
    role: Status;
    createdAt: Date;
}

// Extending
interface AdminUser extends User {
    permissions: string[];
}

// Interface merging — redeclare to add fields
interface Window {
    myCustomProp: string;  // adds to the built-in Window type
}

// Type vs Interface: use interface for objects/classes, type for unions and aliases
```

### 3. Union & Intersection Types

```typescript
// Union — one of several types
type StringOrNumber = string | number;
type ApiResponse = SuccessResponse | ErrorResponse;

function formatId(id: string | number): string {
    if (typeof id === "number") {
        return id.toFixed(0).padStart(6, "0");
    }
    return id;  // TypeScript knows it's string here
}

// Intersection — combines types (must satisfy all)
type AdminUser = User & { permissions: string[] };

// Discriminated union — union with a common literal field for narrowing
type Shape =
    | { kind: "circle"; radius: number }
    | { kind: "rect"; width: number; height: number }
    | { kind: "triangle"; base: number; height: number };

function area(shape: Shape): number {
    switch (shape.kind) {
        case "circle":   return Math.PI * shape.radius ** 2;
        case "rect":     return shape.width * shape.height;
        case "triangle": return 0.5 * shape.base * shape.height;
    }
}
```

### 4. Generics

```typescript
// Generic function
function first<T>(arr: T[]): T | undefined {
    return arr[0];
}
first([1, 2, 3]);         // T inferred as number
first(["a", "b"]);        // T inferred as string

// Generic with constraint
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
    return obj[key];
}
getProperty({ name: "Joshua", age: 28 }, "name");   // string
getProperty({ name: "Joshua", age: 28 }, "age");    // number

// Generic interface
interface Repository<T, ID = number> {
    findById(id: ID): Promise<T | null>;
    findAll(): Promise<T[]>;
    save(entity: T): Promise<T>;
    delete(id: ID): Promise<void>;
}

// Generic class
class Stack<T> {
    private items: T[] = [];
    push(item: T): void { this.items.push(item); }
    pop(): T | undefined { return this.items.pop(); }
    peek(): T | undefined { return this.items[this.items.length - 1]; }
}
```

### 5. Utility Types

```typescript
interface User {
    id: number;
    name: string;
    email: string;
    password: string;
    role: string;
}

Partial<User>          // all fields optional
Required<User>         // all fields required
Readonly<User>         // all fields readonly
Pick<User, "id"|"name"|"email">  // subset of fields
Omit<User, "password">           // all fields except password
Record<string, number>           // { [key: string]: number }
Exclude<"a"|"b"|"c", "a">       // "b" | "c"
Extract<"a"|"b"|"c", "a"|"b">   // "a" | "b"
NonNullable<string | null | undefined>  // string
ReturnType<typeof fetch>         // Promise<Response>
Parameters<typeof fetch>         // [input: RequestInfo, init?: RequestInit]
Awaited<Promise<string>>         // string

// Usage pattern — DTO derived from model
type UserCreateDto = Omit<User, "id">;
type UserUpdateDto = Partial<Pick<User, "name" | "email" | "role">>;
type UserPublicDto  = Omit<User, "password">;
```

### 6. Type Narrowing

```typescript
// typeof guard
function process(val: string | number) {
    if (typeof val === "string") {
        return val.toUpperCase();  // string
    }
    return val.toFixed(2);         // number
}

// instanceof guard
function handleError(error: unknown) {
    if (error instanceof Error) {
        console.error(error.message);
    } else {
        console.error(String(error));
    }
}

// in operator guard
function printShape(shape: Circle | Rectangle) {
    if ("radius" in shape) {
        console.log(shape.radius);   // Circle
    } else {
        console.log(shape.width);    // Rectangle
    }
}

// Type predicate (custom type guard)
function isUser(value: unknown): value is User {
    return typeof value === "object" && value !== null && "email" in value;
}

// Assertion function
function assertDefined<T>(value: T | null | undefined): asserts value is T {
    if (value == null) throw new Error("Value is null or undefined");
}
```

### 7. Enums vs Const Objects

```typescript
// Enum — avoid string enums (runtime overhead)
enum Direction {
    Up = "UP",
    Down = "DOWN",
}

// Const object + typeof — preferred (tree-shakeable, no runtime cost)
const Direction = {
    Up: "UP",
    Down: "DOWN",
} as const;

type Direction = typeof Direction[keyof typeof Direction];  // "UP" | "DOWN"

function move(dir: Direction) { ... }
move(Direction.Up);   // ✅
move("UP");           // ✅ — literal also accepted
move("LEFT");         // ❌ — compile error
```

### 8. Classes

```typescript
class UserService {
    // Class fields (TypeScript)
    private readonly users: Map<number, User> = new Map();
    private static instance: UserService;

    // Constructor parameter shorthand
    constructor(
        private readonly db: Database,
        private readonly logger: Logger,
    ) {}

    // Static factory (singleton pattern)
    static getInstance(db: Database, logger: Logger): UserService {
        if (!UserService.instance) {
            UserService.instance = new UserService(db, logger);
        }
        return UserService.instance;
    }

    async findById(id: number): Promise<User | null> {
        return this.db.query<User>(`SELECT * FROM users WHERE id = $1`, [id]);
    }
}

// Implementing an interface
class PostgresUserRepository implements Repository<User> {
    async findById(id: number): Promise<User | null> { ... }
    async findAll(): Promise<User[]> { ... }
    async save(user: User): Promise<User> { ... }
    async delete(id: number): Promise<void> { ... }
}
```

### 9. Async / Promises

```typescript
// async / await — always prefer over raw Promises
async function fetchUser(id: number): Promise<User> {
    const response = await fetch(`/api/users/${id}`);
    if (!response.ok) {
        throw new Error(`HTTP ${response.status}`);
    }
    return response.json() as Promise<User>;
}

// Error handling
async function safeGetUser(id: number): Promise<User | null> {
    try {
        return await fetchUser(id);
    } catch (error) {
        console.error("Failed to fetch user:", error);
        return null;
    }
}

// Parallel execution
const [user, orders] = await Promise.all([fetchUser(id), fetchOrders(id)]);

// Type a promise result
const result: Awaited<ReturnType<typeof fetchUser>> = await fetchUser(1);
```

---

## Most-Used Patterns

### Zod — Runtime Validation + Type Inference

```typescript
import { z } from "zod";

const UserCreateSchema = z.object({
    name:  z.string().min(2).max(100),
    email: z.string().email(),
    age:   z.number().int().min(18).max(120),
    role:  z.enum(["admin", "user", "guest"]).default("user"),
});

// Infer type from schema — single source of truth
type UserCreate = z.infer<typeof UserCreateSchema>;

// Parse (throws on invalid input)
const user = UserCreateSchema.parse(requestBody);

// Safe parse (no throw)
const result = UserCreateSchema.safeParse(requestBody);
if (!result.success) {
    console.error(result.error.flatten());
} else {
    const user = result.data;
}
```

### Branded Types (Prevent ID mix-ups)

```typescript
type UserId    = number & { readonly _brand: "UserId" };
type ProductId = number & { readonly _brand: "ProductId" };

function makeUserId(id: number): UserId { return id as UserId; }

function getUser(id: UserId): User { ... }

getUser(1);                      // ❌ — plain number
getUser(makeUserId(1));          // ✅
getUser(someProductId);          // ❌ — wrong brand
```

---

## Gotchas

- **`strict: true` is non-negotiable** — without it, TypeScript misses half the bugs it was designed to catch. Always enable.
- **`as` type assertions are lies** — `const user = data as User` doesn't validate at runtime. Use Zod or another validator when you don't control the data source.
- **Optional chaining vs non-null assertion** — `user?.name` safely returns `undefined`; `user!.name` asserts it's not null and will throw at runtime if it is. Prefer `?.`.
- **`any` spreads** — one `any` leaks through function calls, assignments, and type inference. Use `unknown` and narrow instead.
- **Enums emit runtime code** — `const` enums are erased; regular enums generate IIFE boilerplate. Use const objects for simpler output.
- **`interface` vs `type`** — interfaces can be merged (declaration merging) and extended; type aliases can express unions and mapped types. Use interfaces for public API shapes, types for internal utilities.
- **`null` vs `undefined`** — TypeScript distinguishes them. With `strictNullChecks` (part of `strict`), neither is assignable to other types unless explicitly declared.

---

## Quick Links

- [TypeScript Docs](https://www.typescriptlang.org/docs/)
- [TypeScript Playground](https://www.typescriptlang.org/play)
- [Total TypeScript](https://www.totaltypescript.com) — best practical course
- [Zod](https://zod.dev) — schema validation + type inference
- [ts-reset](https://github.com/total-typescript/ts-reset) — sensible TypeScript defaults
- [type-fest](https://github.com/sindresorhus/type-fest) — utility type library
