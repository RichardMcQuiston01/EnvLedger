# EnvLedger

```
Your .env history, encrypted and in sync. Push on one machine, pull on the other.
```

- Author: [Richard McQuiston](mailto:richard.mcquiston@gmail.com)
- Website: https://richardmcquiston.com/

## Table of Contents

| [Overview](#overview) | [Features](#features) | [Getting Started](#getting-started) | [Security](#security) | [More Information](#more-information) | [Buy Me a Coffee](#buy-me-a-coffee) | [License](#license) | [Copyright](#copyright) |

## Overview

**Project status: planned.** No code exists yet. The steps below describe how EnvLedger is designed to work. See the [Roadmap](./ROADMAP.md) for what gets built and when.

EnvLedger gives you encrypted, versioned sync for .env files across your machines. It uses a shared folder, such as OneDrive, a Network Attached Storage (NAS) drive, or a Server Message Block (SMB) share. It is local-first, with no server or account.

It is for developers who work on more than one computer. A .env file holds settings and secrets that stay out of Git. Copying it by hand between machines is slow and easy to get wrong. EnvLedger saves each version of the file in an encrypted vault. Any machine that can reach the folder and knows your passphrase can pull the same file.

EnvLedger is planned as a Visual Studio Code (VS Code) extension built on a TypeScript library.

[Back to Top](#table-of-contents)

## Features

- Push your current .env file to the vault, and pull the latest version on another machine.
- Keep a history of versions, and restore any one of them.
- Merge changes from two machines key by key, not line by line.
- Hide values in diffs, so secrets stay private on screen shares.
- Back up your local .env file before every overwrite.
- Work without a server, an account, or Git.

[Back to Top](#table-of-contents)

## Getting Started

These steps are planned. Each one works once its milestone in the [Roadmap](./ROADMAP.md) is done.

### Prerequisites

- Visual Studio Code (VS Code).
- A folder that all your machines can reach, such as a OneDrive folder or a NAS share.
- A passphrase that you will use on every machine.
- Node.js, only if you build from source.

### Installation

Install the extension (planned for the publish milestone):

1. Open the Extensions view in VS Code.
2. Search for EnvLedger.
3. Select Install.

Use the library in your own code (planned):

```
npm install envledger
```

### Usage

On your first machine:

1. Open your project in VS Code.
2. Open the Command Palette and run **EnvLedger: Initialize Vault**.
3. Choose the shared folder and enter a passphrase.
4. Run **EnvLedger: Push .env**.

On your second machine:

1. Open the same project in VS Code.
2. Set the `envledger.vaultRoot` setting to the same shared folder.
3. Run **EnvLedger: Unlock Vault** and enter your passphrase.
4. Run **EnvLedger: Pull Latest**.

Other commands:

| Command | What it does |
|---|---|
| **EnvLedger: Show History** | Lists saved versions with time and machine name. |
| **EnvLedger: Restore Version** | Writes an older version to your .env file. |
| **EnvLedger: Diff With Vault** | Compares your .env file with a saved version. |
| **EnvLedger: Export Recovery Key** | Shows a key you can store offline. |

### Examples

**Merge changes from two machines.** You edit different keys on your desktop and laptop. EnvLedger merges them on the next pull.

| Key | Desktop | Laptop | Result |
|---|---|---|---|
| `API_URL` | changed | unchanged | desktop value |
| `DEBUG` | unchanged | changed | laptop value |
| `NEW_KEY` | unchanged | added | kept |
| `DB_PASSWORD` | changed | changed differently | you choose |

**Restore an old version.** Run **EnvLedger: Show History** and pick a version. Then run **EnvLedger: Restore Version**. EnvLedger first saves your current file as `.env.bak.<timestamp>`.

[Back to Top](#table-of-contents)

## Security

- File contents are encrypted with the Advanced Encryption Standard (AES) using a 256-bit key in Galois/Counter Mode (GCM). The key comes from your passphrase.
- Version IDs, parent links, times, and machine names are not encrypted. This lets EnvLedger list your history without opening each file.
- Your passphrase is never stored. The derived key is cached in your operating system keychain.
- If you lose your passphrase, the data is gone. Export a recovery key and keep it offline.
- EnvLedger protects you if a cloud drive or NAS is leaked or stolen. It does not protect you from malware on your own machine.
- EnvLedger warns you if your .env file is not listed in `.gitignore`.

[Back to Top](#table-of-contents)

## More Information

- [Roadmap](./ROADMAP.md): milestones and open questions.
- [CLAUDE.md](./CLAUDE.md): the full design, conventions, and working rules for this project.

[Back to Top](#table-of-contents)

## Buy Me a Coffee

If this app, code, or repository has helped you or someone you know, please consider donating. I appreciate any help to offset the costs of development and/or AI Credits.

[**Donate via Stripe**](https://donate.stripe.com/00w5kD3Gj1Xo9v7gVOcs800), or scan:

[![Donate via Stripe](./donate.svg)](https://donate.stripe.com/00w5kD3Gj1Xo9v7gVOcs800)

[Back to Top](#table-of-contents)

## License

Apache 2

[Back to Top](#table-of-contents)

## Copyright

(c)2026 Richard McQuiston. All rights reserved.

[Back to Top](#table-of-contents)
