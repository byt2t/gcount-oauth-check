# G Count — Apple-only recovery gate (2026-10-13)

**Status:** design / feasibility review only. **No production implementation approved.**

## Verified facts

- Owned Supabase project: `gcount-prod` (`aewatvipmggdymyjhwkx`); users copied from Lovable source.
- 14 users, 15 provider identities in snapshot: 10 Google, 5 Apple, one overlapping account, so 4 apparently Apple-only (verify the live inventory before changes).
- Google authentication from temporary GitHub Pages returned the exact pre-existing Auth UUID. Apple returned OAuth callback 422 `signup_disabled` while new signups were disabled; no session was established.
- Apple Developer callback and service ID configuration visually verified; no evidence yet of the precise new Apple `sub`.
- The legacy login page calls `lovable.auth.signInWithOAuth` via `@lovable.dev/cloud-auth-js` (G Count repository `src/routes/login.tsx`, `src/integrations/lovable/index.ts`).
- G Count `src/lib/auth.tsx` can invoke `ensure_my_profile` when a session has no profile: **never route a migration test through this path**.
- The temporary test site's `linkIdentity` action was disabled in commit `ef46ad7d9cad46e39c1e54be85a1e640a411e6ba`. Keep disabled.

## Security invariants

- Preserve legacy Auth UUID and all profile, event, pint, badge, admin and notification ownership references.
- No client-generated proof of old account ownership (email, username, profile screenshot or UUID) may authorize access. Check the old session **server-side** with the original auth service.
- Do not expose target service-role, Apple private key, client secret or recovery links in a public page, logs, emails or GitHub.
- New Apple OAuth proof must come from the *actual* Apple identity and must be verified securely. Distinct Apple private relay addresses are NOT identity evidence.
- Use single-use, short-lived, account-bound challenge/state, reject replays and mismatched identities; audit access without writing sensitive tokens. No extra roles or statuses.
- Never edit `auth.users` or `auth.identities` via SQL merely to force identity linking. Confirm supported Auth APIs first.
- Keep the legacy Lovable service active until all Apple-only members are recovered and their data integrity verified.
- Do not enable open signups on `gcount-prod` for troubleshooting.

## Proposed path, in order (NOT implemented)

### Gate A — Feasibility, no writes

1. Confirm the real Apple-only inventory in the latest data and ownership/contact status; do not infer availability from the snapshot.
2. Validate whether Supabase Auth permits another Apple identity on a user who *already has an imported Apple identity*, and whether manual linking can run when signups are disabled. The docs do not establish this compatibility.
3. Validate an existing-user session bootstrap from a legacy-authenticated account without relying on email/password sign-in, since Email provider is disabled on target and an Apple private relay may not accept project email.
4. Check if Apple identity subject continuity is possible from the original OAuth configuration. Do not treat the Apple account's personal Lovable sign-in as proof of the application's legacy provider configuration.
5. Confirm availability of backups, rollback and post-link read-only checks of `auth.users`, `auth.identities`, profiles and linked data.

### Gate B — Prototype ONE owned test account, only with explicit approval

1. Verify a fresh **legacy** G Count session through the original auth service, server-side; resolve its subject to the *existing* UUID.
2. Obtain a secure, account-bound, single-use migration challenge with short expiry.
3. Establish an authenticated target Supabase session for **that same** migrated UUID, using an officially supported, verified path; **block implementation if this is not viable**. Admin-generated magic link/recovery link is only a candidate: check whether it works with disabled Email auth, private relays and without disclosing token-bearing URLs.
4. Capture new Apple proof and use a supported Auth linking flow if feasible. Manual link must be enabled and must permit the user's pre-existing Apple identity. Do NOT delete or overwrite the imported identity without an approved, tested rollback procedure.
5. Re-read Auth UUID, linked providers, and profile/data access. A different UUID, unexpected user, changed ownership, duplicate identity or missing data is an immediate stop.

### Gate C — Member recovery

- Repeat the proven flow for each Apple-only user through an explicit in-app transition; never blanket-link based on addresses.
- Preserve a staffed escalation channel for those who cannot access their original Apple account. No mass forced resets.
- Go/no-go requires 100% recovery of affected Apple accounts and no duplicate Auth user created.

## Rejected shortcuts

- Turning signups on and asking Apple users to re-register.
- Changing the migrated UUID, manually rewriting provider IDs or mass-moving ownership columns.
- Blindly linking Apple on a dual Google/Apple user when old Apple identity is already present.
- Assuming that a personal “Lovable” authorization in Apple account settings identifies the legacy G Count OAuth provider.

## Official references

- Supabase Identity Linking: https://supabase.com/docs/guides/auth/auth-identity-linking
- Supabase Admin generateLink: https://supabase.com/docs/reference/javascript/auth-admin-generatelink
- Supabase Apple Auth: https://supabase.com/docs/guides/auth/social-login/auth-apple
- Apple app-team transfer technical note (may not apply): https://developer.apple.com/documentation/technotes/tn3159-migrating-sign-in-with-apple-users-for-an-app-transfer

## Next concrete action

Get an answer to Gate A points 2–3 in a throwaway/nonproduction setup **before** drafting or deploying any recovery Edge Function. The exact flow must be validated before members are touched. 
