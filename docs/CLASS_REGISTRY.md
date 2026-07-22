# Class Registry

**Component ID:** core.class-registry  
**Status:** Active  
**Version:** 1.0.0  
**Last Updated:** 2026-07-22

---

## Overview

The Class Registry maintains a structured catalog of all class and type definitions in the codebase. While the Symbol Registry indexes every symbol generically, the Class Registry adds class-specific semantics: inheritance hierarchies, interface implementations, generic type parameters, and relationship metadata.

This registry serves two primary consumers: the Impact Analysis engine (which uses class relationships to compute change propagation) and the MERMAID_DIAGRAMS.md pipeline (which consumes relationship data to auto-generate UML class diagrams).

---

## ClassDefinition Schema

```typescript
interface ClassDefinition {
  name: string;
  kind: "class" | "interface" | "abstract_class" | "enum" | "type_alias";
  file: string;
  line: number;
  type_parameters?: string[];
  extends?: string[];
  implements?: string[];
  uses?: Array<{ type: string; relation: "property" | "parameter" | "return_type" | "composition" }>;
  members: Array<{
    name: string;
    kind: "method" | "property" | "constructor";
    visibility: "public" | "protected" | "private";
    type: string;
    static: boolean;
    line: number;
  }>;
  exports: boolean;
  deprecated: boolean;
}
```

| Field             | Type                          | Description                              |
|-------------------|-------------------------------|------------------------------------------|
| `name`            | `string`                      | Fully qualified class/type name          |
| `kind`            | `ClassKind`                   | Class, interface, abstract, enum, alias  |
| `file`            | `string`                      | Declaring file path                      |
| `line`            | `number`                      | Declaration line number                  |
| `type_parameters` | `string[]`?                   | Generic type params (e.g. `T`, `K`, `V`) |
| `extends`         | `string[]`?                   | Parent classes                           |
| `implements`      | `string[]`?                   | Implemented interfaces                   |
| `uses`            | `Array<TypeUse>`?             | Usage relationships to other types       |
| `members`         | `Array<MemberDefinition>`     | Methods, properties, constructors        |
| `exports`         | `boolean`                     | Exported from module                     |
| `deprecated`      | `boolean`                     | Marked with deprecation annotation       |

---

## Relationship Types

| Relation       | Direction       | Semantics                                  |
|----------------|-----------------|--------------------------------------------|
| `extends`      | child → parent  | Inheritance (single or multi)              |
| `implements`   | impl → interface| Contract fulfillment                        |
| `property`     | owner → type    | Member variable of a given type            |
| `parameter`    | method → type   | Method/constructor parameter type          |
| `return_type`  | method → type   | Return type annotation                     |
| `composition`  | owner → type    | Owned instance (strong lifetime coupling)  |

Relationships form a directed graph. Cycles are permitted for `uses`-category edges only; inheritance cycles are rejected at registration time.

---

## Integration with MERMAID_DIAGRAMS.md

The Class Registry is the data provider for automatic class diagram generation. The engine calls `class_registry.get_diagram_data(package?, depth?)` which returns structured classes and relationships, then renders a Mermaid `classDiagram` block. The diagram regenerates on class registration events (file save, module import, type change).

```typescript
interface DiagramData {
  classes: Array<{
    name: string; kind: string;
    members: Array<{ name: string; kind: string; visibility: string; type: string }>;
  }>;
  relationships: Array<{
    from: string; to: string;
    type: "extends" | "implements" | "composition" | "dependency";
    label?: string;
  }>;
}
```

---

## Interfaces

### `registry.register(def: ClassDefinition): void`
Insert or update a class record. Validates that `extends` targets exist as registered classes (warning on missing, configurable strict mode).

### `registry.lookup(name: string): ClassDefinition | null`
Resolve a class by fully qualified name.

### `registry.get_children(name: string): Array<ClassDefinition>`
Return all classes that directly extend or implement the given class.

### `registry.get_descendants(name: string, depth?: number): Array<ClassDefinition>`
Return the transitive closure of children. Default depth is unlimited.

### `registry.get_diagram_data(package?: string, depth?: number): DiagramData`
Return structured data for Mermaid class diagram generation.

### `registry.search(query: string): Array<ClassDefinition>`
Free-text search across class names, member names, and docstrings.

---

## Related Documents

- SYMBOL_REGISTRY.md — General-purpose symbol tracking
- FUNCTION_REGISTRY.md — Function-specific registry layer
- VARIABLE_REGISTRY.md — Environment variable and config tracking
- MERMAID_DIAGRAMS.md — Auto-generated diagram pipeline
- ARCHITECTURE_GUARDIAN.md — System-wide architectural enforcement
