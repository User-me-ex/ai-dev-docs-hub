# Variable Registry

**Component ID:** core.variable-registry  
**Status:** Active  
**Version:** 1.0.0  
**Last Updated:** 2026-07-22

---

## Overview

The Variable Registry maintains a complete inventory of environment variables and configuration parameters used across the system. Every variable — whether consumed at build time, startup, or runtime — must be declared here to be recognized by the configuration loader.

This registry ensures that no configuration key is used before it is defined, that deprecations are visible across the codebase, and that secret-bearing variables are never logged or leaked. It is the single source of truth for the Configuration subsystem and the Environment Variables initialization pipeline.

---

## VariableDefinition Schema

```typescript
interface VariableDefinition {
  name: string;
  type: "string" | "number" | "boolean" | "path" | "json" | "duration" | "enum";
  default?: string | number | boolean;
  description: string;
  scope: "build" | "startup" | "runtime" | "all";
  secret: boolean;
  required: boolean;
  enum_values?: string[];
  deprecated?: boolean;
  replaced_by?: string;
  source: "env" | "config_file" | "both";
}
```

| Field          | Type                                      | Description                                    |
|----------------|-------------------------------------------|------------------------------------------------|
| `name`         | `string`                                  | Uppercase snake-case variable name             |
| `type`         | `"string" \| "number" \| "boolean" \| "path" \| "json" \| "duration" \| "enum"` | Value type |
| `default`      | `string \| number \| boolean`?            | Default if not set                             |
| `description`  | `string`                                  | Purpose and usage notes                        |
| `scope`        | `"build" \| "startup" \| "runtime" \| "all"` | When the variable is loaded               |
| `secret`       | `boolean`                                 | If true, never log, dump, or expose in errors  |
| `required`     | `boolean`                                 | Startup must fail if unset                     |
| `enum_values`  | `string[]`?                               | Allowed values for `type: "enum"`              |
| `deprecated`   | `boolean`?                                | Scheduled for removal                          |
| `replaced_by`  | `string`?                                 | Use this variable instead                      |
| `source`       | `"env" \| "config_file" \| "both"`        | Where the value is resolved from               |

---

## Standard Environment Variables

| Variable                  | Type      | Default    | Secret | Required | Scope     | Description                              |
|---------------------------|-----------|------------|--------|----------|-----------|------------------------------------------|
| `AIDEVOS_HOME`            | path      | `~/.aidevos` | false  | true     | startup   | Root directory for all runtime data      |
| `AIDEVOS_CONFIG`          | path      | `$AIDEVOS_HOME/config` | false | false | startup | Configuration file directory       |
| `AIDEVOS_LOG_LEVEL`       | enum      | `info`     | false  | false    | startup   | Log verbosity: `debug`, `info`, `warn`, `error` |
| `AIDEVOS_LOG_DIR`         | path      | `$AIDEVOS_HOME/logs` | false | false | startup   | Log file output directory           |
| `AIDEVOS_PLUGIN_DIR`      | path      | `$AIDEVOS_HOME/plugins` | false | false | startup | Plugin installation directory       |
| `AIDEVOS_TEMP_DIR`        | path      | `$AIDEVOS_HOME/tmp` | false   | false    | runtime   | Temporary file storage                   |
| `AIDEVOS_CACHE_DIR`       | path      | `$AIDEVOS_HOME/cache` | false  | false    | runtime   | Persistent cache storage                 |
| `AIDEVOS_MAX_TOKENS`      | number    | `4096`     | false  | false    | runtime   | Maximum LLM tokens per request           |
| `AIDEVOS_TIMEOUT_SECS`    | number    | `30`       | false  | false    | runtime   | Default operation timeout                |
| `AIDEVOS_SECRET_KEY`      | string    | —          | true   | true     | startup   | Master encryption key                    |
| `AIDEVOS_API_KEY`         | string    | —          | true   | false    | runtime   | External API authentication key          |
| `AIDEVOS_ENABLE_METRICS`  | boolean   | `true`     | false  | false    | runtime   | Enable Prometheus metrics endpoint       |
| `AIDEVOS_METRICS_PORT`    | number    | `9090`     | false  | false    | runtime   | Metrics server listen port               |
| `AIDEVOS_MCP_CONFIG`      | json      | `{}`       | false  | false    | startup   | MCP server connection configuration      |
| `AIDEVOS_DEV_MODE`        | boolean   | `false`    | false  | false    | startup   | Development-mode features and verbose logging |

---

## Integration with Configuration

The Configuration subsystem uses the Variable Registry as its schema for loading and validating config:

1. **Load** — Config loader iterates all `source: "env"` and `source: "both"` variables, reads them from environment, and applies defaults.
2. **Validate** — Each loaded value is type-checked against the variable's `type` field. Enum values are checked against `enum_values`. Required variables that are unset and lack a default raise `ConfigValidationError`.
3. **Coerce** — Values are cast to their declared types (e.g. `"true"` → `true` for booleans, `"30"` → `30` for numbers).
4. **Mask** — Variables with `secret: true` have their values replaced with `"****"` in all logs, error messages, and debug dumps.

---

## Validation Rules

- **Unknown variables** — Loading an environment variable not present in the registry emits a warning but does not fail. This allows forward-compatibility with external tooling.
- **Type mismatch** — If a value cannot be coerced to its declared type, the loader raises `TypeCoercionError` with the variable name and expected type.
- **Missing required** — Startup aborts if a required variable has no value and no default.
- **Deprecated usage** — Reading a variable marked `deprecated: true` emits a deprecation warning with the `replaced_by` target.

---

## Related Documents

- SYMBOL_REGISTRY.md — General-purpose symbol tracking
- FUNCTION_REGISTRY.md — Function-specific registry layer
- CLASS_REGISTRY.md — Class/type definition catalog
- CONFIGURATION.md — Configuration loading and resolution
- ARCHITECTURE_GUARDIAN.md — System-wide architectural enforcement
