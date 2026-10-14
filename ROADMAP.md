# Roadmap

[Back to README](./README.md)

| [Milestones](#milestones) | [Open Questions](#open-questions) |

## Milestones

Status: all milestones are planned. Times are rough estimates.

1. **M1: Core library (2 to 3 days).** Result type, cipher, key derivation, .env parser, vault store, linear versions, and unit tests.
2. **M2: Extension MVP (2 days).** Initialize, Unlock, Push, Pull (no merge yet), History, Restore, safe file writes, and backups.
3. **M3: Merge (2 to 3 days).** Detect diverged versions, run the key-by-key merge, pick a winner on conflicts, and show diffs with hidden values.
4. **M4: Hardening (2 days).** The `.gitignore` warning, recovery key export, handling of half-synced files, and a manual test on a desktop and a laptop.
5. **M5: Publish (1 to 2 days).** README, security notes, icon, Marketplace and Open VSX listings, and a sponsor link.

[Back to Top](#roadmap)

## Open Questions

1. Should one vault hold many projects, or should each project get its own vault? The current plan is one vault with a folder per project.
2. Should the project ID live in a file you commit to Git, or stay out of the repo?
3. Is it fine to leave parent IDs and times unencrypted so history loads fast?
4. Should push and pull run on their own, or only when you ask? The current plan is manual first.
5. Should v1 handle more than one env file, such as `.env.local`?
6. Which license and pricing should the project use?
7. Should Open VSX (for VSCodium and similar editors) be supported from day one?
8. Should the library and the extension be two packages in one repo, or one package?

[Back to Top](#roadmap)

---

[Back to README](./README.md)
