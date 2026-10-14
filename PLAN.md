# EnvVault: VS Code Extension Plan

Working name: **EnvVault**. Encrypted, versioned, local-first sync of `.env` files across machines via a shared folder (OneDrive, NAS, SMB share).

## 1. Goals

- Push the current `.env` to an encrypted vault in a user-chosen shared folder.
- Pull the latest version on another machine.
- Keep a version history and allow restoring any version.
- Merge divergent edits from two machines per key, not per line.
- No server, no account, no Git dependency.

## 2. Non-Goals (v1)

- Team permissions, auditing, or role-based access.
- Cloud-hosted vault or hosted sync service.
- Secret injection at runtime (that is what dotenvx, Doppler, and `op` do).
- Multiple environment files in one pass (`.env.local`, `.env.production`). Planned for v1.1 once the single-file flow is solid.

## 3. Tech Stack and Conventions

- TypeScript (strict mode), Google TypeScript Style Guide, Tailwind not applicable (no webview UI in v1).
- Build: `esbuild` bundle, `vsce` for packaging, `vitest` for tests.
- Runtime deps kept minimal: Node `crypto` (AES-256-GCM, scrypt), `node-diff3` (text merge fallback), `uuid` or `crypto.randomUUID()`.
- All variables explicitly typed, camelCase, descriptive names.
- Functions return a `Result<T, VaultError>` that callers must check. No silent catches, no thrown strings.
- Error messages state what failed, on which path or version, and the likely cause.
- JSON for every on-disk format. Every record has a UUID `id`, `createdAt`, and `updatedAt` where mutable.

```ts
type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };

interface VaultError {
  code: VaultErrorCode;
  message: string; // Descriptive, e.g. "Cannot decrypt version 3f2a...: wrong passphrase or corrupted file".
  path?: string;
  cause?: unknown;
}
```

## 4. Architecture

```
src/
  extension.ts          # activate(), command registration only
  commands/             # one file per command (push, pull, restore, diff, init)
  vault/
    vaultStore.ts       # read/write version files in the shared folder
    versionGraph.ts     # parent links, heads, common ancestor lookup
    manifest.ts         # vault.json (immutable settings)
  crypto/
    keyDerivation.ts    # scrypt passphrase -> key
    cipher.ts           # AES-256-GCM encrypt/decrypt
    keyCache.ts         # VS Code SecretStorage wrapper
  env/
    envParser.ts        # raw text <-> ordered key/value entries
    envMerge.ts         # three-way per-key merge
  project/
    projectIdentity.ts  # resolve projectId for the open workspace
  ui/
    prompts.ts          # input boxes, quick picks, conflict picker
    diffView.ts         # wraps vscode.diff with temp virtual documents
  util/
    result.ts
    logger.ts           # OutputChannel logger, never logs secret values
test/
```

Layers: `commands` call `vault`, `crypto`, `env`, `ui`. Nothing in `vault`, `crypto`, or `env` imports `vscode`, so they are unit testable in plain Node.

## 5. Vault Layout (Sync-Safe by Design)

The core rule: **no file is ever modified by two machines.** Versions are immutable, one file each. This avoids OneDrive "conflicted copy" files and SQLite-on-network-drive corruption.

```
<vaultRoot>/
  vault.json                          # written once at init: formatVersion, kdf params, salt, keyCheck blob
  <projectId>/
    <createdAtIso>_<versionId>.json   # one immutable file per version
```

Version file (JSON):

```json
{
  "formatVersion": 1,
  "versionId": "uuid",
  "projectId": "uuid",
  "parentVersionIds": ["uuid"],
  "createdAt": "2026-10-12T14:03:22.000Z",
  "machineLabel": "desktop-office",
  "nonce": "base64",
  "ciphertext": "base64",
  "authTag": "base64"
}
```

- Plaintext inside the cipher: raw `.env` text plus a SHA-256 content hash. Raw text is preserved so comments, ordering, and quoting survive.
- Header fields are authenticated as AES-GCM additional data, so tampering with parents or timestamps is detected.
- Parent links form a small DAG (same idea as Git commits). **Heads** are versions with no children.
  - One head: the vault is linear; Pull fast-forwards.
  - Multiple heads: machines diverged; Pull triggers a merge and writes a merge version with two parents.
- Parent IDs and timestamps are plaintext metadata. Acceptable trade-off for history listing without decrypting; flagged in the security notes.

## 6. Cryptography and Key Management

- KDF: scrypt (Node built-in) with parameters stored in `vault.json`. Argon2id is a possible upgrade if a trustworthy dependency is justified.
- Cipher: AES-256-GCM, fresh random 12-byte nonce per version.
- `vault.json` holds a `keyCheck` blob (known plaintext encrypted with the derived key) so a wrong passphrase fails fast with a clear message instead of failing on the first real version.
- Derived key cached per machine in VS Code `SecretStorage` (OS keychain). Passphrase itself is never stored.
- Recovery: "Export recovery key" command writes the derived key as a printable string for offline storage. Without it, a lost passphrase means lost data; this is stated in the README.
- Passphrase rotation: re-encrypt is possible but deferred (v2). Versions are immutable, so rotation means writing a new vault.

## 6a. Project Identity

Folder paths differ between machines, so identity cannot be path-based.

1. On first Push, generate a UUID `projectId` and write `.vscode/envvault.json` (`{ "projectId": "...", "vaultRoot": null }`). The user may commit this file; it contains no secrets.
2. Optional fallback: match by hashed Git remote URL if `.vscode/envvault.json` is absent.
3. `vaultRoot` is a per-machine user setting (`envvault.vaultRoot`), not stored in the project, since the OneDrive path differs per machine.

