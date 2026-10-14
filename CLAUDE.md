# CLAUDE.md: EnvLedger

Guidance for Claude and any contributor working in this repository. Read this first.

## 1. Project Summary

EnvLedger is encrypted, versioned, local-first sync for `.env` files across a developer's machines through a shared folder (OneDrive, NAS, SMB). No server, no account, no Git dependency.

- **Status:** planned. No code exists yet. Docs must not claim a feature works until it is built and tested.
- **Origin:** the owner develops on a desktop and a laptop and hand-copies `.env` through OneDrive. EnvLedger replaces that.
- **Shape:** a TypeScript core library (npm) plus a thin VS Code extension that wraps it.
- **Name:** EnvLedger. The working name EnvVault was dropped because "Envault" already names several `.env` tools (npm `envault`, a Laravel server, a Python CLI, an encrypt/diff/merge CLI). Do not reintroduce `envvault` or `Envault` in names, paths, commands, or settings.

## 2. Package Metadata

Use these exact strings so every listing matches.

| Field | Value |
|---|---|
| npm package | `envledger` (confirm it is free before first publish; fallback is a scoped name such as `@<scope>/envledger`) |
| GitHub description | Encrypted, versioned sync for `.env` files across your machines via a shared folder (OneDrive, NAS, SMB). Local-first, no server or account. |
| npm description | Encrypted, versioned .env vault with per-key three-way merge. Sync .env files between machines through any shared folder. No server required. |
| Tagline | Your `.env` history, encrypted and in sync. Push on one machine, pull on the other. |
| License | Apache-2.0 |
| Author | Richard McQuiston, https://richardmcquiston.com/ |

```json
{
  "name": "envledger",
  "version": "0.0.0",
  "description": "Encrypted, versioned .env vault with per-key three-way merge. Sync .env files between machines through any shared folder. No server required.",
  "license": "Apache-2.0",
  "keywords": [
    "env",
    "dotenv",
    "environment-variables",
    "secrets",
    "encryption",
    "aes-256-gcm",
    "sync",
    "versioning",
    "merge",
    "vscode"
  ]
}
```

Wording rule: npm says "library"; the Marketplace listing says "VS Code extension".

## 3. Goals and Non-Goals

**Goals (v1)**

- Push the current `.env` to an encrypted vault in a user-chosen shared folder.
- Pull the latest version on another machine.
- Keep a version history and restore any version.
- Merge divergent edits from two machines per key, not per line.
- No server, no account, no Git dependency.

**Non-goals (v1)**

- Team permissions, auditing, or role-based access.
- A hosted or cloud sync service.
- Runtime secret injection (dotenvx, Doppler, and `op` already do this).
- Several env files in one pass (`.env.local`, `.env.production`). Planned for v1.1.

## 4. Tech Stack and Conventions

- TypeScript in strict mode. Follow the Google TypeScript Style Guide.
- Package manager: `bun` for local work. Publish to the npm registry.
- Build: `esbuild` bundle, `vsce` for packaging, `vitest` for tests.
- Runtime dependencies stay minimal: Node `crypto` (AES-256-GCM, scrypt), `node-diff3` (whole-file merge fallback), `crypto.randomUUID()`.
- No Tailwind and no webview UI in v1.
- Type every variable and return value. Use descriptive camelCase names.
- JSON for every on-disk format. Every record has a UUID `id`. Mutable records also carry `createdAt` and `updatedAt`.
- Functions return a `Result<T, VaultError>` that the caller must check. No silent catches and no thrown strings.
- Error messages say what failed, on which path or version, and the likely cause. Never "something went wrong".
- Catch errors early. Continue when safe; otherwise stop cleanly and leave user files untouched.
- Commits use Conventional Commits. Test names follow `<Method>_<Conditions>_<Outcome>`.

```ts
type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };

interface VaultError {
  code: VaultErrorCode;
  message: string; // e.g. "Cannot decrypt version 3f2a...: wrong passphrase or corrupted file"
  path?: string;
  cause?: unknown;
}
```

## 5. Repository Layout (proposed)

Two packages in one `bun` workspace. This is open question 8; confirm before scaffolding.

