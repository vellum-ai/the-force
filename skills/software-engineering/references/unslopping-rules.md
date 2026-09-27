---
name: vellum-unslopping
description: Remove code slop (pass-through wrappers, unjustified alias locals, identity transforms, leftover names, narrating comments, repeated fixed arguments, reusable export laundering, and asserted unknown-data shapes) without changing behavior. Use when running an Unslopping pass, cleaning agent-generated code, replacing handwritten external-data parsing, or reviewing a diff for needless indirection.
---

# Unslopping

Unslopping deletes indirection that does not earn its keep. The code after the pass should do the same thing, with fewer names, fewer hops, and fewer comments that restate the next line.

Unslopping is not restyling, not renaming for taste, and not collapsing a real boundary. If removing a name would change behavior, leak an internal, break a public export, or hide a test seam, keep it.

## How to run a pass

1. Search for the patterns below. Do not rewrite surrounding architecture while you are here.
2. Keep the name callers need. Delete the other name.
3. Delete rather than comment out. If a comment only makes sense as a record of the deletion, drop it. History lives in the PR.
4. Stay behavior-preserving. Run the focused tests for files you touch.
5. Skip generated files, vendored code, and snapshot fixtures unless the slop is in the generator input.

## What to keep

Keep a wrapper, alias, extra local, or parameter when it does at least one of these:

- Maps, validates, logs, or translates at the boundary (HTTP, IPC, package, skill).
- Is a public or CLI-facing name that other packages or shipped clients import.
- Exists so tests can stub one side without importing the other.
- Gives a domain name to a value that is otherwise a magic literal or a messy expression used more than once.

Braces on `if` / `else` / `for` / `while` stay. One-line bodies are not slop in this repo.

---

## Rules

### 1. Pass-through wrappers

A function whose body only calls one other function, with the same arguments and the same return, is slop. After two strategies collapse into one, the leftover dispatcher is the same bug.

Motivating case: after local and managed CES discovery share one socket path, `discoverCes()` became `return discoverManagedCes()`. That hop should go. Keep the better name (usually the one callers already use) and inline or retarget the other.

Slop:

```ts
export function discoverCes(): DiscoveryResult {
  return discoverManagedCes();
}
```

Unslopped: callers use one function. If `discoverCes` is the name the rest of the tree already imports, move the body there and delete `discoverManagedCes`.

```ts
export function discoverCes(): DiscoveryResult {
  const socketPath = getCesSocketPath();
  if (!isNamedPipePath(socketPath) && !existsSync(socketPath)) {
    return { mode: "unavailable", reason: `CES bootstrap socket not found at ${socketPath}` };
  }
  return { mode: "managed", socketPath };
}
```

Keep it when the wrapper is the package's public surface and the callee is private on purpose.

### 2. Alias locals

A variable that only rebinds another value, then gets passed along, is slop. Use the original name at the call site.

Slop:

```ts
const b = a;
callMethod(b);
```

Unslopped:

```ts
callMethod(a);
```

Same for `const socketPath = result.socketPath; connect(socketPath)` and for `const { socketPath } = result` when the alias is used once and the original name is already clear.

Keep it when the alias is a domain name that the original expression does not carry (`const guardianToken = process.env["TOKEN"]`), or when it is assigned so a later mutation or narrowing can happen.

A local is also justified when it avoids at least one repeated operation, or when inlining it would produce equal or greater code or line overhead. Judge net complexity, not variable count. Do not replace a named intermediate with a duplicated call or a longer expression just to remove one declaration.

### 3. Return-only locals

A local whose only job is to be returned on the next line is an alias with extra ceremony.

Slop:

```ts
function loadConfig(): Config {
  const config = readConfigFile();
  return config;
}
```

Unslopped:

```ts
function loadConfig(): Config {
  return readConfigFile();
}
```

Keep it when the next lines log, mutate, or narrow the value.

### 4. Boolean theater

Do not wrap a boolean in another boolean. Do not turn a condition into `true` / `false` by hand.

Slop:

```ts
const isReady = ready === true;
if (isReady) {
  start();
}
```

Unslopped:

```ts
if (ready) {
  start();
}
```

`isEnabled ? true : false`, `if (isEnabled) { return true; } return false;`, and `!!alreadyBoolean` are the same slop. `Boolean(x)` is fine when `x` is not already a boolean and you need a real `boolean`.

### 5. Identity transforms

A copy, map, spread, or template that yields the same value is slop.

Slop:

```ts
const name = `${userName}`;
const copy = { ...options };
const ids = userIds.map((id) => id);
return Promise.resolve(value);
```

Unslopped:

```ts
connect(userName, options, userIds);
return value;
```

Keep a copy when the next lines mutate it, when you must freeze a snapshot against later writes, or when you spread to add or override keys.

### 6. Narrating comments and banner dividers

Comments describe what the code is, not what the next line does, and not how a previous PR got here. Decorative `// ----` section banners are slop. So is a JSDoc that restates the function name.

Slop:

