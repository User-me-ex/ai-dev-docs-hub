# Symbol Registry

**Component ID:** core.symbol-registry  
**Status:** Active  
**Version:** 1.0.0  
**Last Updated:** 2026-07-22

---

## Overview

The Symbol Registry provides a centralized index of all named symbols (functions, classes, variables, types, interfaces) declared across the codebase. It is the foundation for impact analysis, dependency resolution, and cross-module navigation.

Without a symbol registry, the system has no way to answer questions like "which files depend on this function?" or "if I change this class, what breaks?" — making refactoring, deletion, and renaming dangerous operations. The registry solves this by maintaining an always-up-to-date, queryable map of every symbol and its relationships.

The registry is decoupled from any single parser or language. Language-specific extractors (Python, TypeScript, Rust, Go) push symbol records into the same schema, enabling unified analysis across polyglot projects.

---

## Symbol Schema

Each registered symbol is stored as a `SymbolRecord`:

| Field       | Type               | Description                                        |
|-------------|--------------------|----------------------------------------------------|
| `name`      | `string`           | Fully qualified symbol name (e.g. `auth.login`)    |
| `kind`      | `SymbolKind`       | Enum: `function`, `class`, `variable`, `type`, `interface`, `enum`, `method`, `property`, `parameter` |
| `file`      | `string`           | Absolute path to the declaring file                |
| `line`      | `number`           | 1-indexed line number of declaration               |
| `column`    | `number`           | 0-indexed column number                            |
| `exports`   | `boolean`          | Whether the symbol is exported from its module     |
| `imports`   | `Array<ImportRef>` | Symbols this file imports, with source paths       |
| `docstring` | `string?`          | Optional doc comment extracted from source         |
| `tags`      | `Array<string>`    | Optional classification tags (e.g. `deprecated`, `internal`) |

An `ImportRef` is defined as:

```typescript
interface ImportRef {
  source: string;       // Module path or package name
  importedName: string; // Name as used locally
  exportedName: string; // Name in the source module
}
```

---

## Interfaces

### `registry.register(symbol: SymbolRecord): void`

Inserts or updates a symbol. If a symbol with the same `name` and `file` already exists, it is overwritten. The registry re-indexes all import edges after registration.

```typescript
registry.register({
  name: "auth.verify_token",
  kind: "function",
  file: "/src/auth/tokens.ts",
  line: 42,
  column: 0,
  exports: true,
  imports: [
    { source: "jsonwebtoken", importedName: "verify", exportedName: "verify" }
  ]
});
```

### `registry.lookup(name: string, scope?: string): SymbolRecord | null`

Looks up a symbol by fully qualified name. An optional `scope` parameter scopes the search to a specific module. Returns `null` if not found.

### `registry.find_references(name: string, options?: FindReferencesOptions): Array<SymbolReference>`

Returns every location where a symbol is referenced (not declared). Each result includes the referencing file, line, column, and the kind of reference (direct import, dynamic import, type reference, string reference).

```typescript
interface SymbolReference {
  file: string;
  line: number;
  column: number;
  kind: "import" | "call" | "type" | "property" | "string";
}
```

### `registry.dependency_graph(file?: string, depth?: number): DependencyGraph`

Returns a full or scoped dependency graph. When `file` is omitted, returns the graph for the entire codebase. The `depth` parameter limits transitive resolution.

```typescript
interface DependencyGraph {
  nodes: Array<{ name: string; file: string; kind: SymbolKind }>;
  edges: Array<{ from: string; to: string; kind: "imports" | "calls" | "extends" | "implements" }>;
}
```

---

## Integration with Impact Analysis

The Symbol Registry is the primary data source for the Impact Analysis engine. When a file changes, impact analysis:

1. Calls `registry.lookup()` for each symbol in the changed file.
2. Calls `registry.find_references()` to discover all dependents.
3. Calls `registry.dependency_graph()` to compute the transitive closure of affected files.
4. Renders the result as a ranked list of affected symbols and files.

---

## Integration with Obsidian Graph Engine

The Obsidian Graph Engine consumes `dependency_graph()` output to render interactive dependency visualizations. Nodes are symbols (color-coded by `kind`), and edges represent import, call, or inheritance relationships. The graph supports drill-down: clicking a node fetches `find_references()` for that symbol.

---

## Failure Modes

| Mode                | Cause                                  | Effect                                        | Mitigation                              |
|---------------------|----------------------------------------|-----------------------------------------------|-----------------------------------------|
| Stale index         | File updated outside the watcher       | References point to outdated declarations     | File-system watcher reconciliation      |
| Ambiguous symbol    | Two modules export the same name       | `lookup()` returns arbitrary match            | Require fully-qualified names           |
| Circular import     | A imports B imports A                  | `dependency_graph()` infinite loop            | Depth limit + cycle detection           |
| Missing parser      | File in unsupported language           | Symbol silently omitted                       | Warning log for unhandled extensions    |
| Large codebase      | >100K symbols                          | Slow query times                              | Lazy-loading, paginated `find_references` |

---

## Related Documents

- FUNCTION_REGISTRY.md — Function-specific registry layer
- CLASS_REGISTRY.md — Class/type-specific registry layer
- VARIABLE_REGISTRY.md — Environment variable and config tracking
- ARCHITECTURE_GUARDIAN.md — System-wide architectural enforcement
