# Resets and invitations

## Password reset

Use the existing auth framework's reset mechanism when it meets the application's requirements. If implementing tokens, use a cryptographically secure random value with sufficient entropy (for example, 32 random bytes), store a one-way digest for verification, bind it to the account and reset purpose, and give it a short configurable expiry. Never log the token.

Return a consistent response whether an account exists or is suppressed; avoid obvious timing differences. Apply per-identifier and per-source request limits without turning the endpoint into an account-lockout mechanism. Build links using the configured public HTTPS origin. Do not use a caller-controlled Host or return URL. Ensure landing pages do not leak tokens through analytics, logs, referrers, or third-party assets.

If a durable worker needs the raw token, a verification digest alone cannot reconstruct it. Choose an explicit approach: encrypt the minimal outbox payload with tightly scoped worker access and short retention, or create the token at dispatch using an existing secure auth workflow. Keep the digest in the verification store; purge the outbox secret after submission or expiry. Do not substitute a predictable token to avoid secret handling.

Consume the token and update the password atomically, with a conditional write that fails for an already-used or expired token. Concurrent redemptions must yield at most one success. Apply the application's password policy, invalidate affected sessions according to policy, and send a change notification without secrets. Requesting another reset must not itself invalidate a user's active sessions. Decide explicitly whether a new reset revokes earlier unredeemed tokens and handle queued emails consistently.

Do not change account state merely on a GET: link scanners routinely open links. Require the user's intended reset action. Treat password changes and notification sending as separate outcomes; an email failure cannot roll back an already-committed password change.

## Invitations

Bind an invitation to its intended organization/resource, intended recipient, purpose, permitted role, inviter, expiry, and token digest. Check the inviter's authorization when creating it and re-check current organization/role policy on acceptance. Keep role choices server-controlled.

Accept atomically, verifying the recipient's identity or requiring an explicit authorized reassignment flow. A different logged-in user must not silently gain membership intended for another address. Model resend, revoke, already-accepted, removed organization, and concurrent acceptance deliberately. Scanner GET requests must not consume invitations or create membership.

Preserve recipient binding across provider retries and normalize addresses consistently with the identity system. New invitations and resends need rate limits and abuse controls; arbitrary user-provided text in an invitation is untrusted content. Avoid turning invitation endpoints into unrestricted email relays.

## Behavioral checks

Verify unknown-account responses, token expiry, wrong-purpose tokens, replay, concurrent redemption, tampered return URLs, scanner GETs, expired queued messages, unauthorized invitation roles, wrong-recipient acceptance, revoked invitations, and organization deletion. Ensure errors and logs do not expose token values.