```
packages/
  core/                  # npm: envledger. Never imports `vscode`.
    src/
      vault/
        vaultStore.ts    # read/write version files in the shared folder
        versionGraph.ts  # parent links, heads, lowest common ancestor
        manifest.ts      # vault.json (written once)
      crypto/
        keyDerivation.ts # scrypt: passphrase -> key
        cipher.ts        # AES-256-GCM encrypt/decrypt
        keyStore.ts      # KeyStore interface (the extension implements it)
      env/
        envParser.ts     # raw text <-> ordered key/value entries
        envMerge.ts      # three-way per-key merge
      project/
        projectIdentity.ts
      util/
        result.ts
        logger.ts        # never logs secret values
    test/
  extension/             # VS Code extension, thin wrapper over core
    src/
      extension.ts       # activate(), command registration only
      commands/          # one file per command
      ui/
        prompts.ts       # input boxes, quick picks, conflict picker
        diffView.ts      # wraps vscode.diff with virtual documents
      secretStorageKeyStore.ts  # KeyStore backed by VS Code SecretStorage
README.md  ROADMAP.md  CLAUDE.md  SECURITY.md (planned)
```

`core` is unit testable in plain Node. `extension` only wires commands, prompts, and the keychain.

## 6. Vault Layout (sync-safe by design)

Core rule: **no file is ever modified by two machines.** Versions are immutable, one file each. This avoids OneDrive "conflicted copy" files. Do not use SQLite or any single shared database file on a synced or network drive.

```
<vaultRoot>/
  vault.json                          # written once: formatVersion, kdf params, salt, keyCheck blob
  <projectId>/
    <createdAtIso>_<versionId>.json   # one immutable file per version
```

Version file:

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

- Encrypted payload: raw `.env` text plus a SHA-256 content hash. Raw text keeps comments, order, and quoting.
- Header fields are authenticated as AES-GCM additional data, so tampering with parents or times is detected.
- Versions form a small graph, like Git commits. **Heads** are versions with no children.
  - One head: linear history. Pull fast-forwards.
  - Several heads: machines diverged. Pull merges and writes a merge version with two parents.
- Parent IDs, times, and machine labels are plaintext by design (cheap history listing). Document this in `SECURITY.md`.
- Any change to this format needs a `formatVersion` bump. Ask the owner first.

## 7. Cryptography and Key Management

- Key derivation: scrypt (Node built-in). Parameters live in `vault.json`. Argon2id is a possible later upgrade.
- Cipher: AES-256-GCM with a fresh random 12-byte nonce per version.
- `vault.json` holds a `keyCheck` blob (known plaintext, encrypted) so a wrong passphrase fails fast with a clear message.
- The derived key is cached per machine in VS Code `SecretStorage` (OS keychain). The passphrase is never stored.
- Recovery: "Export Recovery Key" shows the derived key once for offline storage. Without it, a lost passphrase means lost data. The README says so.
- Passphrase rotation is deferred to v2 (versions are immutable, so rotation means a new vault).
- Use constant-time comparison wherever hashes or tags are compared by hand.

## 8. Project Identity

Folder paths differ per machine, so identity is not path-based.

1. On first Push, generate a UUID `projectId` and write `.vscode/envledger.json` (`{ "projectId": "...", "vaultRoot": null }`). It holds no secrets and may be committed.
2. Optional fallback: match on a hash of the Git remote URL when that file is absent.
3. `vaultRoot` is a per-machine user setting, `envledger.vaultRoot`, because the OneDrive or NAS path differs per machine.

## 9. Commands

| Command ID | Title | Behavior |
|---|---|---|
| `envledger.init` | EnvLedger: Initialize Vault | Pick shared folder, set passphrase, write `vault.json`, cache key. |
| `envledger.unlock` | EnvLedger: Unlock Vault | Enter passphrase on a new machine; verify against `keyCheck`. |
| `envledger.push` | EnvLedger: Push .env | Parse; skip if hash equals the head; else write a new version with the head as parent. |
| `envledger.pull` | EnvLedger: Pull Latest | Back up local `.env` to `.env.bak.<timestamp>`; fast-forward or merge; write result. |
| `envledger.history` | EnvLedger: Show History | Quick pick of versions (time, machine, key-count change). |
| `envledger.restore` | EnvLedger: Restore Version | Pick a version; back up local file; write it; optionally push as a new version. |
| `envledger.diff` | EnvLedger: Diff With Vault | `vscode.diff` between local `.env` and a vault version. |
| `envledger.exportKey` | EnvLedger: Export Recovery Key | Show the derived key once. |

Diffs mask values by default (`KEY=••••`) with a reveal toggle.

## 10. Merge Algorithm

Input: base (lowest common ancestor), ours, theirs. Each is parsed to an ordered map of key to `{ value, rawLine, leadingComments }`.

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

- Conflicts show a quick pick per key: keep local, keep vault, or edit by hand.
- Output order follows "ours", with new keys appended, to keep diffs stable.
- Parser edge cases that need tests: quoted multi-line values, `export` prefix, inline comments, blank lines, duplicate keys, CRLF vs LF, BOM, values containing `=` and `#`.
- If either side fails to parse, fall back to a whole-file three-way merge with `node-diff3` and conflict markers.

