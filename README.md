# worktrace-auth

**Encrypted credentials store for WorkTrace** — sibling repo to [`worktrace-app`](https://github.com/kjain-Cloudforia/worktrace-app) and the per-user [`worktrace-data-*`](https://github.com/kjain-Cloudforia?tab=repositories&q=worktrace-data-) repos.

This repo is **PUBLIC by design**. The contents of `users/*.json` are AES-GCM-encrypted blobs that cannot be decrypted without each user's password. Making the repo public means the WorkTrace dashboard can fetch credentials at login time without any preexisting auth — which is what makes the username + password sign-in flow possible.

> 🔒 **The threat model** is documented in [`worktrace-app/docs/auth/auth.js`](https://github.com/kjain-Cloudforia/worktrace-app/blob/main/docs/auth/auth.js). TL;DR: PBKDF2-HMAC-SHA256 at 600k iterations + AES-GCM 256-bit + 16-byte unique salt per user. Strong passwords (≥12 chars mixed case + digit, enforced at sign-up) make offline brute-force computationally infeasible at our scale.

## Repository layout

```
worktrace-auth/
├── README.md                          ← this file
├── CONTRIBUTING.md                    ← admin-only onboarding & ops guide
└── users/
    ├── kashish.json                   ← per-user encrypted credentials
    ├── admin.json
    └── <new-user>.json                ← one file per WorkTrace user
```

Each `users/<username>.json` follows the schema at
[`worktrace-app/docs/schema/auth-user/v1.json`](https://github.com/kjain-Cloudforia/worktrace-app/blob/main/docs/schema/auth-user/v1.json). The dashboard fetches the matching file by username, derives an AES key from the typed password using PBKDF2, decrypts the embedded ciphertext, and uses the recovered GitHub PAT to fetch the user's actual data.

## How a sign-in works (high level)

```
1. User opens https://kjain-Cloudforia.github.io/worktrace-app/
2. Types username + password
3. Browser fetches https://raw.githubusercontent.com/kjain-Cloudforia/worktrace-auth/main/users/<username>.json
4. Browser runs PBKDF2(password, file.salt, 600_000 iters) → AES key
5. Browser AES-GCM-decrypts file.ciphertext → user's GitHub PAT
   (If password wrong → AES-GCM MAC fails → throw → "wrong username or password")
6. Browser fetches user's `data_repo` (e.g. worktrace-data-<username>) using the PAT
7. Dashboard renders normally
```

The password never leaves the browser. The PAT is never visible in cleartext anywhere on disk except the user's own browser memory while a session is active.

## For admins

See [`CONTRIBUTING.md`](./CONTRIBUTING.md) for: creating users, password resets, revoking users, key rotation.

## Privacy & security trade-offs

| Property | Status | Notes |
|---|---|---|
| Public repo readability | Yes, intentional | Ciphertext only — no plaintext PATs |
| Offline brute-force resistance | Strong against random ≥12-char passwords | Mitigated by 600k PBKDF2 iterations + per-user salt |
| Password recovery | Not possible without admin help | Forgotten password → admin issues a new PAT + initial password (the user's data is fine; just the credential needs regenerating) |
| 2FA / passkey support | Not supported by this scheme | If you need 2FA, the upgrade path is GitHub OAuth via Cloudflare Worker (Phase 5+) — see project plan |
| Sniffability over network | None | Login traffic is HTTPS to api.github.com; password stays browser-local |

## Related repos

- [`worktrace-app`](https://github.com/kjain-Cloudforia/worktrace-app) — public dashboard (HTML/JS/CSS), schemas, auth library
- `worktrace-data-<username>` — one per user, **private**, holds the user's actual timesheet/module data
- `worktrace-auth` (this repo) — public credentials store
