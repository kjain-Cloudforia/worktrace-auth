# worktrace-auth — admin operations

Day-to-day operations live in the **Admin Console** tile on the WorkTrace dashboard. Almost everything you used to do via the DevTools console or hand-edited JSON is now a button click. This doc is the operational reference.

For the *architectural* "why", see [`README.md`](./README.md).

## The repo's invariants

- **One file per user:** `users/<username>.json` (and `escrow/<username>.json` for non-admins).
- **`username` is a lowercase slug:** letters, digits, hyphens; starts with a letter.
- **Schema:** [`worktrace-app/docs/schema/auth-user/v1.json`](https://github.com/kjain-Cloudforia/worktrace-app/blob/main/docs/schema/auth-user/v1.json).
- **All writes go through the dashboard or the helper Python scripts.** Hand-editing files works in a pinch but skips the crypto round-trip verification — easy way to lock yourself out.
- **No plaintext credentials in commits.** Ever.

## Day-to-day operations

### Add a new teammate

**Full playbook lives at [`worktrace-cli/NEW-TEAMMATE.md`](https://github.com/kjain-Cloudforia/worktrace-cli/blob/main/NEW-TEAMMATE.md)** — that's the single canonical doc. It covers your part (GitHub + dashboard, ~5 min) and the teammate's part (paste one curl line, ~3 min) in one place.

Short version for context:

1. **GitHub (you):** create `worktrace-data-<username>` (private) + add them as collaborator + generate a fine-grained PAT with Contents:RW on that repo + worktrace-auth.
2. **Dashboard (you):** Admin Console → **+ Add team member** → fill the form (username, display name, data repo, PAT, initial password, your recovery code, their work shift). Submit. The dashboard probes everything, encrypts the PAT under both the initial password and the recovery code, and commits `users/<u>.json` + `escrow/<u>.json`.
3. **Out-of-band (you):** send the teammate three values — username, initial password, PAT — plus the install command from NEW-TEAMMATE.md.
4. **Their laptop (them):** they paste the install one-liner. Done in ~3 minutes.

If anything in this doc disagrees with NEW-TEAMMATE.md, treat NEW-TEAMMATE.md as authoritative.

### Reset a forgotten user password

User says "I forgot my password." You don't need to know it — escrow has their PAT.

**In the dashboard:**
1. Admin Console → find their row → **Reset password**.
2. Modal asks for your recovery code + a new temporary password.
3. Submit. The dashboard decrypts the escrow file with your recovery code → user's PAT → re-encrypts under the new temp password → commits the new `users/<u>.json`.
4. Send the temp password out-of-band. Tell them to rotate it on first login.

### Revoke a user

User leaves the team / something compromised / etc.

**In the dashboard:**
1. Admin Console → find their row → **Revoke** (subdued grey button; turns red on hover).
2. Type their username to confirm.
3. Submit. The dashboard deletes `users/<u>.json` + `escrow/<u>.json` from this repo. They can no longer sign in.

**Optional manual cleanup on GitHub:**
- Revoke their PAT in Developer Settings → Personal access tokens.
- Delete or archive their data repo if you want it gone.

The revoke flow leaves data alone by design — history preservation is a policy decision per-departure.

### Change your own password (admin or user)

1. Sign in.
2. Click **Change password** in the header.
3. Enter current password + new password (twice).
4. Submit. The dashboard re-encrypts your PAT under the new password and commits.

Admin password changes do *not* affect escrow files (they're keyed to the recovery code, not the password).

## Recovery operations

### Admin forgot their own password — "Forgot password?" flow

1. On the dashboard's login screen, click **Admin: forgot password?**
2. Paste the recovery code + set a new admin password.
3. Submit. The dashboard decrypts `admin.recovery.json` with the code → admin PAT plaintext → re-encrypts `users/admin.json` under the new password → auto-signs you in.

This recovers from a fresh laptop as long as you have the recovery code in hand. No `config.json` needed.

### Admin forgot password AND lost the recovery code

The dashboard flow can't help. Local fallback:

```bash
cd ~/Documents/DevPlatform
python3 scripts/reset_admin.py
```

This reads the admin PAT from `config.json` (must still be intact on the laptop), prompts for a new password, re-encrypts `admin.json`, and prints the git commit/push commands. After that, mint a fresh recovery code:

```bash
python3 scripts/build_recovery_artifacts.py
```

Then push both repos.

### Rotate the recovery code

If the code is compromised:

```bash
cd ~/Documents/DevPlatform
python3 scripts/build_recovery_artifacts.py
```

The script prompts for the existing code (needed to decrypt existing escrow files), generates a fresh code, re-encrypts `admin.recovery.json` + every `escrow/<u>.json` under the new code, prints the new code once. Commit + push.

## Admin role specifics

The `admin` user record has:
- `is_admin: true`
- `data_repo: null` (admin has no personal data repo — they manage others')
- `managed_repos: [...]` — list of data repos admin can read (informational; the source of truth for the roster is just "every file in `users/`")
- The PAT inside admin's record has Read access to every `worktrace-data-*` repo + Read+Write on `worktrace-auth`

Multiple admins are supported by the data model: just provision them with `is_admin: true`. Today the dashboard's "Reset password" button is hidden for admin rows (so admin can't reset their own from inside an authenticated session). Multi-admin support would relax this to allow admin-bob to reset admin-alice; not implemented yet because there's only one admin.

## Schema evolution

If we bump `schema_version` from 1 to 2:

1. Old records keep working (decryption only depends on `kdf`/`iterations`/`salt`/`iv`/`ciphertext`).
2. New records use v2.
3. The dashboard reads either; helper scripts emit only the current version.
4. Migration: when a v1 record is opened, optionally upgrade-in-place and rewrite — same pattern we'd use for iteration count bumps.

If AES-GCM is ever deprecated (unlikely), bump to v2 with a new `kdf` value + run a one-time migration script that decrypts under the old scheme and re-encrypts under the new.

## What to do if you accidentally commit a secret

A plaintext PAT or password lands in a commit. Treat as compromised:

1. Rotate the secret immediately:
   - For a PAT: revoke on GitHub Developer Settings → Tokens, generate a fresh one, update `config.json`, rebuild auth records via scripts.
   - For a password: change it via the Change Password modal.
   - For a recovery code: run `build_recovery_artifacts.py` to mint a new one.
2. Force-push to remove the commit from history (`git rebase -i` + drop the commit, then `git push --force-with-lease`).
3. Audit subsequent commits to confirm the secret doesn't reappear.

GitHub's secret scanning may also flag this and email you. Don't ignore those emails.
