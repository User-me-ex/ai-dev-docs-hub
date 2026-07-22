# Encryption

> Encryption at rest and in transit for AI Dev OS.

## Overview

AI Dev OS employs a multi-layer encryption strategy that ensures data is
protected at every stage of its lifecycle. Each workspace receives its own
cryptographic boundary: derived keys are scoped per-workspace and per-data-
category so that compromise of one key does not cascade to others. All
encryption is performed client-side before any data leaves the process
boundary; the system never sends plaintext to storage or to the network
layer.

## Goals

- **Per-workspace encryption keys** — data from different workspaces is
  cryptographically isolated.
- **AES-256-GCM for data at rest** — authenticated encryption with a
  ­256-bit key provides confidentiality and integrity for stored data.
- **TLS 1.3 for data in transit** — all remote communication uses the
  latest TLS standard with forward secrecy.
- **Key rotation without downtime** — keys can be rotated while the system
  is running; old keys are retained for decryption of existing data until
  it is re-encrypted.
- **Emergency key access** — designated recovery keys allow authorized
  administrators to access encrypted data when a user's primary key is
  unavailable.

## At-Rest Encryption

| Data Category | Mechanism | Key Scope |
|---|---|---|
| Memory records | AES-256-GCM per record | Derived from workspace master key |
| Configuration files | [age](https://age-encryption.org/) v1.X | Per-file X25519 identity |
| SQLite databases | WAL encryption via `PRAGMA cipher` | Workspace master key |
| Audit log | Hash-chained, each entry signed with HMAC-SHA256 | Audit signing key |

**Key derivation.** Each workspace has a 256-bit master key stored in the OS
keychain. Data-category sub-keys are derived as:

    sub_key = HKDF-SHA384(master_key, salt=category_name, info="aidevos/v1")

This ensures that reading the SQLite cipher key does not reveal the master
key or keys for other categories.

**SQLite WAL encryption.** When the SQLite journal is in WAL mode, the
`cipher` pragma transparently encrypts every page written to the WAL file.
Pages are decrypted on read. The cipher key is held only in process memory
and is zeroed on workspace close.

## In-Transit Encryption

| Channel | Mechanism |
|---|---|
| Local IPC (Unix domain sockets) | Socket file permissions (`0600`); no network exposure |
| Local IPC (Windows named pipes) | ACL restricted to the owning user |
| Remote API | TLS 1.3 with X.509 certificates; cipher suite `TLS_AES_256_GCM_SHA384` |
| MCP connections | TLS 1.3; optional mutual TLS (mTLS) when both peers present certificates |
| WebSocket (real-time events) | WSS over TLS 1.3 |

**Certificate validation.** Remote API and MCP connections validate the
peer certificate against a configurable CA bundle. The default bundle is
the system trust store. In development, self-signed certificates can be
pinned via fingerprint.

## Key Management

**Master key storage.** The workspace master key is stored in the platform
keychain:

- **macOS:** macOS Keychain (AccessGroup restricted to the application)
- **Windows:** Credential Manager / DPAPI
- **Linux:** Secret Service (D-Bus) via libsecret; fallback to an encrypted
  file at `~/.aidevos/keys/` if no keychain daemon is running

**Derived-key hierarchy.**

    Master Key (256-bit)
    ├── Storage Key   → AES-256-GCM for memory records
    ├── Config Key    → age identity for config files
    ├── Database Key  → SQLite cipher key
    ├── Audit Key     → HMAC-SHA256 for audit log integrity
    └── Transport Key → ephemeral (per-session ECDHE)

**Key rotation protocol.**

1. Administrator calls `rekey(workspace_id, new_master_key)`.
2. The system writes the new master key to the OS keychain and retains the
   old key in a "grace" slot.
3. Background workers re-encrypt each data category using the new derived
   sub-keys. Data re-encrypted with the new key is tagged with a key
   version number.
4. Once all data has been re-encrypted, the grace slot is purged.
5. Reads during rotation transparently try the current key, then fall back
   to the grace key if the data carries an older version tag.

**Emergency key access.** An organisation can configure an emergency
recovery public key (X25519). When enabled, every workspace master key is
also wrapped with this recovery public key and stored alongside the primary
encrypted blob. Authorised administrators can unwrap it with the
corresponding private key.

## Encryption Algorithms

| Use Case | Algorithm | Key Size | Mode / Details |
|---|---|---|---|
| Memory records | AES-256-GCM | 256 bit | GCM nonce: 12 bytes random, tag: 16 bytes |
| Config files | X25519 + ChaCha20-Poly1305 (age) | 256 bit | Curve25519 key agreement |
| SQLite pages | AES-256-CBC | 256 bit | CBC + HMAC-SHA256 per page; random IV |
| Audit log entries | HMAC-SHA256 | 256 bit | Keyed-hash for integrity chaining |
| TLS session | TLS_AES_256_GCM_SHA384 | 256 bit | TLS 1.3 mandatory cipher suite |
| Ephemeral key agreement | X25519 (ECDHE) | 256 bit | Per-session forward secrecy |

## Interfaces

```
encrypt(data: bytes, context: EncryptionContext) -> SealedData
```

Encrypts `data` for the given `context` (workspace + data category).
Returns a `SealedData` struct containing the ciphertext, nonce, key version,
and KDF salt.

```
decrypt(sealed_data: SealedData, context: EncryptionContext) -> bytes
```

Decrypts a `SealedData` value. Automatically selects the correct key based
on the embedded key version and the context.

```
rekey(workspace_id: WorkspaceID, new_key: SecretKey) -> Result
```

Initiates a rotation to `new_key` for the given workspace. Returns
immediately; rotation completes asynchronously. Poll `key_status` to
monitor progress.

```
key_status(workspace_id: WorkspaceID) -> KeyStatus
```

Returns the current key version, the number of records still encrypted
with the previous key, and the estimated time until rotation completes.

## Failure Modes

| Failure | Behaviour | Recovery |
|---|---|---|
| Key unavailable | `decrypt` returns `Err(KeyNotFound)` | Re-authenticate to unlock the OS keychain; restore from backup if the keychain was reset |
| Ciphertext corruption | GCM authentication tag mismatch → `Err(IntegrityError)` | Restore from the most recent backup; corruption is logged and alarmed |
| Rotation failure (partial re-encrypt) | Alert is raised; old key remains in grace slot | Administrator can retry rotation or force-purge the grace slot (data encrypted with the old key becomes unrecoverable) |
| Master key lost without recovery key | All data in the workspace is permanently unrecoverable | No recovery possible — the system is designed so that AI Dev OS cannot bypass its own encryption |
| OS keychain daemon not running (Linux) | Falls back to encrypted file at `~/.aidevos/keys/` | Install and start a Secret Service provider (`gnome-keyring`, `keepassxc`) |

## Observability

The encryption module exposes the following metrics:

- `encryption_operations_total{operation="encrypt|decrypt|rekey", status="ok|error"}` —
  counters for every cryptographic operation.
- `encryption_key_version{workspace}` — gauge indicating the active key
  version per workspace.
- `encryption_rotation_remaining{workspace}` — gauge reporting how many
  records still need re-encryption during a rotation.
- `encryption_integrity_errors_total` — counter that triggers an alert when
  it increments, indicating possible data corruption or tampering.

All metrics are safe to export: no key material or plaintext ever appears
in metric labels.

## Acceptance Criteria

1. A workspace can be created and all data written to disk is encrypted;
   reading the raw SQLite or config file without the key yields only
   ciphertext.
2. Two workspaces produce different ciphertexts for the same plaintext
   record (key isolation).
3. TLS 1.3 handshake succeeds between two AI Dev OS nodes. TLS 1.2 or
   below is rejected.
4. Key rotation completes for a workspace with 10 000 records without
   downtime and without data loss.
5. Corrupting a single byte in a stored ciphertext causes `decrypt` to
   return `Err(IntegrityError)`.
6. Removing the OS keychain entry and restarting the daemon produces a
   clear error message guiding the user to re-authenticate or restore
   their key.

## Related Documents

- [Security Model](SECURITY_MODEL.md) — overall threat model and trust
  boundaries
- [Secrets Management](SECRETS_MANAGEMENT.md) — how API keys and tokens are
  stored separately from workspace data
- [Data Retention](DATA_RETENTION.md) — how encrypted data is pruned and
  expired