```ts
// ---------------------------------------------------------------------------
// Managed discovery
// ---------------------------------------------------------------------------

/**
 * Discover CES.
 */
export function discoverCes(): DiscoveryResult {
  // Get the socket path
  const socketPath = getCesSocketPath();
  // Check that it exists
  if (!existsSync(socketPath)) {
    return { mode: "unavailable", reason: `CES bootstrap socket not found at ${socketPath}` };
  }
  return { mode: "managed", socketPath };
}
```

Unslopped:

```ts
export function discoverCes(): DiscoveryResult {
  const socketPath = getCesSocketPath();
  if (!existsSync(socketPath)) {
    return { mode: "unavailable", reason: `CES bootstrap socket not found at ${socketPath}` };
  }
  return { mode: "managed", socketPath };
}
```

Keep a comment that states an invariant a reader cannot see from the code (fail-closed, "do not open a connection here, CES accepts one slot", why a timeout is 3s). Present tense only. No "no longer", "previously", "now uses".

### 7. Leftover names after a unification

When two paths become one, leftover vocabulary is slop: a `SiblingDiscoverySuccess` that is identical to `ManagedDiscoverySuccess`, a `mode: "sibling"` that nothing branches on, an env var name in a comment for a flag that is gone.

Slop:

```ts
export type DiscoveryResult =
  | ManagedDiscoverySuccess
  | SiblingDiscoverySuccess
  | DiscoveryFailure;
```

Unslopped:

```ts
export type DiscoveryResult = ManagedDiscoverySuccess | DiscoveryFailure;
```

If every environment uses the same socket path, stop calling the only remaining function `discoverManagedCes` unless "managed" is still a real distinction. Do not rename a `mode` string that is already on the wire.

Keep a distinct type or mode when callers still switch on it.

### 8. No-op try/catch

A `try` that only rethrows, or a `catch` that logs nothing and does not recover, is slop. So is `catch (error) { throw error; }`.

Slop:

```ts
try {
  return await readSecret(name);
} catch (error) {
  throw error;
}
```

Unslopped:

```ts
return await readSecret(name);
```

Keep try/catch when it translates the error (`RouteError`), records it (`captureError`), or degrades on purpose (daemon startup continues when a subsystem fails).

### 9. Duplicate callee guards

Do not repeat a check the function you are about to call already performs. The second check is a second source of truth and will drift.

Slop:

```ts
function connect(socketPath: string): void {
  if (!existsSync(socketPath)) {
    throw new Error(`missing socket: ${socketPath}`);
  }
  openSocket(socketPath);
}

function start(): void {
  const socketPath = getCesSocketPath();
  if (!existsSync(socketPath)) {
    throw new Error(`missing socket: ${socketPath}`);
  }
  connect(socketPath);
}
```

Unslopped:

```ts
function start(): void {
  connect(getCesSocketPath());
}
```

Keep a caller-side guard when it avoids a costly call, when the error must be a different shape at that layer, or when the callee is a third-party that does not fail closed.

### 10. Single-use speculative helpers

A helper, type alias, or options bag used once, that does not hide mess and does not form a boundary, is slop. Extract on the second occurrence, not the first.

Slop:

```ts
type SocketPath = string;

function joinBootstrapSocket(dir: string): SocketPath {
  return join(dir, "ces.sock");
}

export function discoverCes(): DiscoveryResult {
  const socketPath = joinBootstrapSocket(
    process.env["CES_BOOTSTRAP_SOCKET_DIR"] ?? "/run/ces-bootstrap",
  );
  return lookup(socketPath);
}
```

Unslopped:

```ts
export function discoverCes(): DiscoveryResult {
  const dir =
    process.env["CES_BOOTSTRAP_SOCKET_DIR"] ?? "/run/ces-bootstrap";
  return lookup(join(dir, "ces.sock"));
}
```

If CLI, CES, and assistant must agree on the path, the helper is the contract and stays (`resolveIpcEndpoint`, not a private `join` alias). Delete a helper when it is a private alias for a stdlib call used once.

### 11. Repeated arguments from the same import

When every caller of a function passes the same argument, and every caller computes that argument from the same imported function, the callee should own the import. Drop the parameter. Callers stop importing the producer.

This is not "the helper has a default." It is "the parameter is not a parameter." The value is not chosen at the call site. It is always the same producer, always invoked the same way.

Slop:

```ts
// foo.ts
import { generateInput } from "./generate-input.js";
import { helper } from "./helper.js";

const fooOutput = helper({ input: generateInput() });

// bar.ts
import { generateInput } from "./generate-input.js";
import { helper } from "./helper.js";

const barOutput = helper({ input: generateInput() });

// helper.ts
export function helper({
  input,
}: {
  input: ReturnType<typeof generateInput>;
}): Output {
  return use(input);
}
```

Unslopped:

```ts
// foo.ts
import { helper } from "./helper.js";

const fooOutput = helper();

// bar.ts
import { helper } from "./helper.js";

const barOutput = helper();

// helper.ts
import { generateInput } from "./generate-input.js";

export function helper(): Output {
  const input = generateInput();
  return use(input);
}
```

Applies to positional args the same way: `helper(generateInput())` at every site becomes `helper()`, with `generateInput()` inside.

Keep the parameter when any of these is true:

