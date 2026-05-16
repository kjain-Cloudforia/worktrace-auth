# Contributing to worktrace-auth

Admin operations for managing WorkTrace users. Most of these will move into the dashboard's admin module in Phase 5e — until then, here's the manual flow.

## The repo's invariants

- One file per user: `users/<username>.json`
- `username` is a lowercase slug: letters, digits, hyphens; starts with a letter
- File contents are AES-GCM ciphertext + PBKDF2 metadata + plaintext public fields (display name, repo pointer, admin flag)
- Schema: [`worktrace-app/docs/schema/auth-user/v1.json`](https://github.com/kjain-Cloudforia/worktrace-app/blob/main/docs/schema/auth-user/v1.json)
- The PAT inside the ciphertext must have the correct GitHub permissions for that user's `data_repo`. Issuing the PAT and verifying scopes is part of every create/reset flow.

## Onboarding a new user (manual, until Phase 5e)

For each new user "alice":

### 1. Create their data repo

In the admin's GitHub UI:
- New repo `worktrace-data-alice`
- **Private**
- Initialize with a tiny README

Then invite the user as a collaborator with **Write** access.

### 2. Generate a fine-grained PAT scoped to their data repo

Admin generates this on the user's behalf (the user never sees a PAT):
- GitHub → Settings → Developer settings → Fine-grained tokens → Generate new
- Token name: `worktrace-<username>` (e.g. `worktrace-alice`)
- Resource owner: admin's account (the org/user that owns the data repo)
- Repository access: **Only select repositories** → `worktrace-data-alice` + `worktrace-auth`
  - `worktrace-data-alice` — Contents: Read + Write (for `dpsync` from the user's laptop)
  - `worktrace-auth` — Contents: Read (so the user can read their own auth file on login; and Read+Write later if they want to change their own password)
- Expiration: 365 days (the maximum)

### 3. Build the encrypted user record

Until the admin module ships in Phase 5e, this is done from the dashboard's DevTools console:

```js
import('./auth/auth.js').then(async (m) => {
  const record = await m.buildUserRecord({
    username:     'alice',
    displayName:  'Alice Example',
    dataRepo:     'kjain-Cloudforia/worktrace-data-alice',
    pat:          'github_pat_…',       // ← the PAT from step 2
    password:     'Some-Strong-Pass-1!', // ← initial password; user changes on first login
    isAdmin:      false,
  });
  console.log(JSON.stringify(record, null, 2));
});
```

Copy the output JSON.

### 4. Commit the file

Create `users/alice.json` in this repo with that JSON. Commit. Push.

```bash
echo "<paste>" > users/alice.json
git add users/alice.json
git commit -m "Add user: alice"
git push
```

### 5. Hand the user their credentials (out-of-band)

Send via Signal, in person, or another secure channel:
- Their username (`alice`)
- The initial password (`Some-Strong-Pass-1!`)
- Dashboard URL

The user logs in, ideally changes their password immediately (Phase 5f flow).

## Password reset (user forgot password)

The encrypted PAT can't be recovered without the password. Admin's options:

1. **Reissue PAT, set new password** (preferred — clean slate)
   - Revoke the existing PAT in GitHub
   - Generate a fresh PAT with the same scopes
   - Re-run the build-record flow with a new initial password
   - Replace `users/<username>.json` with the new record
   - Hand the user their new password out-of-band

2. **Re-encrypt the existing PAT under a new password** (if PAT still valid and recoverable — usually it's not)
   - Use `m.rekeyUserRecord(record, oldPw, newPw)` if you somehow still have the old password
   - Not typical — included for completeness

## Revoking a user

1. **Revoke the PAT in GitHub** (immediate — they can no longer access their data repo)
2. **Remove `users/<username>.json` from this repo** (so they can't log into the dashboard)
3. **Optional: archive their data repo** (GitHub → Settings → Archive — preserves history without active access)
4. **Optional: remove them as collaborator** on the data repo

## Admin role specifics

The `admin` user record (`users/admin.json`) has:
- `is_admin: true`
- `data_repo: null` (admin has no personal data of their own — they manage others')
- `managed_repos: ["worktrace-data-kashish", "worktrace-data-alice", ...]` — list of data repos the admin can read
- The encrypted PAT inside admin's record has **Read** access to all `worktrace-data-*` repos plus **Read + Write** on `worktrace-auth` (so admin can manage user records from the dashboard in Phase 5e)

Multiple admins are supported: just add `admin-bob.json`, `admin-carla.json`, etc. with `is_admin: true`. The dashboard's admin module routes everyone with `is_admin: true` into the admin view.

## Key rotation

If the PBKDF2 iteration count moves up in a future version of `auth.js`:

1. Each user logs in once with their current password (correctly decrypts under old iterations)
2. Dashboard detects `record.iterations < KDF_ITERATIONS` and silently re-encrypts under the new iteration count
3. Commits the updated record back via the user's `worktrace-auth` Write scope (Phase 5f scope)

If AES-GCM is ever deprecated (unlikely), we'd bump the schema to v2 and run a one-time migration script.

## Don't commit secrets

`.gitignore` rejects `*.pat`, `.env*`, and other obvious patterns. The only sensitive data that should live in this repo is the AES-GCM ciphertext inside each `users/*.json`. The plaintext PATs and the user passwords must never appear in commits, comments, issue trackers, or chat.