## 11. Error Handling Rules

- Every filesystem and crypto call returns a `Result`. No bare `try/catch` that swallows.
- On failure: log details to the OutputChannel (never values), show a descriptive notification, leave the local `.env` untouched.
- Writes to `.env` are atomic: write a temp file in the same directory, then rename. A backup always exists before an overwrite.
- If the shared folder is unreachable (NAS offline, OneDrive signed out), report the exact path and operation. Push and Pull never apply partially.
- Zero-byte or truncated version files (mid-sync) fail JSON parse or auth-tag checks. Report "version file may still be syncing" and skip them; do not call it corruption.

## 12. Security Checklist

- [ ] Warn if `.env` is not covered by `.gitignore`.
- [ ] Never log or send secret values or key material anywhere.
- [ ] No network calls from the extension or library.
- [ ] No crypto dependencies beyond Node built-ins.
- [ ] `SECURITY.md` states what is encrypted, what is plaintext metadata, and the threat model (protects against a leaked or stolen cloud drive or NAS, not a compromised dev machine).
- [ ] Release artifacts built from a tagged commit, with a published checksum.

## 13. Testing

- Unit (`vitest`, no VS Code): parser, merge table, cipher round trip, wrong-key failure, tamper detection, version graph (linear, diverged, merge-of-merge).
- Integration: simulate two machines as two temp directories sharing one vault directory; interleave pushes and pulls.
- Sync fault injection: zero-byte file, truncated JSON, duplicate version IDs.
- Extension host tests (`@vscode/test-electron`) cover command wiring only.
- Test data uses fake values. Never use real secrets in fixtures.

## 14. Milestones

1. **M1: Core library (2 to 3 days).** Result type, cipher, key derivation, parser, vault store, linear versions, unit tests.
2. **M2: Extension MVP (2 days).** Init, Unlock, Push, Pull (fast-forward only), History, Restore, atomic writes, backups.
3. **M3: Merge (2 to 3 days).** Head detection, three-way merge, conflict picker, masked diff view.
4. **M4: Hardening (2 days).** `.gitignore` warning, recovery key export, sync fault handling, two-machine manual test.
5. **M5: Publish (1 to 2 days).** README, `SECURITY.md`, icon, Marketplace and Open VSX listings, sponsor link.

## 15. Open Questions

1. One vault for all projects (folder per project, shared passphrase), or one vault per project? Current plan: one vault.
2. Commit `.vscode/envledger.json`, or keep the project ID out of the repo with a per-machine mapping?
3. Keep parent IDs and times in plaintext, or encrypt the whole graph and lose cheap history listing?
4. Auto-push on save and auto-pull on open, or manual only? Current plan: manual in v1, opt-in auto later.
5. Multiple env files in v1 or v1.1?
6. License and monetization: Apache-2.0 plus an optional sponsor link, or a paid tier for extras?
7. Open VSX (VSCodium, Cursor, Windsurf) from day one?
8. Two packages (`core` and `extension`) or one?

## 16. Risks

| Risk | Mitigation |
|---|---|
| Lost passphrase means lost data | Recovery key export; clear README warning |
| Cloud sync delays produce partial files | Immutable files; validation; "still syncing" handling |
| A merge bug overwrites a good `.env` | Mandatory backup before every overwrite; tests on the merge table |
| Crowded market (dotenvx, SOPS, secret managers, other "envault" tools) | Local-first, no server or Git needed; treat as a personal tool first |
| Trust barrier for a secrets tool | Open source, tiny dependency surface, `SECURITY.md` |

## 17. Documentation Conventions

- `README.md` follows the owner's template: title, author and website lines, one-row Table of Contents, Overview, Getting Started (Prerequisites, Installation, Usage, Examples), Buy Me a Coffee, License, Copyright. Every H2 ends with a "Back to Top" link.
- Linked docs (`ROADMAP.md`, etc.) have a "Back to README" link at top and bottom, a menu row of section links, and "Back to Top" after each section.
- Write prose at about grade 8 reading level. Expand acronyms on first use in each file.
- Mark anything unbuilt as "planned". Never invent commands, versions, or defaults.

## 18. Working Rules for Claude

- Read this file and `ROADMAP.md` before changing code or docs.
- Never open a real `.env` or other secret file. Use `.env.example` and variable names only.
- Never print, log, or commit secret values or key material, including in tests and error messages.
- Ask before changing the vault file format, the crypto parameters, or the command IDs.
- Keep `core` free of `vscode` imports.
- Keep README, ROADMAP, and this file consistent when a decision changes. Update open questions when they are settled.
- When something is unclear, ask one short question instead of guessing.