## 7. Commands

| Command | Behavior |
|---|---|
| `EnvVault: Initialize Vault` | Pick shared folder, set passphrase, write `vault.json`, cache key. |
| `EnvVault: Unlock Vault` | Enter passphrase on a new machine; verify against `keyCheck`. |
| `EnvVault: Push .env` | Parse, skip if hash equals current head, otherwise write new version with head as parent. |
| `EnvVault: Pull Latest` | Back up local `.env` to `.env.bak.<timestamp>`, fast-forward or merge, write result. |
| `EnvVault: Show History` | Quick pick of versions (time, machine, key-count delta). |
| `EnvVault: Restore Version` | Pick a version, back up local file, write it, optionally push as a new version. |
| `EnvVault: Diff With Vault` | Open `vscode.diff` between local `.env` and a vault version. |
| `EnvVault: Export Recovery Key` | Show derived key once for offline storage. |

Diff output masks values by default (`KEY=••••`) with a "reveal" toggle, so screen shares do not leak secrets.

## 8. Merge Algorithm

Input: base (lowest common ancestor), ours, theirs. Each parsed to an ordered map of key to `{ value, rawLine, leadingComments }`.

Per key:

| Base | Ours | Theirs | Result |
|---|---|---|---|
| same | same | same | keep |
| same | changed | same | take ours |
| same | same | changed | take theirs |
| absent | added | absent | keep ours |
| absent | absent | added | keep theirs |
| present | deleted | same | delete |
| present | deleted | changed | **conflict** |
| any | changed | changed differently | **conflict** |

- Conflicts surface in a quick pick per key: keep local, keep vault, or edit manually.
- Output order follows "ours" with new keys appended, to keep diffs stable.
- Parser edge cases to cover with tests: quoted multi-line values, `export` prefix, inline comments, blank lines, duplicate keys, CRLF vs LF, BOM, values containing `=` and `#`.
- Fallback: if parsing fails on either side, offer a whole-file three-way merge via `node-diff3` with conflict markers.

## 9. Error Handling Rules

- Every filesystem and crypto call returns a `Result`; no bare `try/catch` that swallows.
- On failure: log details to the OutputChannel (never values), show a descriptive notification, leave local `.env` untouched.
- Writes to `.env` are atomic: write temp file in the same directory, then rename. A backup always exists before overwrite.
- If the shared folder is unreachable (NAS offline, OneDrive not signed in), report the exact path and operation attempted. Push and Pull do not partially apply.
- Incomplete or zero-byte version files (mid-sync OneDrive) are detected by JSON parse and auth-tag failure, reported as "version file may still be syncing", and skipped rather than treated as corruption.

## 10. Security Checklist

- [ ] Warn if `.env` is not covered by `.gitignore`.
- [ ] Never log or telemetry secret values or key material.
- [ ] No network calls from the extension.
- [ ] Zero runtime dependencies that touch crypto beyond Node built-ins.
- [ ] `SECURITY.md` documents what is encrypted, what is plaintext metadata, and the threat model (protects against a compromised cloud drive or NAS, not a compromised dev machine).
- [ ] Constant-time comparison where hashes or tags are compared manually.
- [ ] Release artifacts built from a tagged commit with a published checksum.

## 11. Testing

- Unit (vitest, no VS Code): parser, merge table, cipher round trip, wrong-key failure, tamper detection, version graph (linear, diverged, merge-of-merge).
- Integration: simulate two machines as two temp dirs sharing one vault dir; interleave pushes and pulls.
- Sync fault injection: zero-byte file, truncated JSON, duplicate version IDs.
- Extension host tests (`@vscode/test-electron`) for command wiring only.

## 12. Milestones

1. **M1: Core library (2 to 3 days).** Result type, cipher, KDF, parser, vault store, linear versions, unit tests.
2. **M2: Extension MVP (2 days).** Init, Unlock, Push, Pull (fast-forward only), History, Restore, atomic writes and backups.
3. **M3: Merge (2 to 3 days).** Version DAG heads, three-way merge, conflict picker, diff view with masking.
4. **M4: Hardening (2 days).** `.gitignore` warning, recovery key export, sync fault handling, two-machine manual test on the desktop and laptop.
5. **M5: Publish (1 to 2 days).** README, SECURITY.md, icon, Marketplace and Open VSX listing, GitHub Sponsors link.

## 13. Open Questions for Discussion

1. One vault per user with many projects, or one vault per project? (Current plan: one vault, many project folders, shared passphrase.)
2. Should `projectId` live in a committed `.vscode/envvault.json`, or stay out of the repo entirely and rely on a per-machine mapping?
3. Plaintext parent IDs and timestamps: acceptable, or encrypt the whole graph and give up cheap history listing?
4. Auto-push on save and auto-pull on workspace open, or strictly manual commands? (Current plan: manual in v1; opt-in auto later.)
5. Multiple env files (`.env.local`, `.env.test`) in v1 or v1.1?
6. Licensing and monetization: MIT plus optional sponsor link, or a paid tier for extras (unlimited projects, team vaults)?
7. Support Open VSX (VSCodium, Cursor, Windsurf) from day one?

## 14. Risks

| Risk | Mitigation |
|---|---|
| Lost passphrase means lost data | Recovery key export, clear README warning |
| Cloud sync delays produce partial files | Immutable files, validation, "still syncing" handling |
| Merge bug overwrites a good `.env` | Mandatory backup before every overwrite, tests on merge table |
| Crowded market (dotenvx, SOPS, secret managers) | Position as local-first with no server or Git needed; treat as personal tool first |
| Trust barrier for a secrets tool | Open source, small dependency surface, SECURITY.md |