- At least one caller passes a different value, or builds the value without that import.
- Tests (or a public API) inject the argument as a seam.
- The callee must not import the producer (package boundary: assistant vs gateway vs skills vs meta).
- There is only one caller. Wait for the second site before moving the import.

### 12. Reusable export laundering

A helper's return object should contain values the helper computes or owns. Do not import a stable reusable export only to copy it into the return shape, forcing consumers to call the helper and destructure an alias for that export. Consumers should import the canonical export directly, and the helper should return only its actual output.

Slop:

```ts
// constants.ts
export const FOO = "foo";

// helper.ts
import { FOO } from "./constants.js";

export function helper() {
  const result = computeResult();
  return { foo: FOO, result };
}

// a.ts
import { helper } from "./helper.js";

const { foo, result } = helper();
useResult(result, foo);

// b.ts
import { helper } from "./helper.js";

const { foo, result } = helper();
useResult(result, foo);
```

Unslopped:

```ts
// helper.ts
export function helper() {
  const result = computeResult();
  return { result };
}

// a.ts
import { FOO } from "./constants.js";
import { helper } from "./helper.js";

const { result } = helper();
useResult(result, FOO);

// b.ts
import { FOO } from "./constants.js";
import { helper } from "./helper.js";

const { result } = helper();
useResult(result, FOO);
```

This remains slop when a consumer also needs other fields from `helper()`. Calling the helper for real output does not make it the owner of an unrelated reusable export. Remove the field from the helper's return type as well as from its implementation and consumers.

Keep the field when any of these is true:

- The value is computed, selected, transformed, or validated for that invocation.
- The return shape is a public, package, wire, or persisted contract that must stay stable.
- The field is an intentional test seam or dependency supplied by the helper's caller.
- The helper must capture a snapshot rather than expose a live or mutable binding.
- Importing the canonical export would cross a package boundary the consumer must not cross.

### 13. Handwritten unknown-data parsing

Structured values from unknown boundaries should be parsed as complete shapes with Zod before field access. This includes parsed JSON, HTTP response bodies, persisted blobs, and unknown tool or plugin inputs. Handwritten object, null, and array checks followed by `as Record<string, unknown>` or narrower property assertions are type-narrowing theater: the assertion tells the compiler to trust the value at the boundary where it is least trustworthy.

Slop:

```ts
const body: unknown = await response.json();
if (typeof body !== "object" || body === null || Array.isArray(body)) {
  return null;
}
const commit = (body as Record<string, unknown>).commit;
if (typeof commit !== "object" || commit === null) {
  return null;
}
const sha = (commit as Record<string, unknown>).sha;
return typeof sha === "string" ? sha : null;
```

Unslopped:

```ts
const CommitResponseSchema = z.object({
  commit: z.object({
    sha: z.string(),
  }),
});

const parsed = CommitResponseSchema.safeParse(await response.json());
return parsed.success ? parsed.data.commit.sha : null;
```

Define the complete structure the caller consumes, call `safeParse`, check `success`, and use `parsed.data`. Keep optional sibling metadata independently lenient when one malformed field must not invalidate another valid field. Do not replace one assertion with a different `as` assertion.

Keep deterministic handwritten checks for primitive values or domain rules after parsing, and preserve bespoke user-facing validation errors when a generic schema error would change the interface.

### 14. Reuse the authoritative boundary schema

Before defining a schema for unknown data, search the producer contract and nearby consumers for the schema that already owns that boundary. Parse with one source-of-truth schema. Do not add a second local partial schema just to inspect one field before parsing the complete contract. Parallel schemas create overlapping definitions, duplicate validation, and drift about which wire shapes are valid.

Slop:

```ts
const EnvelopeSchema = z.object({
  contact: z.unknown().optional(),
});

const envelope = EnvelopeSchema.safeParse(result);
if (!envelope.success || envelope.data.contact == null) {
  return undefined;
}
const parsed = GetContactResponseSchema.safeParse(result);
```

Unslopped:

```ts
const parsed = GetContactResponseSchema.safeParse(result);
if (!parsed.success) {
  return undefined;
}
```

If the producer legitimately returns more than one shape, fix or compose the authoritative contract so it represents those variants, then reuse it everywhere. Keep a separate schema only when it validates a genuinely different boundary or intentionally narrower public contract, not when it exists to preserve a logging distinction or another piece of local control-flow trivia.

---

## Out of scope

These look adjacent and are not Unslopping:

- Collapsing two similar but branching functions into one "clever" function.
- Rewording identifiers that already match the current design.
- Removing braces, turning `if` into a ternary, or shrinking a file for line count.
- Deleting migrations, compat shims that shipped clients still hit, or generated OpenAPI clients.
- Moving code across `assistant/` / `gateway/` / `skills/` / `meta/` to make a wrapper go away.
- Flagging the substitutions made in `assistant/src/tools/executor.ts` by vellum-assistant PR #43376 as slop, or applying them elsewhere by analogy. Those edits were deliberate, not an Unslopping pattern.

If a candidate fails the "same behavior, fewer hops" test, leave it and note it in the PR instead of forcing it.
