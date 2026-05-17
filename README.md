# worktrace-auth

**Encrypted credentials store for [WorkTrace](https://github.com/kjain-Cloudforia/worktrace-app)** — the public dashboard's authentication backend.

> ⚠ **This repo is PUBLIC by design.** Every file in here is either AES-GCM-encrypted ciphertext, schema, or operational docs. Decrypting any user record requires that user's password; decrypting an escrow or recovery record requires the admin recovery code. No plaintext credentials are ever committed.

## Why a public repo?

Because GitHub Pages on a private repo requires a paid plan, and we deliberately don't want a backend service. So instead the dashboard makes the auth flow work *with just public-readable files*:

1. User opens the dashboard, types their username + password.
2. Browser fetches `users/<username>.json` (this repo, public) — no auth header, no PAT needed.
3. Browser runs `PBKDF2(password, salt, 600_000 iters)` → AES key.
4. Browser AES-GCM-decrypts the embedded ciphertext → the user's GitHub PAT.
5. Browser uses that PAT to fetch the user's private data repo.

The password never leaves the browser. The PAT lives in `sessionStorage` only for the duration of the tab. If the password is wrong, AES-GCM's MAC fails and the decrypt throws — no false positives.

## Repository layout

```
worktrace-auth/
├── README.md                  ← this file
├── CONTRIBUTING.md            ← admin operations reference
├── users/
│   ├── kashish.json           ← user record, encrypted under kashish's password
│   ├── admin.json             ← admin record, encrypted under admin's password
│   └── <other-users>.json
├── escrow/
│   ├── kashish.json           ← kashish's PAT, encrypted under the admin recovery code
│   └── <other-users>.json     ← (admins don't have escrow files — see Threat model)
└── admin.recovery.json        ← admin's PAT, encrypted under the recovery code
```

| Artifact | Purpose | Encryption key | When written |
|---|---|---|---|
| `users/<u>.json` | The thing the user signs in with | Their password | Created at provisioning; rewritten on every password change |
| `escrow/<u>.json` | Lets admin reset the user's password without contacting them | Admin recovery code | Created at provisioning; never rewritten unless the recovery code rotates |
| `admin.recovery.json` | Lets admin recover their own password from a fresh laptop | Admin recovery code | Created once at admin setup; rewritten only on code rotation |

## Crypto in one screen

Everything in this repo uses the same primitives:

```
KDF      = PBKDF2-HMAC-SHA256, 600_000 iterations  (OWASP 2023 baseline)
SALT     = 16 bytes random per record
CIPHER   = AES-GCM-256, 12-byte random IV per record
AAD      = none
LIBRARY  = Web Crypto API (crypto.subtle) in the browser
           cryptography (Python) in the helper scripts — round-trip verified
```

Why these choices:
- **PBKDF2 over Argon2:** Web Crypto has no Argon2; we don't want a wasm dependency on the login path.
- **600k iterations:** ~2 seconds of PBKDF2 on a 2023-era laptop. Login feels snappy, offline brute force is computationally prohibitive at our team size.
- **AES-GCM:** authenticated, fast, hardware-accelerated. The MAC tag is what tells us "wrong password" (vs. corrupted file).
- **No AAD:** we considered binding the username into AAD as a tamper-prevention measure but the public fields outside the ciphertext don't need cryptographic binding for our threat model (no one benefits from swapping ciphertexts between users — they'd still need the matching password).

## The recovery code

A 24-character Crockford-base32 string formatted as `XXXX-XXXX-XXXX-XXXX-XXXX-XXXX`. Roughly 120 bits of entropy. Generated once at admin setup, shown to admin once, stashed offline (1Password, paper). **There is no copy anywhere except the admin's offline storage.**

The recovery code is the **root of trust** for the entire system. It unlocks two things:

1. `admin.recovery.json` → admin's PAT plaintext → admin can set a new admin password.
2. `escrow/<u>.json` → that user's PAT plaintext → admin can re-encrypt under a new password and let the user back in.

If the code is lost AND admin loses their password, the system is unrecoverable from the dashboard — admin must rebuild from a laptop that still has `config.json` (which has the admin PAT in plaintext) by running `scripts/reset_admin.py` + `scripts/build_recovery_artifacts.py` to mint a fresh code.

### Why escrow is keyed to the recovery code, not the admin password

Earlier design (rejected): escrow was encrypted under the admin's password. Every admin password change invalidated every escrow file — silently — and there was no path to rebuild them after an admin password *recovery* (since recovery means the old password is lost).

Current design: escrow is encrypted under the recovery code, which doesn't rotate when passwords do. Admin password changes don't touch escrow. Admin recovery doesn't touch escrow. The only operation that requires rebuilding escrow is rotating the recovery code itself, which is rare-and-deliberate.

