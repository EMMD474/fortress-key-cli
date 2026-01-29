# Fortress-Key CLI (fk)

`fk` is the command-line interface for **Fortress-Key**, a secure, offline-first password manager. The CLI allows you to initialize a local encrypted vault, manage secrets entirely offline, and optionally synchronize encrypted data with a remote server.

This document explains **how the fk CLI works**, its security model, and its command workflow.

---

## Core Design Principles

* **Offline-first**: The CLI works fully without an internet connection.
* **Zero-knowledge**: Secrets are encrypted locally; plaintext never leaves your machine.
* **Single source of authority**: A master password controls access to the vault.
* **Local-first storage**: A local SQLite database is the authoritative store.
* **Optional sync**: Remote storage is used only for synchronization and backup.

---

## Security Model

### Master Password

The **master password** is the only secret the user must remember.

It is used to:

* Derive a master key using a strong KDF (e.g. Argon2id)
* Decrypt the vault encryption key
* Unlock the vault for all read/write operations

There is **no separate local username or authentication password** for the CLI.

> If the vault is unlocked, the user is authenticated.

### Key Hierarchy (Conceptual)

```
Master Password
      ↓ (KDF)
Master Key
      ↓ (decrypts)
Vault Key
      ↓ (encrypts)
Secrets
```

* The master password is never stored.
* The master key is derived at runtime.
* The vault key is stored **encrypted** in the local database.

---

## Local Storage

### SQLite Vault

On initialization, `fk` creates a local **SQLite database** which stores:

* Encrypted vault key
* KDF parameters (salt, memory, iterations, etc.)
* Encrypted vault items (passwords, notes, secrets)
* Metadata required for sync (timestamps, versions, tombstones)

**Plaintext secrets are never written to disk.**

The SQLite database is the **authoritative source of truth** for the CLI.

---

## Application Lifecycle

### 1. Initialization

```bash
fk init
```

This command:

1. Creates the local SQLite database
2. Generates cryptographic salts and a vault key
3. Prompts the user to create a master password
4. Derives a master key using a KDF
5. Encrypts and stores the vault key

This step is required only once per device.

---

### 2. Unlocking the Vault

```bash
fk unlock
```

* Prompts for the master password
* Derives the master key
* Decrypts the vault key into memory
* Marks the vault as unlocked for the current session

If the vault is locked, **no CRUD operations are allowed**.

---

### 3. Managing Secrets

Once unlocked, the following commands are available:

```bash
fk add <name>
fk get <name>
fk edit <name>
fk remove <name>
fk list
```

All operations:

* Encrypt data locally using the vault key
* Store only ciphertext in SQLite
* Never expose plaintext outside process memory

---

### 4. Locking the Vault

```bash
fk lock
```

* Wipes sensitive keys from memory
* Returns the CLI to a locked state

Locking happens automatically when the process exits.

---

## Offline-First Behavior

The fk CLI is fully functional without internet access:

* Secrets can be added, read, edited, and removed offline
* All changes are recorded locally
* Each change updates metadata for later synchronization

No network connection is required for normal usage.

---

## Synchronization (Optional)

When connectivity is available, the CLI can synchronize with a remote server:

```bash
fk sync
```

Sync behavior:

* Only encrypted data is transmitted
* The server never sees plaintext or keys
* Conflicts are resolved using versioning or timestamps
* Deletions are propagated using tombstones

The remote database (e.g. Postgres) is a **sync target**, not a source of truth.

---

## Web Compatibility

The fk CLI and the Fortress-Key web app:

* Share the same encrypted data model
* Use identical cryptographic algorithms and formats
* Differ only in client implementation (Go vs Web Crypto API)

This allows seamless cross-device and cross-interface access.

---

## Threat Model (Summary)

The fk CLI is designed to protect against:

* Database compromise
* Server-side breaches
* Network interception
* Offline attacks on stored data

It does **not** protect against:

* A compromised operating system
* Malware with live process access
* Weak master passwords

---

## Command Summary

```text
fk init      Initialize a new local vault
fk unlock    Unlock the vault for the current session
fk lock      Lock the vault and wipe keys from memory
fk add       Add a new secret
fk get       Retrieve a secret
fk edit      Edit an existing secret
fk remove    Delete a secret
fk list      List stored secrets
fk sync      Synchronize with remote storage (optional)
```

---

## Philosophy

Fortress-Key CLI is designed around one principle:

> **Security through simplicity.**

By minimizing secrets, avoiding unnecessary authentication layers, and keeping encryption local, `fk` remains secure, predictable, and easy to reason about.

---
