# Secrets Management

> How API keys, tokens, passwords, and other sensitive credentials are
> stored and accessed in AI Dev OS.

## Overview

AI Dev OS never inlines secrets in configuration files or source code.
Every secret is stored in a dedicated secrets backend, encrypted at rest,
and retrieved on demand by the component that needs it. The system supports
multiple backends with a fallback chain so that it works on every platform
without mandatory cloud dependencies.

## Goals

- **Never inline secrets in config** — configuration files reference secrets
  by name; the value is resolved at runtime from a backend.
- **OS keychain integration** — primary backend is the platform-native
  credential store.
- **Scoped access** — each secret is associated with a scope (e.g. a
  workspace or provider) and can only be read by components that hold the
  corresponding capability.
- **Audit trail** — every secret access is recorded in the audit log
  (access time, actor, secret name, operation; the value is never logged).
- **Rotation support** — secrets can be rotated without restarting the
  daemon; providers automatically pick up the new value on the next
  credential check.

## Backends

| Priority | Backend | Platform | Details |
|---|---|---|---|
| 1 | OS keychain | macOS | Keychain via Security Framework |
| 1 | OS keychain | Windows | Credential Manager (wincred) |
| 1 | OS keychain | Linux | Secret Service (libsecret) |
| 2 | Encrypted file | All | `~/.aidevos/secrets/secrets.json` encrypted with age |
| 3 | Environment variables | All | Fallback — variables prefixed `AIDEVOS_SECRET_` |
| 4 | Cloud KMS | All | Optional — AWS KMS / GCP Cloud KMS / Azure Key Vault |

The backend chain is tried in priority order. The first backend that
succeeds for a given operation is used. Users can pin a specific backend
with `secrets.primary_backend` in their config.

## Secret Schema

Every secret is stored as a record with the following fields:

| Field | Type | Description |
|---|---|---|
| `name` | `string` | Unique identifier within its scope |
| `value` | `bytes` | Encrypted secret value (AES-256-GCM) |
| `scope` | `string` | Owning scope — `workspace:<id>`, `provider:<name>`, `global` |
| `created_at` | `timestamp` | When the secret was first stored |
| `updated_at` | `timestamp` | When the secret was last modified |
| `version` | `uint32` | Monotonically increasing version number |
| `metadata` | `map[string]string` | User-attached key-value pairs (e.g. `endpoint`, `region`) |

The `value` field is encrypted with a key derived from the workspace master
key (or a global master key for `global`-scoped secrets). The rest of the
record is stored in plaintext inside the backend so that listing and
scoping queries do not require decryption.

## Interfaces

```
secrets.set(name: string, value: SecretValue, scope?: string) -> Result
```

Stores or overwrites a secret. Increments the version. If `scope` is
omitted, defaults to `global`.

```
secrets.get(name: string, scope?: string) -> SecretValue
```

Retrieves and decrypts a secret. Returns `Err(SecretNotFound)` if the name
does not exist in the given scope. Returns `Err(PermissionDenied)` if the
caller's capability does not match the scope.

```
secrets.list(scope?: string) -> SecretMetadata[]
```

Returns metadata (all fields except `value`) for every secret in the given
scope, or all scopes if omitted.

```
secrets.delete(name: string, scope?: string) -> Result
```

Removes a secret from the backend. The record is purged immediately; the
audit log retains evidence of the deletion.

```
secrets.rotate(name: string, scope?: string) -> Result
```

Generates a new secret value using the provider's native rotation API
(where supported) or prompts the user for a new value. Updates the stored
record and increments the version. All active consumers are notified via
the event bus so they can re-authenticate with the new credential.

## Access Control

Access to secrets is gated by the RBAC system. Each actor (user, plugin,
automation) carries a set of capabilities. The following rules apply:

| Capability | Grants Access To |
|---|---|
| `secrets:read:global` | All `global`-scoped secrets |
| `secrets:read:workspace:<id>` | Secrets scoped to workspace `<id>` |
| `secrets:read:provider:<name>` | Secrets scoped to provider `<name>` |
| `secrets:write:*` | Set, delete, and rotate in any scope |
| `secrets:admin` | Bypass scope restrictions; list all secrets |

By default, only workspace-scoped actors can read secrets in their own
workspace. Providers read their own provider-scoped secrets. Plugins must
declare the secrets they need in their manifest; the user is prompted to
grant access on first use.

## CLI Usage

```
aidevos secrets set <name> [--scope <scope>] [--from-env <var>]
```

Prompts for a value (or reads from `--from-env`) and stores it.

```
aidevos secrets list [--scope <scope>]
```

Lists secret metadata for the given scope. Values are never displayed.

```
aidevos secrets delete <name> [--scope <scope>]
```

Removes a secret after confirmation.

```
aidevos secrets rotate <name> [--scope <scope>]
```

Rotates the secret value. For provider-managed secrets (e.g. API keys that
can be regenerated via the provider's API), the rotation is automatic. For
user-managed secrets, a new value is prompted interactively.

## Provider Integration

Each model provider adapter reads its API key from Secrets Management at
initialisation and on each credential check:

```
key = secrets.get("api_key", scope="provider:openai")
```

The credential check runs:
- On first connection to the provider
- Whenever a 401/403 response is received from the provider API
- When the event bus emits a `secret.rotated` event for the provider's
  scope

This means providers automatically pick up rotated keys without a daemon
restart.

Search providers, vector store providers, and plugin credentials follow
the same pattern.

## Failure Modes

| Failure | Behaviour | Recovery |
|---|---|---|
| OS keychain unavailable | Falls through to encrypted file backend | Install/start keychain daemon; migrate with `aidevos secrets migrate --to keychain` |
| Backend unreachable (cloud KMS) | Falls through to next backend in chain | Verify network connectivity and IAM permissions |
| Secret not found | `get` returns `Err(SecretNotFound)` | Check the name and scope; set the secret if missing |
| Permission denied | `get`/`set` returns `Err(PermissionDenied)` | Grant the required capability to the actor |
| Corrupted encrypted value | Decryption fails → `Err(IntegrityError)` | Delete and re-set the secret; restore from backup if available |
| All backends exhausted | `get` returns `Err(NoBackendAvailable)` | Configure at least one working backend |

## Observability

The secrets module exposes:

- `secrets_access_total{operation="set|get|delete|list|rotate", status="ok|error"}` —
  counter for every operation.
- `secrets_backend_active` — gauge (1 or 0) per backend indicating whether
  it is reachable.
- `secrets_rotation_events_total` — counter of completed rotations.

**No secret values ever appear in metric labels, logs, or trace attributes.**
Only the secret name (and only when metadata logging is enabled) and
operation type are recorded.

## Acceptance Criteria

1. A user can store a secret with `aidevos secrets set` and retrieve it
   from a provider configuration without ever writing the value to a file.
2. Secrets stored on macOS can be viewed in the Keychain Access app.
3. Listing secrets displays metadata but never the secret value.
4. Rotating a provider secret causes the provider adapter to pick up the
   new key within 5 seconds without a restart.
5. An actor without the required capability receives
   `Err(PermissionDenied)`.
6. When the OS keychain is locked, the encrypted file backend is used as
   fallback.

## Related Documents

- [Security Model](SECURITY_MODEL.md) — overall threat model and trust
  boundaries
- [Auth System](AUTH_SYSTEM.md) — RBAC capabilities and actor identities
- [Configuration](CONFIGURATION.md) — how configuration files reference
  secrets by name
- [Encryption](ENCRYPTION.md) — how secret values are encrypted at rest