Trade-off: admin must paste the recovery code into the dashboard each time they reset a user's password. For a small team where resets happen a few times a year, this is fine — and it forces periodic verification that the code is still findable in 1Password.

## Threat model

| Threat | Defense | Residual risk |
|---|---|---|
| Public repo is world-readable | All credentials are AES-GCM ciphertext. PBKDF2 600k iters makes offline brute-force computationally infeasible against any password ≥12 chars mixed-case + digit (enforced at sign-up). | Weak/leaked passwords. Mitigated by policy enforcement. |
| Attacker captures a user's password | Single user compromised. Their data repo is accessible, but admin can revoke + reissue. | Lateral movement to other users — none, each user's PAT is scoped to their own data repo. |
| Attacker captures admin's password | Admin's PAT exposed. Can read/write `worktrace-auth` + read all data repos. **Cannot decrypt escrow/recovery files** — those use the recovery code, not the password. | Admin can recover via recovery code, then rotate the admin PAT to invalidate the captured one. |
| Attacker captures the recovery code | Can decrypt `admin.recovery.json` → admin PAT plaintext. Effectively full admin compromise. | Mitigated by keeping the code offline only. |
| Stolen laptop with `~/Documents/DevPlatform/config.json` | PATs in plaintext on that machine. Same severity as "attacker captures admin password" but worse — they have the PAT *and* the password. | Revoke PATs on GitHub Settings → Developer settings → Tokens. Generate new ones via `scripts/`. |
| Compromised CDN (raw.githubusercontent.com serves modified file) | The Contents API (api.github.com) is what we actually use — keyed by commit, not by CDN edge. Decrypted ciphertext that's been tampered with fails AES-GCM's MAC check and we treat it as wrong-password. | None — both paths require GitHub's HTTPS to be broken. |
| Loss of recovery code + admin password | Cannot recover from a fresh laptop. Must rebuild from a machine with `config.json` still intact (admin PAT plaintext). | If laptop is also gone: generate new admin PAT on GitHub, re-run `scripts/build_initial_user_records.py` + `scripts/build_recovery_artifacts.py`. Existing users' auth files become orphaned until each user is re-provisioned. |

The trust hierarchy:

```
Recovery code (offline)
   │
   ├─▶ admin.recovery.json  →  admin PAT plaintext  →  admin can rewrite anything in worktrace-auth
   │
   └─▶ escrow/<u>.json      →  user PAT plaintext   →  admin can rewrite users/<u>.json
```

A user password compromise stays local to that user. An admin password compromise is limited (no escrow access). A recovery code compromise is system-wide.

## How a sign-in works

```
1. User opens https://kjain-Cloudforia.github.io/worktrace-app/
2. Types username + password
3. Browser fetches api.github.com/repos/.../contents/users/<u>.json
   (Accept: application/vnd.github.v3.raw — authoritative, no CDN cache)
4. Browser runs PBKDF2(password, file.salt, 600_000) → AES-256 key
5. Browser AES-GCM-decrypts file.ciphertext → user's GitHub PAT
6. Browser probes the PAT against the user's data_repo to confirm it works
7. PAT stored in sessionStorage; cleared on tab close
8. Dashboard renders modules
```

Step 3 is critical: we use the **Contents API**, not `raw.githubusercontent.com`. The raw CDN keys its cache by path and ignores query-string cache-busters, so just-rewritten files can serve stale for several minutes. The Contents API has no such layer — it's authoritative for the current commit.

## For admins

Day-to-day admin operations (provision users, reset passwords, revoke users) all live in the dashboard's **Admin Console** tile, which is visible only to records with `is_admin: true`. See [`CONTRIBUTING.md`](./CONTRIBUTING.md) for the operational reference.

For emergencies (admin can't sign in, escrow files corrupted, etc.) see `~/Documents/DevPlatform/scripts/README.md` for the local Python fallbacks.

## What NOT to commit

`.gitignore` blocks the obvious patterns. Concretely:
- Never commit a plaintext PAT or password
- Never commit `config.json` (which lives outside this repo anyway, at `~/Documents/DevPlatform/config.json`)
- Don't commit decryption test scripts that hardcode credentials
- Don't paste a recovery code in a commit message or issue

If a secret accidentally lands in a commit, treat it as compromised: rotate it (rebuild via scripts), revoke the original on GitHub, force-push to remove the commit from history.

## Related repos

- [`worktrace-app`](https://github.com/kjain-Cloudforia/worktrace-app) — public dashboard (HTML/JS/CSS), `auth.js` library, schemas
- `worktrace-data-<username>` — one per user, **private**, holds module data
- `worktrace-auth` (this repo) — public credentials store
