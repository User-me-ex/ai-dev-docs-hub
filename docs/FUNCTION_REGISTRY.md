# Function Registry

**Component ID:** core.function-registry  
**Status:** Active  
**Version:** 1.1.0  
**Last Updated:** 2026-07-22

---

## Overview

The Function Registry is a specialized catalog of every callable function available to the system — including built-in kernel functions, MCP tool invocations, plugin hooks, and agent-defined callables. Unlike the general-purpose Symbol Registry, the Function Registry enriches each entry with execution metadata: input/output schemas, cost estimates, backend routing hints, and observability labels.

This registry is the single source of truth for tool-calling dispatch, function discovery (both for human developers and LLM agents), and cost-aware scheduling. Every function that can be invoked — whether by a user, an agent, or another function — must be registered here before it becomes available at runtime.

---

## FunctionDefinition Schema

Each registered function conforms to `FunctionDefinition`:

| Field          | Type               | Description                                                 |
|----------------|--------------------|-------------------------------------------------------------|
| `name`         | `string`           | Unique dot-separated name (e.g. `fs.read_file`)             |
| `description`  | `string`           | Human-readable purpose statement                            |
| `input_schema` | `JSONSchema`       | JSON Schema v2020-12 describing valid arguments             |
| `output_schema`| `JSONSchema`       | JSON Schema describing the return value                     |
| `backend`      | `"local" \| "remote" \| "mcp" \| "plugin"` | Execution environment |
| `cost`         | `CostTier`         | Enum: `free`, `cheap`, `moderate`, `expensive`              |
| `timeout_ms`   | `number`           | Maximum allowed execution time                              |
| `idempotent`   | `boolean`          | Safe to re-call without side effects                        |
| `tags`         | `Array<string>`    | Classification tags (e.g. `filesystem`, `network`, `dangerous`) |

```typescript
type CostTier = "free" | "cheap" | "moderate" | "expensive";

interface FunctionDefinition {
  name: string;
  description: string;
  input_schema: Record<string, unknown>;
  output_schema: Record<string, unknown>;
  backend: "local" | "remote" | "mcp" | "plugin";
  cost: CostTier;
  timeout_ms: number;
  idempotent: boolean;
  tags: string[];
}
```

---

## Built-in Functions

The following functions are registered at startup:

| Name                      | Backend   | Cost       | Idempotent | Description                           |
|---------------------------|-----------|------------|------------|---------------------------------------|
| `fs.read_file`            | local     | free       | true       | Read file contents                    |
| `fs.write_file`           | local     | cheap      | true       | Write content to file                 |
| `fs.list_directory`       | local     | free       | true       | List directory entries                |
| `code.search`             | local     | cheap      | true       | Grep/glob across codebase             |
| `code.edit`               | local     | moderate   | false      | Apply structured edit to a file       |
| `code.run_tests`          | local     | expensive  | false      | Execute test suite                    |
| `llm.chat`                | remote    | expensive  | false      | Send prompt to LLM                    |
| `llm.embed`               | remote    | moderate   | true       | Generate text embeddings              |
| `mcp.list_tools`          | mcp       | free       | true       | Enumerate MCP tools                   |
| `mcp.call_tool`           | mcp       | cheap      | false      | Invoke an MCP server tool             |
| `network.http_get`        | remote    | cheap      | true       | Fetch URL via GET                     |
| `network.http_post`       | remote    | cheap      | false      | POST to URL                           |
| `secrets.get`             | local     | free       | true       | Retrieve a secret by key              |
| `plugin.invoke`           | plugin    | moderate   | false      | Call a registered plugin hook         |

---

## Registration Flow

1. **Declaration** — A function is defined in source code with an explicit or inferred schema.
2. **Extraction** — A build-time or runtime extractor parses the definition and constructs a `FunctionDefinition`.
3. **Validation** — The registry validates uniqueness of `name`, well-formedness of schemas, and enum values for `backend` and `cost`.
4. **Registration** — The definition is inserted into the in-memory registry and persisted to the symbol store.
5. **Publishing** — A registry change event is emitted, triggering cache invalidation and UI updates.

Registration is idempotent: re-registering the same `name` with identical schemas is a no-op. Conflicts (same name, different schema) raise `RegistryConflictError`.

---

## Discovery for MCP / Plugin Functions

MCP server tools and plugin functions are discovered dynamically rather than declared at build time. The flow differs:

- **MCP:** On MCP server connection, the registry calls `mcp.list_tools` and registers each returned `Tool` as a `FunctionDefinition` with `backend: "mcp"`. If an MCP server disconnects, its tools are removed.
- **Plugin:** Plugin manifests declare exported hooks. The registry reads each manifest at plugin load time, validates schemas, and registers them with `backend: "plugin"`. Plugin functions are unregistered on plugin unload.

Dynamic registrations are tagged with a `source` metadata field to enable rollback on disconnection.

---

## Interfaces

### `registry.register(fn: FunctionDefinition): void`
Insert or update a function. Throws on schema conflict.

### `registry.lookup(name: string): FunctionDefinition | null`
Resolve a function by name.

### `registry.find_by_tag(tag: string): Array<FunctionDefinition>`
List all functions matching a given tag.

### `registry.filter(criteria: Partial<FunctionDefinition>): Array<FunctionDefinition>`
General-purpose query by any field.

### `registry.list_by_backend(backend: string): Array<FunctionDefinition>`
Return all functions for a specific execution backend.

---

## Integration with Tool Calling

The Tool Calling subsystem queries the registry before dispatch:

1. Receives a function name and arguments.
2. Calls `registry.lookup(name)` to retrieve the definition.
3. Validates arguments against `input_schema`.
4. Routes to the correct backend using the `backend` field.
5. Respects `timeout_ms` as the deadline.
6. Records execution cost from the `cost` field in the observability pipeline.

---

## Observability

Every function call is traced with:

- `function.name` — the registered name
- `function.cost` — cost tier
- `function.backend` — execution backend
- `duration_ms` — wall-clock execution time
- `success` — boolean outcome
- `error` — error message on failure

Aggregated metrics (call count, p50/p95/p99 latency, error rate, total cost) are exposed via Prometheus at `:9090/metrics`.

---

## Related Documents

- SYMBOL_REGISTRY.md — General-purpose symbol tracking
- CLASS_REGISTRY.md — Class/type definition catalog
- VARIABLE_REGISTRY.md — Environment variable and config tracking
- ARCHITECTURE_GUARDIAN.md — System-wide architectural enforcement
